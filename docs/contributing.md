# Guía de contribución

---

## Estructura del proyecto

Este es un **repositorio umbrella** que integra tres repositorios independientes como
submódulos de Git:

| Directorio | Repositorio | Descripción |
|------------|-------------|-------------|
| `web/`     | <!-- TODO: agregar URL del repositorio web --> | Aplicación web |
| `mobile/`  | <!-- TODO: agregar URL del repositorio móvil --> | Aplicación móvil |
| `iot/`     | <!-- TODO: agregar URL del repositorio IoT --> | Firmware ESP32 |

Cada repositorio tiene su propio flujo de trabajo, issues y pull requests.
Este repositorio umbrella solo gestiona la sincronización y la documentación compartida.

---

## Configuración inicial

### 1. Clonar con submódulos

```bash
git clone --recurse-submodules <URL-de-este-repositorio>
```

### 2. Actualizar submódulos

```bash
git submodule update --init --recursive
```

### 3. Trabajar en un submódulo

Cada submódulo es un repositorio Git independiente. Para hacer cambios:

```bash
cd web/          # o mobile/ o iot/
git checkout main
# ... hacer cambios, commits, push en el repositorio del submódulo
```

Después de actualizar un submódulo, actualizar la referencia en el umbrella:

```bash
cd ..            # volver a la raíz del umbrella
git add web/
git commit -m "chore: update web submodule to <commit-hash>"
git push
```

---

## Sincronización automática

El workflow [`.github/workflows/sync-submodules.yml`](../.github/workflows/sync-submodules.yml)
sincroniza automáticamente los submódulos cada 2 horas mediante un commit directo a la
rama principal. Como todos los repositorios son públicos, no requiere secretos adicionales;
usa los permisos de escritura del `GITHUB_TOKEN` integrado en GitHub Actions.

### Ejecución manual

El workflow también tiene un botón **"Run workflow"** disponible en
**Actions → Sync submodules** para sincronizar manualmente cuando sea necesario.

---

## Convenciones de commits

Este proyecto sigue [Conventional Commits](https://www.conventionalcommits.org/):

```
<tipo>(<alcance>): <descripción corta>

[cuerpo opcional]

[pie de página opcional]
```

| Tipo | Uso |
|------|-----|
| `feat` | Nueva funcionalidad |
| `fix` | Corrección de error |
| `docs` | Solo documentación |
| `chore` | Mantenimiento, dependencias, CI |
| `refactor` | Refactorización sin cambio de comportamiento |
| `test` | Tests nuevos o modificados |
| `perf` | Mejoras de rendimiento |

**Alcance** (opcional): `web`, `mobile`, `iot`, `backend`, `docs`, `ci`

Ejemplos:
```
feat(iot): agregar deduplicación por boot_id y sequence
fix(backend): corregir conversión de timestamp a UTC
docs: actualizar contrato JSON con campo clock_quality
chore(ci): ajustar cron de sincronización a cada 2 h
```

---

## Ramas

| Rama | Propósito |
|------|-----------|
| `main` | Código listo para producción / entrega |
| `develop` | Integración de features en curso |
| `feat/<nombre>` | Feature nueva |
| `fix/<nombre>` | Corrección de error |
| `chore/<nombre>` | Mantenimiento |
| `chore/sync-submodules` | Rama automática del workflow de sincronización |

---

## Pull Requests

- Abre el PR contra `develop`, no directamente contra `main`
- Describe qué cambia y por qué (no solo qué hace el código)
- Enlaza el issue relacionado con `Closes #<número>`
- Asegúrate de que los tests pasen antes de solicitar revisión
- Un PR de submódulo debe incluir la referencia actualizada en el umbrella

---

## Entornos

| Entorno | Propósito |
|---------|-----------|
| Local | Desarrollo con datos sintéticos |
| Integración | Pruebas con identidad y broker propios |
| Demostración | Domicilio piloto autorizado |

Mantén proyectos y credenciales separados entre entornos.

---

## Versionado del contrato JSON

- Cambios **aditivos** (nuevos campos opcionales) → mantienen `schema_version: "1.0"`
- Cambios de **significado** o **eliminaciones** → crean `schema_version: "2.0"` con:
  - Adaptador de compatibilidad en el backend
  - Periodo de transición documentado
  - Despliegue en orden: backend → firmware → clientes

**No desplegar** cambios destructivos de esquema de forma automática.
**No eliminar** datos de un domicilio con un script genérico.

---

## Normas de privacidad y datos

- Los datos de telemetría solo se usan para las finalidades declaradas en la instalación
- El video de la ESP32-CAM no forma parte del JSON de telemetría y tiene tratamiento separado
- La retención y eliminación se aplican por domicilio y propósito
- Cualquier prueba con datos reales del domicilio piloto requiere autorización explícita
