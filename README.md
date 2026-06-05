# 💸 Asistente Financiero Multiagente

> Bot de Telegram que registra y consulta gastos personales en **lenguaje natural**, orquestado por un sistema multiagente con IA, persistencia en PostgreSQL y aislamiento multi-tenant por usuario.

**Estado:** 🟢 **Funcional end-to-end** — Agente Registro + Agente Consultas + Orquestador validados en Telegram real.

---

## 🎯 ¿Qué hace?

Conversaciones reales con el bot:

| Usuario escribe | El bot hace |
|---|---|
| `gasté 15mil en almuerzo` | Registra: $15.000 · Alimentación · hoy |
| `pagué 30mil de uber ayer` | Registra: $30.000 · Transporte · ayer |
| `¿cuánto llevo este mes?` | Responde el total acumulado del mes |
| `¿en qué gasto más?` | Devuelve desglose por categoría |
| `muéstrame mis últimos gastos` | Lista los últimos N movimientos |

Un **AI Agent orquestador** clasifica la intención del mensaje y enruta al subworkflow especializado correcto. Cada usuario (`chat_id`) ve solo sus propios datos.

---

## 🏗️ Stack

| Componente | Tecnología |
|---|---|
| Orquestación de flujos | n8n (Docker) |
| Persistencia | PostgreSQL 16 (multi-tenant por `chat_id`) |
| IA | Claude (Anthropic) vía AI Agent + Information Extractor + Basic LLM Chain |
| Interfaz | Telegram Bot |
| Visor BD | Adminer |
| Túnel público | Cloudflare Quick Tunnel *(pendiente migrar a Named Tunnel)* |

---

## 🧩 Arquitectura

```
Telegram → Orquestador (AI Agent) → enruta a:
   ├── Agente Registro   → guarda gastos en Postgres
   ├── Agente Consultas  → totales, desgloses y listados
   └── Agente Contexto   → APIs externas (pendiente)
```

**Patrón:** AI Agent como orquestador que razona y enruta, cada agente especializado como **subworkflow** invocado como *tool*. Mantiene el orquestador delgado y los agentes desacoplados.

---

## 🤖 Agentes implementados

### Agente Registro
Subworkflow que recibe un mensaje natural y persiste el gasto.

```
[Trigger: chat_id, mensaje, nombre, username]
   ↓
[Information Extractor] → monto, categoría, descripción, fecha
   ↓ (Anthropic Chat Model)
[Upsert Usuario] (INSERT ... ON CONFLICT DO NOTHING)
   ↓
[Execute SQL] INSERT gasto con query parameters → respuesta
```

El Information Extractor convierte `"50mil"` → `50000` y `"ayer"` → fecha real. Categorías restringidas por system prompt a la lista exacta.

### Agente Consultas
Subworkflow con enfoque **híbrido seguro**: la IA interpreta intención y extrae parámetros, pero las queries SQL se construyen desde **plantillas predefinidas** en un nodo Code.

```
[Trigger: chat_id, pregunta]
   ↓
[Information Extractor] → tipo_consulta, fechas, categoría, límite
   ↓
[Code: constructor de query] → SQL + params parametrizados
   ↓
[Execute SQL] (filtrado por chat_id desde el TRIGGER)
   ↓
[Basic LLM Chain: interpreta resultados] → respuesta natural en HTML
```

**Tipos soportados:** `total_periodo`, `total_categoria`, `desglose_categorias`, `lista_gastos`.

> 🛡️ **Decisión de seguridad clave:** el `chat_id` se inyecta SIEMPRE desde el trigger del subworkflow, nunca desde la IA. Aunque el modelo alucinara, es físicamente imposible una fuga multi-tenant.

### Orquestador Finanzas (workflow principal)

```
[Telegram Trigger] → [Normalizar] → [AI Agent] → [Send Telegram]
                                       ├─ Anthropic Chat Model
                                       ├─ Simple Memory (key = chat_id)
                                       ├─ Tool: registrar_gasto
                                       └─ Tool: consultar_gastos
```

System Message del AI Agent define la regla de enrutamiento entre las dos tools según la intención.

---

## 🗄️ Base de datos

Diseño normalizado en 3 tablas:

```sql
CREATE TABLE usuarios (
    chat_id     BIGINT PRIMARY KEY,
    nombre      VARCHAR(255),
    username    VARCHAR(255),
    moneda      VARCHAR(10) DEFAULT 'COP',
    creado_en   TIMESTAMP DEFAULT NOW()
);

CREATE TABLE categorias (
    id      SERIAL PRIMARY KEY,
    nombre  VARCHAR(100) NOT NULL UNIQUE,
    emoji   VARCHAR(10)
);

CREATE TABLE gastos (
    id            SERIAL PRIMARY KEY,
    chat_id       BIGINT NOT NULL REFERENCES usuarios(chat_id),
    categoria_id  INTEGER REFERENCES categorias(id),
    monto         NUMERIC(12,2) NOT NULL,
    descripcion   TEXT,
    fecha         DATE DEFAULT CURRENT_DATE,
    creado_en     TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_gastos_chat_id ON gastos(chat_id);
CREATE INDEX idx_gastos_categoria ON gastos(categoria_id);
```

