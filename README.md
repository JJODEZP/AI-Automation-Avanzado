# AI Automation Avanzado — Proyecto integrador

Curso AI Automation Avanzado (Coderhouse) · Jose Ortiz de Zevallos

Copiloto de Estrategia y Gestión de Personas para una empresa productora de alimentos (prototipo con datos ficticios). El mismo proyecto crece en cada pre-entrega.

## Pre-entrega 3: Memoria persistente y resumen automático

**Archivos (exportados de n8n):**
- [`manager_modulo3_ortiz_de_zevallos_jose.json`](manager_modulo3_ortiz_de_zevallos_jose.json): Manager de la pre-entrega 2 con la capa de memoria.
- [`worker1_modulo3_ortiz_de_zevallos_jose.json`](worker1_modulo3_ortiz_de_zevallos_jose.json): Worker 1 actualizado; recibe el contexto de memoria. El Worker 2 no cambia.

```
[Chat Trigger] → [Airtable: buscar por Session_ID] → [IF ¿usuario con memoria?]
   ├─ sí → [Set: user_name, status, last_summary]
   └─ no → [Airtable: crear fila] → [Set: valores neutros]
→ [Router + Workers de la pre-entrega 2, con el contexto entre delimitadores]
→ [Set: preparar memoria] → [Airtable: upsert por Session_ID] → [IF ¿más de 5 mensajes?]
   └─ sí → [Modelo mini: JSON {asunto_principal, puntos_clave, accion_requerida}] → [Set: validar JSON] → [Airtable: upsert del resumen]
→ [Log Gmail] → [Respuesta]
```

- **Base de memoria (Airtable, tabla `Memoria Usuarios`):** una fila por `Session_ID`, con los campos Nombre del usuario, Fecha de actualización, Resumen consolidado, Estado del caso, Datos clave, Última intención, Cantidad de mensajes e Historial reciente.
- **Aislamiento de sesión:** la búsqueda usa `filterByFormula` sobre `Session_ID` justo después del trigger; todas las escrituras son upsert sobre ese mismo campo (idempotentes, sin filas duplicadas).
- **Inyección de contexto:** `user_name`, `status` y `last_summary` entran al System Prompt del agente entre `[INICIO DE CONTEXTO COMPARTIDO]` y `[FIN DEL CONTEXTO COMPARTIDO]`, como dato y no como instrucción.
- **Resumen:** al superar 5 mensajes, un modelo económico (gpt-5-mini) condensa el historial con un prompt estricto; un nodo Set valida el JSON antes de escribirlo y el búfer vuelve a 0. Solo se guardan hechos accionables: nada de conversación completa, logs ni HTML.
- **Prueba:** usuario nuevo, usuario recurrente con referencias al contexto, activación del resumen, recuperación desde la memoria consolidada y aislamiento entre tres sesiones.

**Para importarlos:** crear en Airtable la tabla con esos campos, reemplazar `PEGA_AQUI_EL_ID_DE_TU_BASE_AIRTABLE` y `PEGA_AQUI_EL_ID_DE_LA_TABLA_MEMORIA`, y asignar la credencial "Airtable Personal Access Token" (scopes `data.records:read`, `data.records:write`, `schema.bases:read`). El resto de los placeholders son los mismos de la pre-entrega 2.

## Pre-entrega 2: Orquestación multi-agente con sub-workflows

**Archivos (exportados de n8n):**
- [`manager_modulo2_ortiz_de_zevallos_jose.json`](manager_modulo2_ortiz_de_zevallos_jose.json): Workflow Manager.
- [`worker1_modulo2_ortiz_de_zevallos_jose.json`](worker1_modulo2_ortiz_de_zevallos_jose.json): Worker 1 · Check-in OKR.
- [`worker2_modulo2_ortiz_de_zevallos_jose.json`](worker2_modulo2_ortiz_de_zevallos_jose.json): Worker 2 · Indicadores de Personas.

El agente de la pre-entrega 1 se encapsuló como especialista y se agregó un Manager que interpreta la intención y delega (patrón Manager-Worker).

```
[Chat Trigger] → [Router: LLM + JSON estructurado] → [Switch por intención]
   ├─ CHECKIN_OKR          → [Set payload] → [Execute Workflow → Worker 1 · Check-in OKR]
   ├─ INDICADORES_PERSONAS → [Set payload] → [Execute Workflow → Worker 2 · Indicadores]
   ├─ FUERA_DE_ALCANCE     → respuesta de alcance
   └─ fallback (REVISION_HUMANA, confianza < 0,6 o falla del Router) → supervisor humano
[IF status = success] → integrar respuesta | contingencia → [Log Gmail] → [Respuesta]
```

- **Router:** Basic LLM Chain con taxonomía cerrada y Structured Output Parser; devuelve `{intencion, confianza, unidad, mes, indicador, motivo}`. Un Switch determinista decide el destino.
- **Workers:** dos sub-workflows separados, cada uno con Execute Workflow Trigger y una tarea atómica. El Worker 2 no usa LLM.
- **Handoff:** un Set limpia el payload antes de cada Execute Workflow, con *Wait for child to finish* activado. Timeouts: Manager 180 s, Worker 1 120 s, Worker 2 30 s.
- **Contrato de salida:** todo Worker devuelve `{"status": "success" | "error", "worker", "entrada", "data"}`. Ante fallos devuelve un código (`SIN_DATOS`, `ERROR_FUENTE`, `ERROR_AGENTE`, `LIMITE_ITERACIONES`) en vez de cortar la ejecución; si el sub-workflow no responde, la salida de error del Execute Workflow arma el mismo contrato.
- **Trazabilidad:** correo final con intención, Worker invocado, parámetros enviados y respuesta devuelta, sin credenciales ni trazas técnicas.
- **Prueba:** 8 casos ejecutados en n8n (check-in, indicadores, revisión humana, fuera de alcance, mes sin datos, Worker inexistente, fuente caída y consulta de todas las unidades).

**Para importarlos:** importar primero los dos Workers, copiar sus IDs en los nodos `Delegar a Worker 1` y `Delegar a Worker 2` del Manager (`PEGA_AQUI_EL_ID_DEL_WORKER_1` y `_2`), reemplazar `PEGA_AQUI_EL_ID_DE_TU_PLANILLA` (pestañas `OKR_Tracker`, `Checkins` e `Indicadores`) y `TU_CORREO@ejemplo.com`, y asignar las credenciales de OpenAI, Google Sheets y Gmail.

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
