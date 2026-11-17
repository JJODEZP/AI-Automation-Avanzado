# AI Automation Avanzado — Proyecto integrador

Curso AI Automation Avanzado (Coderhouse) · Jose Ortiz de Zevallos

## Pre-entrega 1: Agente autónomo básico funcional

**Archivo:** [`checkpoint1_jose_ortiz_de_zevallos.json`](checkpoint1_jose_ortiz_de_zevallos.json), workflow exportado de n8n.

Asistente de seguimiento estratégico para la Gerencia de Personas de una empresa productora de alimentos (prototipo con datos ficticios). Un líder reporta por chat el avance de un Key Result; el agente consulta el tracker OKR, calcula avance y semáforo, registra el check-in y escala los KR en rojo.

```
[Chat Trigger] → [AI Agent · Tools Agent · máx. 7 iteraciones] → [Log observabilidad (Gmail)] → [Respuesta]
                    ↑ OpenAI Chat Model
                    ↑ Tool: consultar_estado_okr  (Google Sheets, solo lectura)
                    ↑ Tool: registrar_checkin_kr  (Google Sheets, agregar fila)
```

- **Trigger:** Chat Trigger.
- **AI Agent:** modo Tools Agent, `Max Iterations = 7`, System Prompt modular (Rol → Ámbito → Objetivo → Reglas → Escalamiento).
- **Herramientas:** 2 nodos de Google Sheets conectados lateralmente al agente, cada uno con descripción semántica de cuándo usarlo.
- **Observabilidad:** nodo final de Gmail con el estado "Tarea completada" y el Execution Log (herramientas llamadas, entrada y resultado). La salida de error del agente envía una alerta.
- **Prueba:** ejecutado manualmente en n8n con 6 casos (check-ins en rojo, amarillo y verde; consulta general; pedido de cambio de meta; pedido de datos individuales).

**Modelo:** gpt-5-mini. Los créditos gratuitos de OpenAI incluidos en n8n Cloud no permiten gpt-4o; con una API key propia se puede seleccionar gpt-4o en el nodo `OpenAI Chat Model`.

**Para importarlo:** reemplazar `PEGA_AQUI_EL_ID_DE_TU_PLANILLA` (Google Sheet con las pestañas `OKR_Tracker` y `Checkins`) y `TU_CORREO@ejemplo.com`, y asignar las credenciales de OpenAI, Google Sheets y Gmail.