**Categorías semilla (9):** Alimentación 🍔 · Transporte 🚗 · Vivienda 🏠 · Entretenimiento 🎬 · Salud 💊 · Educación 📚 · Ropa 👕 · Servicios 💡 · Otros 📦

**Decisiones de diseño:**
- `BIGINT` para `chat_id` (los IDs de Telegram son grandes)
- `NUMERIC(12,2)` para dinero — **nunca** `FLOAT` (errores de redondeo)
- Categorías en tabla aparte → normalización, sin duplicación
- Índice en `chat_id` para queries multi-tenant rápidas

---

## 🚀 Cómo levantarlo

### Requisitos
- Docker + Docker Compose
- Bot de Telegram creado con BotFather (token a mano)
- API key de Anthropic
- Un túnel público (Cloudflare Quick Tunnel o equivalente)

### Pasos
```bash
git clone git@github.com:VladT-Tempest/n8n-finanzas-multiagente.git
cd n8n-finanzas-multiagente/docker

# Copiar plantilla y rellenar con tus valores
cp .env.example .env
# Editar .env: N8N_ENCRYPTION_KEY, POSTGRES_PASSWORD, WEBHOOK_URL (del túnel)

docker compose up -d
```

**Accesos:**
- n8n: `http://localhost:5680`
- Adminer: `http://localhost:8081`

### Importar los workflows
En n8n → menú **⋯ → Import from file** y carga los JSON de `workflows/`:
1. `Agente_Registro.json`
2. `Agente_Consultas.json`
3. `Orquestador_Finanzas.json`

Configura las credenciales (Telegram, Postgres, Anthropic), **publica los 3 workflows**, y el bot está listo.

---

## 📁 Estructura del repo

```
n8n-finanzas-multiagente/
├── .gitignore
├── README.md
├── docker/
│   ├── docker-compose.yml
│   ├── .env              # NO se sube (claves reales)
│   └── .env.example      # plantilla pública
├── docs/                 # documentación adicional
├── screenshots/          # capturas de la app
└── workflows/            # JSON exportados de n8n
    ├── Orquestador_Finanzas.json
    ├── Agente_Registro.json
    └── Agente_Consultas.json
```

**Servicios en `docker-compose`:**

| Servicio | Contenedor | Puerto host | Puerto interno |
|---|---|---|---|
| n8n | `n8n-finanzas` | 5680 | 5678 |
| postgres | `postgres-finanzas` | 5434 | 5432 |
| adminer | `adminer-finanzas` | 8081 | 8080 |

---

## ⚠️ Gotchas — lecciones aprendidas en el camino

Estos son los problemas reales que encontramos y resolvimos. Si replicas el setup, te ahorrarán horas.

### Infraestructura y entorno

1. **`N8N_ENCRYPTION_KEY` mismatch.** Si n8n arranca con la variable vacía, genera una clave random y la guarda en el volumen. Define la clave en `.env` **antes** del primer arranque, o resetea el volumen con `docker volume rm n8n_finanzas_data` si ya pasó.

2. **`POSTGRES_PASSWORD` vacía.** Postgres entra en loop de reinicio si la contraseña llega vacía (típicamente porque el `.env` no se carga). El `.env` debe estar en la misma carpeta que el `docker-compose.yml`, o usar `--env-file`.

3. **Conexión n8n → Postgres.** Host = `postgres` (nombre del servicio, **NO** `localhost`); Port = `5432` (interno, **NO** 5434). Los contenedores se ven por nombre en la red de Docker.

4. **Comentarios `#` pegados a valores en docker-compose YAML.** En YAML, un `#` solo es comentario si tiene **espacio antes**. `WEBHOOK_URL=https://foo.com# this changes` deja el `#` y el texto pegados al valor → URL corrupta. Siempre dejar espacio antes del `#`, o el comentario en línea aparte.

5. **GitHub usa SSH, no HTTPS.** Al crear el repo nuevo, usa la URL SSH (`git@github.com:...`) para evitar el error de "password authentication not supported".

### n8n y AI Agents

6. **Subworkflow debe estar Published/Active.** Un subworkflow invocado como tool por un AI Agent debe estar activo (botón "Publish" en versiones nuevas de n8n), aunque funcione en ejecución manual.

