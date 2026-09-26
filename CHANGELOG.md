# Changelog

Todos los cambios relevantes de **SnackCheck - Nutrition Verdict** se documentan en este archivo.

El proyecto sigue el versionado semántico **SemVer** (`MAJOR.MINOR.PATCH`).

---

## [1.0.0] - 2026-09-26

### Added

- Creado el endpoint `POST /nutrition-check`.
- Añadida validación de códigos de barras.
- Añadida respuesta de autodocumentación para solicitudes vacías.
- Integración con la API de Open Food Facts.
- Extracción de nombre del producto, marca, Nutri-Score, azúcar, sal, grasa y energía.
- Clasificación del azúcar en `low`, `medium` y `high`.
- Clasificación de sal y grasa en `high` y `not_high`.
- Cálculo de `concern_score`.
- Cálculo de `nutrient_verdict`.
- Clasificación del Nutri-Score.
- Regla para seleccionar el peor resultado entre `nutrient_verdict` y `nutriscore_verdict`.
- Fallback a `nutrient_verdict` cuando Nutri-Score no está disponible.
- Tres rutas de generación de texto mediante Groq:
  - `healthy`
  - `moderate`
  - `unhealthy`
- Formato final de respuesta en texto plano.
- Añadidos marcadores visuales 🟢, 🟡 y 🔴 al veredicto final y a las clasificaciones de nutrientes.
- Manejo estructurado de errores `400` y `404`.
- Respuesta específica para productos con datos nutricionales insuficientes.
- Documentación inline en los nodos principales.
- README del proyecto.
- Diagrama del workflow en Excalidraw.
- Registro de pruebas `TC-001` a `TC-012`.

### Changed

- Mejorado el formato del mensaje final para mantener secciones y saltos de línea claros en la respuesta de texto plano.
- Mejorados los prompts de IA para que respeten exactamente las clasificaciones nutricionales calculadas por el workflow.
- La solicitud vacía `{}` devuelve ahora autodocumentación con HTTP `200`.
- La respuesta de datos nutricionales insuficientes utiliza HTTP `200` según la especificación del proyecto.

### Fixed

- Corregida la comparación del Nutri-Score `a/b`, que inicialmente utilizaba valores en mayúsculas mientras Open Food Facts devuelve valores en minúsculas.
- Corregida la lógica del veredicto `moderate`, cambiando la combinación de condiciones de `AND` a `OR`.
- Corregida la descripción generada por IA para evitar describir como `high` un nutriente clasificado como `medium`.
- Corregido el código HTTP de la respuesta de datos nutricionales insuficientes.

---

## Versionado

Las futuras versiones seguirán estas reglas:

- `PATCH` (`1.0.1`) para correcciones de errores.
- `MINOR` (`1.1.0`) para nuevas funcionalidades compatibles.
- `MAJOR` (`2.0.0`) para cambios importantes o incompatibles.