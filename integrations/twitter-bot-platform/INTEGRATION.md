# Tweet Bot Platform — CLAW Integration Guide

> Versión: 1.0.0  
> Actualizado: 2026-04-09  
> Agente responsable de contenido: `marketing-twitter-engager`  
> Plataforma destino: `twitter-bot-platform` (Fly.io + Supabase)

---

## Resumen

Este documento describe cómo el orquestador **CLAW/CROW** genera y entrega contenido
para Twitter/X a la plataforma de bots, usando al agente `marketing-twitter-engager`
como cerebro creativo especializado.

**Principio clave:** el orquestador decide *qué* publicar; la plataforma decide *cuándo* y *cómo*.

---

## División de responsabilidades

| Responsabilidad | Quién la tiene |
|---|---|
| Generación del texto del tweet | `marketing-twitter-engager` (CLAW) |
| Revisión editorial / coherencia con calendario | `marketing-content-creator` (CLAW, opcional) |
| Validación de límites por plan | Tweet Bot Platform (`worker.py`) |
| Programación y colas horarias | Tweet Bot Platform (`scheduler.py`) |
| Publicación en la API de Twitter | Tweet Bot Platform (`publisher.py`) |
| Logging y auditoría | Tweet Bot Platform (`AuditLogger`) |
| Persistencia de sesiones / métricas | Tweet Bot Platform (Supabase) |

---

## Endpoint de recepción

La plataforma expone un endpoint dedicado para el orquestador:

```
POST /api/orchestrator/tweet
Authorization: Bearer <ORCHESTRATOR_SECRET>
Content-Type: application/json
```

### Request body

Este es el `TwitterContentPayload` que genera el agente `marketing-twitter-engager`
y que el orquestador envía sin modificar:

```json
{
  "status": "ok",

  "tweet": {
    "text": "Los modelos de razonamiento no reemplazan al dev: eliminan el trabajo sin pensar para que quede el que importa. El 73 % de los equipos que los adoptaron reporta menos bugs en revisión de código (JetBrains, 2025).",
    "char_count": 219,
    "language": "es",
    "personality_used": "professional"
  },

  "alternatives": [
    "Razonamiento ≠ más velocidad. Razonamiento = menos errores en producción. Los datos de JetBrains (2025) muestran 73 % de reducción de bugs. El cuello de botella ya no es el código: es el criterio."
  ],

  "thread": [],

  "scheduling": {
    "publish_at": null,
    "timezone": "America/Bogota"
  },

  "meta": {
    "pipeline_name": "twitter_content_v1",
    "trace_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "tokens_used": 312,
    "model": "gpt-4.1-mini"
  },

  "error": null
}
```

> El campo `bot_id` lo incluye el orquestador en el header `X-Bot-Id: <uuid>` para que
> la plataforma sepa a qué cuenta de `platform_bots` asociar el tweet.

### Response body

```json
{
  "tweet_id": "uuid-generado-por-plataforma",
  "status": "queued" | "scheduled" | "published" | "error",
  "publish_at": "2026-04-09T20:00:00Z" | null,
  "message": "Tweet encolado para publicación automática"
}
```

---

## Flujo completo paso a paso

```
1. TRIGGER
   El scheduler de la plataforma (cron) o una acción de usuario
   detecta que un bot necesita nuevo contenido.

2. REQUEST al orquestador
   La plataforma llama a CLAW:
   POST https://claw-host/api/task
   Body: {
     "pipeline_name": "twitter_content_v1",
     "bot_id": "<uuid>",
     "topic": "<tema del día o vacío para auto-selección>",
     "request_type": "generate"
   }

3. CLAW ejecuta el pipeline
   Fase 1 → marketing-twitter-engager genera TwitterContentPayload
   Fase 2 → marketing-content-creator valida coherencia editorial (opcional)

4. CLAW devuelve TwitterContentPayload
   La plataforma recibe el payload en la respuesta del POST /api/task.

5. La plataforma procesa el payload
   - Valida límites del plan del bot.
   - Si publish_at=null → encola en scheduler respetando ventanas horarias.
   - Si publish_at tiene fecha → inserta en platform_tweets con status='scheduled'.
   - Guarda texto, alternativas y metadata en Supabase.

6. Publisher ejecuta la publicación
   En el momento programado, publisher.py toma el tweet de la cola
   y llama a la API de Twitter (Tweepy).
   Registra resultado en AuditLogger.
```

---

## Tabla Supabase — campos añadidos para la integración

Añadir a `platform_tweets`:

```sql
-- Trazabilidad del orquestador
orchestrator_trace_id  UUID,
pipeline_name          TEXT,
tokens_used            INTEGER,
model_used             TEXT,
alternatives           JSONB,   -- array de variantes alternativas
thread_tweets          JSONB,   -- array ordenado si es un hilo
personality_used       TEXT,
```

---

## Variables de entorno requeridas

Añadir a `.env` y a los secrets de Fly.io:

```bash
# Secret compartido para autenticar llamadas del orquestador
ORCHESTRATOR_SECRET=<token-seguro-generado>

# URL base del orquestador CLAW
CLAW_BASE_URL=https://tu-claw-host.fly.dev

# Pipeline por defecto para generación de contenido Twitter
CLAW_TWITTER_PIPELINE=twitter_content_v1
```

---

## Manejo de errores

| Escenario | Comportamiento de la plataforma |
|---|---|
| CLAW no responde (timeout) | Reintentar 2 veces con backoff de 5 s. Si falla, registrar en `audit_logs` con `event=orchestrator_timeout` y no publicar. |
| `status: error` en payload | Registrar en `audit_logs`, marcar tweet como `status='failed'`, notificar al usuario si tiene alertas activas. |
| Texto > 280 caracteres | Rechazar localmente antes de publicar, registrar `event=content_too_long`, solicitar rewrite al orquestador. |
| Límite del plan superado | Devolver 429 al orquestador con `retry_after` calculado según ventana del plan. |
| Bot desactivado o suspendido | Devolver 403, no encolar. |

---

## Seguridad

- El header `Authorization: Bearer <ORCHESTRATOR_SECRET>` es obligatorio en toda llamada.
- La plataforma valida que `X-Bot-Id` pertenezca a un bot activo antes de procesar.
- El payload del orquestador nunca incluye credenciales de Twitter — la plataforma las gestiona internamente.
- Todos los payloads se registran en `audit_logs` con `source='orchestrator'`.

---

## Roadmap de esta integración

- [x] Definir contrato de payload (esta documentación)
- [ ] Implementar `POST /api/orchestrator/tweet` en `server.py`
- [ ] Añadir columnas de trazabilidad a `platform_tweets`
- [ ] Configurar `ORCHESTRATOR_SECRET` en Fly.io
- [ ] Pipeline `twitter_content_v1` en CLAW apuntando a `marketing-twitter-engager`
- [ ] Test de integración end-to-end en staging
- [ ] Activar en producción para el primer bot piloto
