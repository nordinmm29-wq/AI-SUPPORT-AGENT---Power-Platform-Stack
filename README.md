# AI Support Agent — Power Platform Stack

Un agente de soporte que analiza correos entrantes con IA, los clasifica por categoría y prioridad, responde automáticamente, escala los urgentes y registra cada ticket — todo integrado con el ecosistema Microsoft (Power Automate, Outlook, Teams y Dataverse) en lugar de Gmail, Telegram y Notion.

Es la versión Microsoft-native del AI Email Support Agent — misma lógica de triage con IA, pero con Outlook como buzón de entrada, Teams para las alertas urgentes y Dataverse como sistema de tickets.

## Qué hace

1. **Captura** — Un flujo de Power Automate detecta un correo nuevo en Outlook y lo envía al webhook de n8n.
2. **Normaliza** — Extrae remitente, asunto, cuerpo y fecha de recepción en una estructura limpia.
3. **Análisis con IA** — GPT-4.1-nano analiza el contenido del correo y devuelve categoría, prioridad y una propuesta de respuesta en formato JSON.
4. **Enrutamiento por prioridad**:
   - **Prioridad alta**: envía un acuse de recibo automático al cliente vía Outlook.
   - **Resto de casos**: envía una alerta a un canal de Teams para que el equipo lo revise cuanto antes.
5. **Registro del ticket** — Ambas ramas confluyen en la creación de un ticket en Dataverse, con título, cliente, prioridad, categoría y estado (`In progress`).

## Arquitectura

```
Nuevo correo en Outlook (Power Automate) ─> Set Data ─> GPT-4.1-nano (Análisis) ─> Set Priority/Category
                                                                                            │
                                                                                            ▼
                                                                                ¿Prioridad alta?
                                                                  ┌─────────────────────┴─────────────────────┐
                                                                 SÍ                                           NO
                                                                  │                                             │
                                                    Acuse de recibo (Outlook)                       Alerta urgente (Teams)
                                                                  └─────────────────────┬─────────────────────┘
                                                                                        ▼
                                                                        Crear ticket en Dataverse (Soporte)
```

## Stack

- **n8n** — motor de orquestación
- **Power Automate** — puente entre n8n y el ecosistema Microsoft mediante flujos activados por HTTP (endpoints de Logic Apps)
- **Outlook** — buzón de entrada y envío del acuse de recibo automático
- **Microsoft Teams** — canal de alerta para tickets urgentes
- **Dataverse** — sistema de tickets de soporte
- **OpenAI GPT-4.1-nano** — análisis, clasificación y redacción de respuesta sugerida, vía `@n8n/n8n-nodes-langchain`

## Setup

1. **Importa el workflow** en tu instancia de n8n.
2. **Crea los flujos de Power Automate** que respaldan cada nodo, cada uno expuesto como endpoint de Logic App:
   - Un flujo disparado **"Cuando llega un nuevo correo"** en Outlook que llame al webhook `power-automate-outlook-trigger` de n8n, enviando `from`, `subject`, `bodyPreview` y `receivedDateTime`.
   - `POWER_AUTOMATE_SEND_OUTLOOK_ACK` — envía el correo de acuse de recibo al cliente
   - `POWER_AUTOMATE_TEAMS_URGENT_ALERT` — publica la alerta de ticket urgente en el canal de Teams
   - `POWER_AUTOMATE_DATAVERSE_CREATE_TICKET` — crea el ticket en la tabla de Dataverse
3. **Sustituye las URLs de ejemplo** (`https://prod-xx.westeurope.logic.azure.com/...`) en cada nodo HTTP Request por las URLs reales de tus flujos, y la ruta del webhook de entrada por la URL pública de tu instancia n8n.
4. **Configura la credencial de OpenAI** en n8n para el nodo `Message a model` (modelo: `gpt-4.1-nano`).
5. **Crea la tabla de tickets en Dataverse** con las columnas: `title`, `customer`, `priority`, `category`, `status`.
6. **Revisa la condición de prioridad** en el nodo `If` — actualmente compara `priority` con el valor `"alta"`; ajústalo si tu prompt de IA devuelve las prioridades en otro idioma o formato (p. ej. `"HIGH"`).
7. Activa el workflow.

## Notas

- El nodo `Edit Fields1` conserva un valor fijo de ejemplo (`priority: "HIGH"`, `category: "URGENT"`) tal como estaba en el original — reemplázalo por la salida real del análisis de IA (`Message a model`) antes de producción.
- Todas las acciones del lado Microsoft pasan por Power Automate en lugar de nodos nativos de n8n, útil si tu organización centraliza integraciones por Power Platform por motivos de gobernanza o seguridad.
- El nodo de análisis con IA no se modificó — sigue devolviendo `categoria`, `prioridad`, `respuesta` y `Email` en español; solo cambiaron el canal de entrada (Gmail → Outlook), el canal de alerta (Telegram → Teams) y el almacenamiento (Notion → Dataverse).