7. **Referencias en sub-nodos de AI Agent.** Usa referencias explícitas `$('NodoX').item.json.campo` en lugar de `$json.campo`, porque el contexto de datos en sub-nodos (Memory, Tools) no es lineal.

8. **Fecha en prompts de IA.** Los modelos no conocen la fecha actual. Inyecta `{{ $now.format('yyyy-MM-dd') }}` en el campo Text del nodo (que evalúa expresiones), NO en System Prompt Templates especiales que a veces no interpolan.

9. **Information Extractor anida la salida.** Los campos extraídos no quedan planos en `$json`, sino en `$json.output`. Acceder con `const e = $input.first().json.output`. Esto pilla a todo el mundo una vez.

10. **Credenciales fantasma tras borrarlas.** Si una credencial se elimina pero el workflow aún referencia su ID, n8n bloquea el Publish con *"Credential with ID X does not exist"*. Truco: en el nodo afectado, cambia la credencial a otra cosa, guarda, vuelve a la correcta, guarda. Eso fuerza a n8n a soltar la referencia podrida.

### Telegram + webhook + túnel (el círculo del dolor)

11. **🌟 Verificar el bot con `/getMe` — el gotcha rey.** Si tienes varios bots de Telegram, un token válido del **bot equivocado** pasa **todas** las pruebas (`setWebhook`/`getWebhookInfo` devuelven `ok:true`) pero los mensajes nunca disparan, porque le escribes a otro bot. **Diagnóstico estrella ante "webhook sano + cero ejecuciones":**
    ```
    https://api.telegram.org/bot<TOKEN>/getMe
    ```
    El campo `username` te dice de qué bot es el token. Si no coincide con el bot al que chateas, ahí está el problema. **Único síntoma:** silencio total — ningún log, ningún error, ningún warning. Costó 8+ horas de depuración.

12. **Cambiar token requiere re-registrar webhook.** Cambiar la credencial en n8n **no** re-registra automáticamente. Hay que hacer Unpublish → Publish del workflow para forzar el `setWebhook` con el secret/token nuevo. Y revisar que **TODOS** los nodos de Telegram (Trigger Y Send) usen la credencial actualizada.

13. **Cloudflare Quick Tunnel = URL efímera frágil.** La URL cambia en cada reinicio de `cloudflared`, lo que rompe el webhook silenciosamente. Cada cambio obliga a: actualizar `WEBHOOK_URL` en docker-compose → recargar el contenedor (`docker compose up -d`) → republicar el workflow. Para uso real, migrar a **Cloudflare Named Tunnel** (URL fija, gratis con dominio propio).

14. **Telegram HTML limitado.** Solo acepta `<b>`, `<i>`, `<u>`, `<s>`, `<code>`, `<pre>`, `<a>`. No `<p>`, `<div>`, `<ul>`, `<li>`, `<br>`. Usar Parse Mode HTML y limitar tags.

### 🔧 Comandos útiles de diagnóstico

```bash
# Ver qué WEBHOOK_URL tiene n8n cargada AHORA mismo
docker exec n8n-finanzas env | grep WEBHOOK_URL

# Logs en vivo (para ver si el webhook llega)
docker logs -f n8n-finanzas

# Listar workflows registrados
docker exec n8n-finanzas n8n list:workflow
```

```
# Info del webhook registrado en Telegram
https://api.telegram.org/bot<TOKEN>/getWebhookInfo

# Identidad del bot del token (gotcha #11)
https://api.telegram.org/bot<TOKEN>/getMe

# Borrar webhook y vaciar cola de mensajes pendientes
https://api.telegram.org/bot<TOKEN>/deleteWebhook?drop_pending_updates=true
```

---

## 🛣️ Roadmap — próximas iteraciones

- [ ] Migrar Quick Tunnel → **Cloudflare Named Tunnel** (URL fija, sin más sustos)
- [ ] **Deduplicación / idempotencia** por `message_id` de Telegram (evitar gastos duplicados por reintentos o doble-tap)
- [ ] **Agente Contexto** — APIs externas (conversión de divisas, etc.)
- [ ] **Memoria persistente** — migrar Simple Memory → Postgres para que la conversación sobreviva reinicios
- [ ] Tests automatizados de los subworkflows
- [ ] Dashboard de gastos (extender Adminer o construir uno propio)

---

## 🔑 Datos de referencia (para retomar)

- **Repo:** `github.com/VladT-Tempest/n8n-finanzas-multiagente`
- **Bot Telegram:** `@UserMisFinanzasBot` (token en credencial **"Bot Finanzas v2"**)
- **chat_id de prueba real:** `6986002891` (Alejandro / VladtTempest)
- **Credenciales n8n necesarias:** Telegram (Bot Finanzas v2), Postgres (postgres account), Anthropic (Anthropic account)
