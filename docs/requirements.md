# Requerimientos

Requerimientos funcionales (RF) y no funcionales (RNF) del sistema Robot Centinela.
Cada uno indica su prioridad y el subsistema responsable.

**Prioridades**: `Must` — MVP · `Should` — mejora del mes · `Could` — ampliación futura

---

## Requerimientos funcionales

### RF01 · Acceso autenticado · Must · Backend + Clientes
Web y móvil permiten iniciar y cerrar sesión y recuperar acceso mediante Supabase Auth.
Ningún dato se expone antes de validar la sesión.
Una sesión vencida bloquea nuevas consultas y permite autenticar de nuevo sin revelar otro domicilio.

### RF02 · Ámbito de domicilio · Must · Backend
El usuario ve únicamente domicilios y robots para los que tiene membresía activa.
Un usuario de otro domicilio obtiene cero filas o rechazo al consultar un UUID conocido
(también en Realtime y Storage).

### RF03 · Ingesta MQTT · Must · Backend + IoT
El puente consume `sentinel/data` y `sentinel/alerta` autenticados y los adapta al contrato
versionado sin cambiar el significado físico.
Un JSON válido de `centinela_01` se guarda; otra identidad autenticada no puede suplantarlo.

### RF04 · Idempotencia · Must · Backend + IoT
Frames únicos por `(device_id, boot_id, sequence)` se persisten una vez y reciben ACK de
aplicación después del commit.
Publicar tres veces el mismo frame produce una fila y un mismo identificador;
un contenido distinto con la misma identidad produce conflicto auditado.

### RF05 · Catálogo de capacidades · Must · Backend + Diseño
Las interfaces muestran sensores instalados, gas objetivo, unidad, límites del método y estado
de calibración.
Sin sensor específico instalado, la tarjeta indica «No instalado» — no muestra cero ni una
lectura MQ reinterpretada.

### RF06 · Dashboard común · Must · Web + Móvil
Ambas apps muestran gas con calidad, temperatura, humedad, batería, robot, conectividad,
hora y misión a partir de la misma API.
Para un frame de prueba, ambas muestran el mismo valor, unidad y `event_id`;
los datos inexistentes aparecen como «Sin dato».

### RF07 · Sincronización en vivo · Must · Clientes
La suscripción dispara recuperación de frames completos ordenados por `stream_version`.
Un evento duplicado no parpadea ni duplica un punto en la gráfica.

### RF08 · Recuperación tras desconexión · Must · Clientes + Backend
Al reconectar se consultan deltas persistidos y snapshot, sin depender de Realtime para
reproducción.
Tras 60 s desconectado, el cliente recupera todos los frames retenidos y converge al mismo
watermark que el otro cliente.

### RF09 · Frescura por canal · Must · Clientes
Cada tarjeta y gráfica conserva `captured_at`, `received_at` y calidad del reloj; indica la
edad del dato.
Un frame atrasado no reemplaza una lectura reciente; un canal sin actualización cambia a
«dato atrasado» aunque otros canales sigan activos.

### RF10 · Historial ambiental · Must · Backend + Clientes
Permite consultar por rango temporal, robot, misión, canal y calidad con paginación y
agregación explícita.
Un filtro de un día en zona local corresponde al intervalo UTC correcto; mantiene huecos en
periodos de fallo.

### RF11 · Mapa persistente · Must · Clientes + IoT
Muestra la versión de mapa validada, habitaciones, zonas prohibidas, estaciones y leyenda.
Una misión muestra su versión histórica aunque exista una versión más reciente;
mapa ausente ofrece explicación y listado de estaciones.

### RF12 · Posición y recorrido · Must · Clientes
Dibuja pose y trayectoria con incertidumbre, tiempo y calidad; permite seguir al robot o
explorar el mapa.
Una pose `null` no se dibuja en el origen; una pose obsoleta usa símbolo de «dato atrasado»
y no animación de movimiento.

### RF13 · Muestreo por estación · Must · IoT + Backend
Asocia muestra con estación, habitación, mapa, altura y ventana temporal cuando la posición
sea suficientemente confiable.
Una lectura sin posición permanece en historial temporal y no se asigna automáticamente a
una habitación.

### RF14 · Resumen de misión · Should · Backend + Clientes
Al completarse una misión se genera un resumen con duración, cobertura, alertas, lecturas
por habitación e incidencias de posición.
El resumen incluye la versión de mapa y permanece disponible tras actualizar el mapa.

### RF15 · Exportación de datos · Should · Backend + Clientes
Exportar CSV de consulta y resumen de misión con cabecera, unidades, zona horaria y
versión de esquema.

### RF16 · Preferencias de usuario · Must · Clientes
Elegir tema claro, oscuro o sistema; formato local de fecha y hora; idioma inicial español.
Las preferencias se aplican sin reiniciar sesión y persisten entre sesiones del mismo usuario.

---

## Requerimientos no funcionales

### RNF01 · Latencia de visualización · Must
El indicador de dato fresco refleja la última muestra en tiempo definido por el SLO del
equipo, medido desde `received_at`.
Aplica en LAN con broker local y con Supabase accesible.

### RNF02 · Disponibilidad del pipeline de ingesta · Must
El pipeline de ingesta tiene un objetivo de disponibilidad definido por el equipo.
Una caída del servicio web no interrumpe la adquisición ni la seguridad local del robot.

### RNF03 · Seguridad de transporte · Must
Todas las comunicaciones externas usan TLS con verificación de servidor.
No se usan brokers MQTT públicos anónimos para datos del domicilio.

### RNF04 · Aislamiento de datos entre domicilios · Must
RLS garantiza que ninguna consulta SQL, suscripción Realtime ni acceso a Storage cruce
el límite de domicilio, incluso si la aplicación omite filtros.

### RNF05 · Idempotencia de ingesta · Must
Entregar el mismo frame más de una vez produce exactamente una fila en base de datos.
El backend distingue duplicado de conflicto de identidad y audita ambos casos.

### RNF06 · Accesibilidad web · Should
Objetivo WCAG 2.2 nivel AA en flujos incluidos: contraste, teclado, nombre accesible,
foco visible, manejo de errores y reflow a 400 % de zoom.

### RNF07 · Mantenibilidad y versionado · Must
El contrato JSON usa `schema_version`; cambios aditivos mantienen `v1`.
Cambios de significado o eliminaciones crean `v2` con adaptador y periodo de transición.
El backend acepta ambas versiones durante la transición antes de actualizar clientes.

### RNF08 · Retención y eliminación de datos · Must
Retención y eliminación se aplican por domicilio y propósito, preservando dependencias
referenciales.
Video no forma parte del JSON de telemetría.
No se eliminan datos de un domicilio por un script genérico sin un plan específico.

### RNF09 · Rollback y continuidad · Must
Los clientes pueden volver a una versión compatible; el firmware dispone de recuperación
de configuración.
Los cambios de datos usan un plan específico de restauración o migración inversa probada.
Una migración que pierde datos no se revierte con una etiqueta Git.

### RNF10 · Auditoría de seguridad · Must
Cada frame de telemetría y cada alerta conservan `device_id`, `captured_at` y `received_at`
para trazabilidad completa.
Los eventos de sesión de usuario y los cambios de membresía son auditables.
