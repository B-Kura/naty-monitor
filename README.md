# 📡 Naty Agent Monitor

Dashboard en tiempo real para monitorear el estado de los agentes de call center de **Naty Multicable**, construido sobre GitHub Pages, Supabase y n8n.

🔗 **Live:** [b-kura.github.io/naty-monitor](https://b-kura.github.io/naty-monitor/)

---

## ¿Qué hace?

- Muestra en tiempo real qué agentes están **online** u **offline**
- Indica cuántos minutos lleva desconectado cada agente offline
- Muestra cuántas **conversaciones de Chatwoot** tiene asignadas cada agente
- Muestra los totales globales del equipo: chats totales, asignados y sin asignar
- Permite **asignar automáticamente** las conversaciones sin asignar con un solo clic
- Se actualiza cada **15 segundos** sin recargar la página
- Soporta **modo claro y oscuro** con preferencia guardada en el navegador

---

## Arquitectura

```
Chatwoot API
    │
    ▼
n8n (cada 15s)
    ├── GET /agents           → estado online/offline de cada agente
    ├── GET /conversations    → conteo de chats por agente y totales del equipo
    └── Supabase (naty_logs)  → guarda y actualiza el estado
            │
            ▼
    GitHub Pages (index.html)
            ├── Lee Supabase cada 15s y renderiza el dashboard
            └── Botón "Asignar" → POST a n8n Webhook
                                        │
                                        └── n8n asigna en Chatwoot (sin CORS)
```

---

## Stack

| Componente | Tecnología |
|---|---|
| Dashboard | HTML + CSS + JS vanilla (sin dependencias) |
| Hosting | GitHub Pages |
| Base de datos | Supabase (PostgreSQL) |
| Automatización | n8n (self-hosted en `n8.watiteam.com`) |
| Chat / Agentes | Chatwoot (`naty.watiteam.com`) |

---

## Tabla en Supabase: `naty_logs`

| Columna | Tipo | Descripción |
|---|---|---|
| `id_agente` | INTEGER (PK) | ID del agente en Chatwoot. `0` = fila de meta globales |
| `nombre` | TEXT | Nombre del agente |
| `estado` | TEXT | `online` \| `offline` |
| `ultima_conexion` | TEXT | ISO timestamp de cuando se desconectó |
| `tiempo_desconectado_minutos` | INTEGER | Minutos acumulados offline |
| `ultimo_chequeo` | TEXT | ISO timestamp del último ciclo de n8n |
| `total_conversaciones` | INTEGER | Chats asignados al agente ahora mismo |
| `all_count` | INTEGER | Total chats del equipo (solo fila id=0) |
| `assigned_count` | INTEGER | Chats asignados del equipo (solo fila id=0) |
| `unassigned_count` | INTEGER | Chats sin asignar del equipo (solo fila id=0) |

> La fila con `id_agente = 0` es especial: almacena los totales globales del equipo bajo el nombre `_meta_globales`.

---

## Flujos de n8n

### 1. Flujo principal (cada 15s)
```
Schedule Trigger
    → Get Todos los Logs (Supabase)
    → Agentes online (Chatwoot /agents)
    → Filtro de selección (IDs monitoreados)
    → Conteo de conversaciones (Chatwoot /conversations)
    ├── Procesar Agentes + Chats (Code) → ¿Es nuevo? → Insert / Update
    └── Preparar Meta Globales (Code)   → Upsert HTTP (Supabase)
```

### 2. Webhook de asignación (`POST /webhook/naty-asignar`)
```
Webhook (POST)
    → Calcular Asignaciones (round-robin ponderado)
    → ¿Hay asignaciones?
    ├── true  → Asignar en Chatwoot → Consolidar Resultado → Responder
    └── false → Sin asignaciones                           → Responder
```

---

## Lógica de asignación automática

Al presionar **⚡ Asignar conversaciones** el dashboard:

1. Obtiene las conversaciones sin asignar del equipo desde Chatwoot
2. Envía la lista al webhook de n8n junto con el estado actual de los agentes
3. n8n filtra los agentes **elegibles**:
   - `online` → siempre elegible
   - `offline` con ≤ 15 min desconectado → elegible
   - `offline` con > 15 min → excluido (posiblemente ya salió de turno)
4. Distribuye las conversaciones en **round-robin ponderado**: siempre asigna al agente con menos chats en ese momento
5. Devuelve cuántas se asignaron correctamente

---

## Configuración

Las credenciales están directamente en el `index.html` (variables al inicio del `<script>`):

```js
const SUPABASE_URL    = 'https://...supabase.co';
const SUPABASE_KEY    = '...';          // anon key (solo lectura pública)
const WEBHOOK_ASIGNAR = 'https://n8.watiteam.com/webhook/naty-asignar';
const CHATWOOT_URL    = 'https://naty.watiteam.com/api/v1/accounts/1';
const CHATWOOT_TOKEN  = '...';          // api_access_token
const CHATWOOT_TEAM   = 20;
const MAX_OFFLINE_MINS = 15;            // minutos máx offline para ser elegible
```

---

## Despliegue

El repositorio usa **GitHub Pages** con la rama `main` y el archivo `index.html` en la raíz.

Cualquier `git push` a `main` actualiza el dashboard en vivo en segundos.

```bash
git add index.html README.md
git commit -m "update dashboard"
git push origin main
```
