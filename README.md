# 🤖 Robot Centinela

**Monitoreo de gases y navegación doméstica en dos dimensiones**

Robot Centinela es un robot autónomo de tracción diferencial capaz de patrullar un domicilio, detectar gases de riesgo (LP, metano), medir condiciones ambientales (temperatura, humedad) y transmitir
telemetría en tiempo real a dashboards web y móvil. El robot construye y reutiliza un mapa 2D del
espacio, evita obstáculos y puede detener su misión de forma autónoma ante condiciones de riesgo.

---

## Repositorios del proyecto

Este es el **repositorio sombrilla**. Cada componente principal vive en su propio repositorio e
se integra aquí como submódulo de Git.

| Submódulo | Descripción |
|-----------|-------------|
| `web/`    | Aplicación web — dashboard de monitoreo |
| `mobile/` | Aplicación móvil — dashboard de monitoreo |
| `iot/`    | Firmware y software embebido del rover (ESP32) |

> Los enlaces a cada repositorio se agregan en [`.gitmodules`](.gitmodules) una vez que estén
> disponibles. Ver también el workflow de sincronización en
> [`.github/workflows/sync-submodules.yml`](.github/workflows/sync-submodules.yml).

---

## Descripción general del sistema

### Capa física — el rover
El robot es un chasis de orugas con tracción diferencial impulsado por dos motores con encoder.
Embarca un ESP32 como unidad de cómputo principal y una ESP32-CAM para captura de video local.
Sus sensores incluyen:

- **Gas**: MQ6 (LP/butano) y MQ4 (gas natural/metano)
- **Ambiental**: DHT22 (temperatura y humedad)
- **Distancia**: HC-SR04 (ultrasónico frontal) y VL53L0X (láser para barrido)
- **Obstáculos / bordes**: FC51 (IR lateral) y TCRT5000 (IR piso)
- **Orientación**: MPU6050 (IMU — acelerómetro + giroscopio)
- **Energía**: paquete 2S Li-Ion 18650 con BMS

### Capa de comunicación
El rover publica telemetría y alertas vía **MQTT con TLS y QoS 1** cada 2 segundos.
Un puente de ingesta autentica y transforma los mensajes hacia **Supabase** (PostgreSQL),
que actúa como fuente de verdad para todos los clientes.

### Capa de aplicaciones
Web y móvil consumen únicamente la API de Supabase — nunca se conectan directamente al broker
ni escriben telemetría. El firmware es la autoridad para la detención inmediata del robot;
la nube es la fuente común de visualización e historial.

---

## Licencia

[MIT](LICENSE)
