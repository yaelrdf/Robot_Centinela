# Subsistemas

Descripción técnica de cada subsistema del proyecto Robot Centinela.

---

## 1. Firmware IoT (ESP32)

El firmware corre en un **ESP32** como unidad principal de cómputo.
Una **ESP32-CAM V3** complementa con captura de video de forma independiente.

### Sensores embarcados

| Sensor | Función | Interfaz |
|--------|---------|----------|
| MQ6 | Gas LP / butano | ADC analógico |
| MQ4 | Gas natural / metano | ADC analógico |
| DHT22 | Temperatura y humedad | Digital 1-wire |
| HC-SR04 | Distancia frontal (ultrasónico) | Digital trigger/echo |
| VL53L0X | Distancia por láser (barrido con servo) | I²C |
| FC51 (×4) | Obstáculos laterales IR | Digital |
| TCRT5000 (×2) | Detección de borde / piso IR | Digital |
| MPU6050 | Acelerómetro + giroscopio (IMU) | I²C |
| Encoders de motor | Odometría | Pulsos digitales |

### Actuadores

| Actuador | Función |
|----------|---------|
| 2× JGA25-370 con encoder | Tracción diferencial (orugas) |
| Servo SG90 | Barrido del sensor VL53L0X |
| Ventilador 5010 5 V | Refrigeración de la electrónica |
| Buzzer 5 V | Alertas sonoras locales |

### Responsabilidades del firmware

- Leer y calibrar todos los sensores
- Ejecutar el modelo de movimiento diferencial y la odometría
- Mantener y reutilizar el mapa 2D (ocupación por celdas)
- Detectar y evitar obstáculos, bordes y condición de volcamiento
- Publicar un frame completo cada 2 s vía MQTT con TLS
- Mantener un búfer local hasta recibir ACK de persistencia
- Ejecutar paradas de seguridad locales (independientes de la nube)

### Parámetros de operación (primera entrega)

| Parámetro | Valor |
|-----------|-------|
| Velocidad de validación | 0.10 m/s |
| Velocidad máxima de diseño | 0.20 m/s |
| Pendiente de cálculo | 5° |
| Intervalo de publicación MQTT | 2 s |
| Entorno soportado | Un piso, superficie firme y seca, sin escaleras accesibles |

---

## 2. Backend e ingesta

El backend actúa como puente entre el rover y las aplicaciones de usuario.

### Responsabilidades

- Autenticar cada conexión de dispositivo
- Consumir los tópicos MQTT `sentinel/data` y `sentinel/alerta`
- Validar el contrato JSON (`schema_version`, tipos, rangos)
- Deduplicar frames por `(device_id, boot_id, sequence)`
- Guardar en PostgreSQL en transacciones atómicas
- Enviar `sentinel/ack` al rover tras el commit
- Evaluar reglas de alertas informativas y emitir notificaciones
- Exponer consultas autorizadas a web y móvil vía API Supabase

### Restricciones

- No hardcodear umbrales de gas como valores de seguridad
- Las reglas de alerta son configurables por gas y método
- Los datos crudos de ADC no se renombran como `ppm` sin calibración documentada
- El backend conserva siempre la muestra cruda y la calibración usada

---

## 3. Aplicación Web

Dashboard de monitoreo accesible desde navegador.

### Responsabilidades

- Autenticación de sesión vía Supabase Auth
- Visualización del estado actual del robot (gas, temperatura, humedad, batería, conectividad)
- Mapa 2D persistente con pose y trayectoria del robot
- Historial ambiental con filtros por rango de tiempo, misión y canal
- Suscripción en tiempo real vía Supabase Realtime
- Recuperación de datos tras reconexión (sin depender de Realtime para reproducción)
- Exportación autorizada de datos (CSV)
- Preferencias de usuario (tema claro/oscuro/sistema, formato de fecha)
- Accesibilidad: objetivo WCAG 2.2 nivel AA

### Alojamiento

El código fuente de la aplicación web vive en su propio repositorio,
integrado en este umbrella como submódulo en `web/`.
<!-- TODO: agregar URL del repositorio web aquí -->

---

## 4. Aplicación Móvil

Dashboard de monitoreo para dispositivos iOS y Android.

### Responsabilidades

- Mismas responsabilidades de visualización que la web, desde la misma API
- Autenticación de sesión vía Supabase Auth
- Notificaciones push para alertas
- Funcionamiento correcto en redes locales (LAN) y WiFi del domicilio
- Compatible con HTTPS, permisos de red local y contenido mixto

### Restricciones compartidas con web

- No escribe telemetría directamente
- No recibe credenciales del broker MQTT
- Usa la misma semántica de tiempo, unidad y calidad que la web
- Un dato sin actualización muestra su edad e indicador de dato atrasado

### Alojamiento

El código fuente de la aplicación móvil vive en su propio repositorio,
integrado en este umbrella como submódulo en `mobile/`.
<!-- TODO: agregar URL del repositorio móvil aquí -->

---

## 5. Supabase (infraestructura de datos)

Supabase provee la base de datos compartida y los servicios en tiempo real.

### Servicios utilizados

| Servicio | Uso |
|----------|-----|
| PostgreSQL | Persistencia principal de telemetría, misiones, mapas y usuarios |
| Auth | Autenticación de usuarios web y móvil |
| RLS (Row Level Security) | Aislamiento de datos por domicilio en todas las consultas |
| Realtime | Suscripción a cambios para los dashboards |
| Storage | Almacenamiento privado de mapas validados |

### Principio de aislamiento

Cada usuario accede únicamente a los domicilios y robots para los que tiene membresía activa.
El aislamiento se aplica en PostgreSQL (RLS), Realtime y Storage — no solo en la capa de aplicación.
