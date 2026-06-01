# Asistente Financiero Multiagente por Telegram

Sistema multiagente que permite registrar y consultar gastos personales
mediante lenguaje natural a través de Telegram, orquestado con n8n e IA.

## Arquitectura
- **Orquestador:** AI Agent que clasifica intenciones y enruta a subworkflows
- **Agente Registro:** extrae y guarda gastos en PostgreSQL
- **Agente Consultas:** responde preguntas sobre los datos con análisis IA
- **Agente Contexto:** enriquece con datos externos vía APIs

## Stack
- n8n (orquestación)
- PostgreSQL (persistencia, multi-tenant por chat_id)
- Claude / GPT (procesamiento de lenguaje natural)
- Docker Compose (infraestructura)

## Estado
🚧 En construcción

