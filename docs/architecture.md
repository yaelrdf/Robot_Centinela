# Arquitectura del sistema

## Visión general

Robot Centinela se organiza en cuatro capas independientes que se comunican mediante
contratos versionados:

```
┌─────────────────────────────────────────────────────┐
│                    Aplicaciones                      │
│          Web Dashboard      Móvil Dashboard          │
└──────────────────┬──────────────┬────────────────────┘
                   │  Supabase    │
                   │  Realtime +  │
                   │  REST API    │
┌──────────────────▼──────────────▼────────────────────┐
│                     Backend                          │
│   Ingesta MQTT → Validación → PostgreSQL (Supabase)  │
└──────────────────────┬───────────────────────────────┘
                       │ MQTT TLS QoS 1
┌──────────────────────▼───────────────────────────────┐
│                   Firmware (IoT)                     │
│   ESP32 — sensores — navegación — transmisión        │
└──────────────────────────────────────────────────────┘
```

---

## Flujo de datos principal

```
ESP32
  │  publica cada 2 s
  ▼
Broker MQTT (TLS, QoS 1)
  │  tópicos: sentinel/data  sentinel/alerta
  ▼
Puente de ingesta (backend)
  │  autentica, valida JSON, deduplica, guarda
  ▼
PostgreSQL en Supabase
  │  RLS por domicilio · Realtime · Storage (mapas)
  ▼
Web / Móvil  ──  consultan y se suscriben, nunca escriben telemetría
```

> **Principio clave**: la nube es la fuente común de visualización e historial.
> El firmware es la autoridad para la detención inmediata del robot.
> Un fallo de nube no deshabilita la parada local.

---

## Responsabilidades por capa

### Firmware (IoT)
- Adquisición de sensores (gas, ambiental, distancia, IMU, encoders)
- Seguridad de movimiento local (parada ante obstáculo, borde o volcamiento)
- Calibración aplicada
- Estimación de posición y odometría
- Construcción y reutilización del mapa 2D
- Búfer y transmisión MQTT con ACK de aplicación

### Backend
- Autenticación de dispositivos y personas
- Validación de contrato JSON y versionado
- Deduplicación por `event_id / boot_id / sequence`
- Persistencia transaccional en PostgreSQL
- Evaluación de reglas de alertas informativas
- Exposición de consultas autorizadas y emisión de cambios por Realtime

### Supabase
- PostgreSQL con RLS (Row Level Security) por domicilio
- Auth para usuarios humanos
- Storage privado para mapas validados
- Realtime para suscripciones de cambios

### Web y Móvil
- Autenticación de sesión
- Consultas y suscripción a Realtime
- Presentación de telemetría, mapa, posición y alertas
- Exportación autorizada de datos
- Preferencias de usuario (tema, formato de fecha, idioma)

---

## Tópicos MQTT

| Tópico | Dirección | Descripción |
|--------|-----------|-------------|
| `sentinel/data` | ESP32 → broker | Frame de telemetría (cada 2 s) |
| `sentinel/alerta` | ESP32 → broker | Evento de alerta inmediata |
| `sentinel/ack` | broker → ESP32 | Confirmación de persistencia en BD |
| `sentinel/status` | ESP32 → broker | Presencia (Last Will, retain=true) |

Para múltiples robots el esquema se extiende a:
`sentinel/{device_public_id}/{subtópico}`

---

## Protocolo de comunicación

- **Transporte**: MQTT 3.1.1 con TLS (verificación de servidor)
- **QoS**: 1 para telemetría y alertas — puede duplicar; la idempotencia es responsabilidad de la aplicación
- **Sesión**: persistente con expiración acotada
- **Retain**: `false` para datos, `true` solo para `sentinel/status`
- **Ingesta HTTP**: `Content-Type: application/json; charset=utf-8`, timestamps RFC 3339 UTC, unidades SI

---

## Restricciones de seguridad

- Web y móvil no reciben credenciales del broker
- Un `id_robot` en el JSON no autoriza escritura para otro dispositivo
- No usar broker público anónimo para datos de domicilio
- RLS en Supabase garantiza aislamiento por domicilio en cada consulta, incluyendo Realtime y Storage
