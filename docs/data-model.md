# Modelo de datos y contrato JSON

---

## Contrato JSON — frame de telemetría

Cada frame publicado por el rover en `sentinel/data` debe seguir este esquema.
Los campos marcados como **obligatorios** son necesarios para deduplicación y trazabilidad.

```jsonc
{
  // --- Identidad y deduplicación (obligatorios) ---
  "schema_version": "1.0",          // versión del contrato
  "event_id":       "<UUID v4>",    // único por frame; usado para deduplicación
  "device_id":      "<UUID v4>",    // identificador permanente del robot
  "boot_id":        "<UUID v4>",    // nuevo en cada arranque del ESP32
  "sequence":       1842,           // contador monotónico desde el último boot

  // --- Tiempo ---
  "captured_at":    "2026-10-01T15:24:10.000Z",  // RFC 3339 UTC — hora de captura
  "monotonic_ms":   184200,          // ms desde boot (siempre presente)
  "clock_quality":  "synced",        // "synced" | "unsynced" | "estimated"

  // --- Gas (valores crudos; no renombrar como ppm sin calibración) ---
  "gas_adc_raw":    1640,            // lectura ADC directa del sensor MQ
  "gas_rs_ohm":     null,            // resistencia del sensor (si se calcula)
  "calibration_id": null,            // ID del perfil de calibración aplicado

  // --- Ambiental ---
  "temperatura_c":  23.4,
  "humedad_pct":    58.1,

  // --- Posición y navegación ---
  "pose": {
    "x_m":            1.23,
    "y_m":            0.45,
    "theta_rad":      0.78,
    "incertidumbre":  0.08          // radio de incertidumbre en metros (p95)
  },
  "map_version_id": "<UUID v4>",    // versión de mapa usada en esta misión
  "mission_id":     "<UUID v4>",

  // --- Sensores de distancia y obstáculos ---
  "distancia_frontal_m": 0.42,      // convertida a metros; null si timeout
  "ir_lateral_izq":      true,      // true = libre, false = obstáculo
  "ir_lateral_der":      true,
  "ir_piso_izq":         true,      // true = piso detectado, false = riesgo de borde
  "ir_piso_der":         true,

  // --- IMU ---
  "pitch_rad": 0.02,
  "roll_rad":  0.01,

  // --- Energía y conectividad ---
  "bateria_tension_v":   7.4,
  "bateria_pct":         null,       // null si el método de estimación no está declarado
  "bateria_metodo":      null,       // descripción del método de cálculo
  "wifi_rssi_dbm":       -62,
  "motores_activos":     true        // orden enviada, no confirma giro efectivo sin encoder
}
```

> **Importante**: `gas_adc_raw` no es una medición en PPM certificada.
> Solo se convierte a una unidad física si existe un perfil de calibración con `calibration_id`
> y el método está documentado. El backend nunca renombra un ADC crudo como PPM.

---

## Contrato JSON — alerta

Publicado en `sentinel/alerta` cuando el firmware detecta una condición inmediata.

```jsonc
{
  "schema_version": "1.0",
  "event_id":       "<UUID v4>",
  "device_id":      "<UUID v4>",
  "boot_id":        "<UUID v4>",
  "sequence":       1843,
  "captured_at":    "2026-10-01T15:24:12.000Z",
  "tipo_alerta":    "ATASCADO_FISICO",
  // Tipos conocidos del firmware:
  // GAS_CRITICO · BATERIA_CRITICA · ATASCADO_FISICO
  // VOLCADO · IMPACTO_DETECTADO · WIFI_LIMITE · ABISMO_DETECTADO
  "parametros":     {}               // contexto específico de la alerta
}
```

> Los umbrales de las alertas (e.g. 10 % batería, -85 dBm WiFi, 70° inclinación)
> son parámetros configurables por instalación — no constantes hardcodeadas.
> La parada física del robot depende de su envolvente validada, no de un ángulo genérico.

---

## Confirmación de persistencia (ACK)

El broker envía `sentinel/ack` al robot después de que el backend haya confirmado
el commit en PostgreSQL.

```jsonc
{
  "event_id": "<UUID v4>",    // el mismo event_id del frame original
  "persisted_at": "2026-10-01T15:24:10.500Z"
}
```

El rover conserva el frame en su búfer local hasta recibir este acuse.
`PUBACK` del broker solo confirma el tramo MQTT, no el commit en base de datos.

---

## Modelo relacional (esquema simplificado)

```
domicilio
  id · nombre · zona_horaria · ...

robot
  id · domicilio_id · nombre · ...

mision
  id · robot_id · map_version_id · inicio · fin · estado

map_version
  id · robot_id · domicilio_id · validado · storage_path · ...

telemetria_frame
  id · robot_id · mission_id · map_version_id
  event_id · boot_id · sequence            ← deduplicación UNIQUE
  captured_at · received_at · schema_version · clock_quality
  gas_adc_raw · calibration_id
  temperatura_c · humedad_pct
  pose_x_m · pose_y_m · pose_theta_rad · pose_incertidumbre
  distancia_frontal_m · ir_lateral_izq · ir_lateral_der
  ir_piso_izq · ir_piso_der · pitch_rad · roll_rad
  bateria_tension_v · bateria_pct · bateria_metodo
  wifi_rssi_dbm · motores_activos
  payload_raw (JSONB)                      ← copia íntegra del frame original

alerta
  id · robot_id · event_id · boot_id · sequence
  captured_at · received_at
  tipo_alerta · parametros (JSONB)
  estado · atendida_at

estacion_muestra
  id · telemetria_frame_id · estacion_id · habitacion_id · map_version_id
  altura_m · ventana_inicio · ventana_fin

membresia_usuario
  usuario_id · domicilio_id · rol · activa
```

### Principios de diseño

- **Aislamiento por domicilio**: RLS en PostgreSQL — se aplica en cada tabla, Realtime y Storage.
- **Muestra cruda preservada**: `payload_raw` guarda el JSON original; el backend guarda también `received_at` y `captured_at` por separado.
- **Deduplicación confiable**: la restricción UNIQUE sobre `(device_id, boot_id, sequence)` garantiza exactamente una fila por frame.
- **Posición sin inferencia**: una lectura sin pose conocida permanece en historial temporal y no se asigna automáticamente a una habitación.
- **Versionado de contrato**: cambios aditivos mantienen `v1`; cambios de significado o eliminaciones crean `v2` con adaptador y periodo de transición.
