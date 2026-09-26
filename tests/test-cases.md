# Casos de prueba - SnackCheck

## Resumen

Breve descripción del objetivo de las pruebas.

## Resumen de pruebas

| ID | Caso | Tipo | Estado | Qué comprobamos|
|---|---|---|---|---|
| TC-001 | Producto válido end-to-end | Funcional | PASS | Que el flujo completo funciona |
| TC-002 | Nutri-Score A/B | Funcional | PASS | Ruta healthy |
| TC-003 | Nutri-Score C | Funcional | PASS | Ruta moderate |
| TC-004 | Nutri-Score D/E | Funcional | PASS | Ruta unhealthy |
| TC-005 | Sin Nutri-Score | Funcional | PASS | Fallback a nutrient_verdict |
| TC-006 | Datos insuficientes | Error | PASS | Respuesta de datos insuficientes, HTTP 200 |
| TC-007 | Barcode no numérico | Validación | PASS | Error de validación, HTTP 400 |
| TC-008 | Solicitud vacía | Validación / Autodocumentación | PASS | HTTP 200 + autodocumentación |
| TC-009 | Producto no encontrado | Error | PASS | HTTP 404 |
| TC-010 | Open Food Facts | Integración | PASS | Datos reales recuperados correctamente |
| TC-011 | Generación IA | Integración / IA | PASS | Mensaje coherente con `final_verdict` y clasificaciones |
| TC-012 | Tiempo de respuesta | Rendimiento | PASS | Tiempo medio observado: ~936 ms |

## TC-001 - Producto válido end-to-end

**Objetivo:**  
Comprobar que un código de barras válido recorre correctamente todo el workflow, desde la recepción de la solicitud hasta la generación y devolución del veredicto final.

**Tipo de prueba:**  
Funcional / End-to-end

**Precondiciones:**

- El workflow `SnackCheck - Nutrition Verdict` está activo.
- La instancia local de n8n está disponible.
- La conexión con Open Food Facts funciona correctamente.
- Las credenciales de Groq están configuradas.
- El endpoint `/nutrition-check` está disponible.

**Entrada:**

```json
{
  "barcode": "5449000000996"
}
```

**Producto utilizado:**  
Coca-Cola

**Pasos de prueba:**

1. Abrir Hoppscotch.
2. Seleccionar el método HTTP `POST`.
3. Introducir el endpoint:

   ```text
   http://localhost:5678/webhook/nutrition-check
   ```

4. Seleccionar `application/json` como tipo de contenido.
5. Enviar el cuerpo JSON con el código de barras `5449000000996`.
6. Comprobar el código de estado HTTP recibido.
7. Revisar la respuesta final.
8. Confirmar en n8n que el workflow se ha ejecutado correctamente y ha alcanzado la ruta correspondiente al veredicto `unhealthy`.

**Resultado esperado:**

- HTTP `200`.
- El producto es encontrado correctamente en Open Food Facts.
- Producto: `Coca-Cola`.
- Azúcar: `10.6 g/100g`.
- `sugar_level = medium`.
- Sal: `0 g/100g`.
- `salt_level = not_high`.
- Grasa: `0 g/100g`.
- `fat_level = not_high`.
- `concern_score = 0`.
- `nutrient_verdict = healthy`.
- `nutriscore_grade = e`.
- `nutriscore_verdict = unhealthy`.
- Se aplica la regla del peor veredicto.
- `final_verdict = unhealthy`.
- Se ejecuta la ruta de IA correspondiente a `unhealthy`.
- El veredicto final incluye un marcador visual 🔴.
- El azúcar `medium` se muestra con marcador 🟡.
- La sal y la grasa `not_high` se muestran con marcador 🟢.
- La respuesta final mantiene secciones separadas mediante saltos de línea.
- La respuesta final se devuelve como texto plano.

**Resultado obtenido:**

- HTTP `200 OK`.
- Producto: `Coca-Cola`.
- Marca: `COCA-COLA SERVICES SA/NV`.
- Azúcar: `10.6 g/100g (medium)`.
- Sal: `0 g/100g (not_high)`.
- Grasa: `0 g/100g (not_high)`.
- Nutri-Score: `e`.
- Concern score: `0/3`.
- Veredicto final: `UNHEALTHY`.
- El workflow finalizó correctamente.
- Tiempo registrado en Hoppscotch: `1365 ms`.
- Tiempo registrado en n8n: aproximadamente `1.329 s`.
- Se generó correctamente un párrafo explicativo mediante IA.
- En la comprobación final del formato, el veredicto `UNHEALTHY` se mostró con marcador 🔴.
- El azúcar `medium` se mostró con marcador 🟡.
- La sal y la grasa `not_high` se mostraron con marcador 🟢.
- El mensaje final mantuvo correctamente las distintas secciones y saltos de línea.

**Estado:**  
`PASS`

**Notas:**  

Este caso verifica especialmente la regla de combinación de veredictos. Aunque el `concern_score` es `0` y la lectura basada en nutrientes es `healthy`, el Nutri-Score `e` produce una lectura `unhealthy`, por lo que el resultado final debe ser `unhealthy`.

En una ejecución inicial, la explicación generada por IA utilizó la expresión `"relatively high sugar content"` aunque el azúcar estaba clasificado como `medium`. Esta observación se revisó en TC-011, se reforzó el prompt y la re-ejecución posterior confirmó que la IA respeta correctamente la clasificación `medium`.

## TC-002 - Producto con Nutri-Score A/B

**Objetivo:**  
Comprobar que un producto con Nutri-Score `A` o `B`, y sin nutrientes clasificados como `high`, recorre correctamente la ruta `healthy` y devuelve un veredicto final saludable.

**Tipo de prueba:**  
Funcional / Clasificación

**Precondiciones:**  

- El workflow `SnackCheck - Nutrition Verdict` está activo.
- La instancia local de n8n está disponible.
- Open Food Facts está disponible.
- Las credenciales de Groq están configuradas.
- El endpoint `/nutrition-check` está disponible.
- Se dispone de un producto real con Nutri-Score `A` o `B` y datos completos de azúcar, sal y grasa.

**Entrada:**

```json
{
  "barcode": "8480000171511"
}
```

**Producto utilizado:**  
Tomate frito Hacendado

**Pasos de prueba:**

1. Enviar una petición `POST` a `/nutrition-check` desde Hoppscotch.
2. Comprobar que Open Food Facts encuentra el producto.
3. Revisar las clasificaciones de azúcar, sal y grasa.
4. Comprobar `concern_score`.
5. Verificar `nutrient_verdict`.
6. Verificar `nutriscore_grade` y `nutriscore_verdict`.
7. Comprobar `final_verdict`.
8. Confirmar que se ejecuta `AI - Healthy Verdict`.
9. Revisar la respuesta final en Hoppscotch y la ejecución completa en n8n.

**Resultado esperado:**

- HTTP `200`.
- Producto: `Tomate frito`.
- Marca: `Hacendado`.
- Azúcar: `7.2 g/100g`.
- `sugar_level = medium`.
- Sal: `1 g/100g`.
- `salt_level = not_high`.
- Grasa: `3 g/100g`.
- `fat_level = not_high`.
- Ningún nutriente clasificado como `high`.
- `concern_score = 0`.
- `nutrient_verdict = healthy`.
- `nutriscore_grade = b`.
- `nutriscore_verdict = healthy`.
- `final_verdict = healthy`.
- Se ejecuta la ruta `AI - Healthy Verdict`.
- Se devuelve una respuesta final en texto plano.

**Resultado obtenido:**

### Primera ejecución

- Open Food Facts devolvió correctamente `nutriscore_grade = b`.
- El nodo `Check - Nutri-Score A or B?` no reconoció el valor y envió el ítem por la rama `false`.
- El flujo continuó por las comprobaciones de Nutri-Score posteriores y produjo un veredicto final incorrecto.
- Se identificó que las condiciones del nodo comparaban contra `A` y `B` en mayúsculas, mientras que Open Food Facts devuelve `a` y `b` en minúsculas.

### Corrección aplicada

- Se modificaron las comparaciones del nodo `Check - Nutri-Score A or B?` para utilizar `a` y `b` en minúsculas.

### Segunda ejecución

- HTTP `200 OK`.
- Producto: `Tomate frito`.
- Marca: `Hacendado`.
- Azúcar: `7.2 g/100g (medium)`.
- Sal: `1 g/100g (not_high)`.
- Grasa: `3 g/100g (not_high)`.
- Nutri-Score: `b`.
- Concern score: `0/3`.
- `nutrient_verdict = healthy`.
- `nutriscore_verdict = healthy`.
- Veredicto final: `HEALTHY`.
- Se ejecutó correctamente la ruta `AI - Healthy Verdict`.
- Se generó un párrafo explicativo coherente con el veredicto.
- Tiempo registrado en Hoppscotch: `934 ms`.
- Tiempo registrado en n8n: aproximadamente `912 ms`.
- El workflow finalizó correctamente.

**Estado:**  
`PASS`

**Notas:**  

La primera ejecución de este caso permitió detectar un error de sensibilidad a mayúsculas y minúsculas en la clasificación del Nutri-Score. Open Food Facts devolvía `b`, mientras que el nodo comparaba el valor con `A` y `B`.

Después de corregir las condiciones y repetir exactamente el mismo caso de prueba, el producto siguió correctamente la ruta `healthy` y el resultado coincidió con el comportamiento esperado.

Este caso demuestra el ciclo de prueba, detección del error, corrección y re-ejecución del mismo test.

## TC-003 - Producto con Nutri-Score C

**Objetivo:**  
Comprobar que un producto con Nutri-Score `C` produce una lectura `moderate` y que esta prevalece en el veredicto final cuando no existe una lectura `unhealthy`.

**Tipo de prueba:**  
Funcional / Clasificación

**Precondiciones:**  

- El workflow `SnackCheck - Nutrition Verdict` está activo.
- La instancia local de n8n está disponible.
- Open Food Facts está disponible.
- Las credenciales de Groq están configuradas.
- El endpoint `/nutrition-check` está disponible.
- Se dispone de un producto real con Nutri-Score `C` y datos completos de azúcar, sal y grasa.

**Entrada:**

```json
{
  "barcode": "8480000683229"
}
```

**Producto utilizado:**  
Cuajada Hacendado

**Pasos de prueba:**

1. Enviar una petición `POST` a `/nutrition-check` desde Hoppscotch.
2. Comprobar que Open Food Facts encuentra el producto.
3. Revisar las clasificaciones de azúcar, sal y grasa.
4. Comprobar `concern_score`.
5. Verificar `nutrient_verdict`.
6. Confirmar que `nutriscore_grade = c`.
7. Verificar que `nutriscore_verdict = moderate`.
8. Comprobar la lógica del nodo `Check - Any Verdict Moderate?`.
9. Confirmar que `final_verdict = moderate`.
10. Verificar que se ejecuta `AI - Moderate Verdict`.
11. Revisar la respuesta final.

**Resultado esperado:**

- HTTP `200`.
- Producto: `Cuajada`.
- Marca: `Hacendado`.
- Azúcar: `6.4 g/100g`.
- `sugar_level = medium`.
- Sal: `0.16 g/100g`.
- `salt_level = not_high`.
- Grasa: `5.1 g/100g`.
- `fat_level = not_high`.
- `concern_score = 0`.
- `nutrient_verdict = healthy`.
- `nutriscore_grade = c`.
- `nutriscore_verdict = moderate`.
- Al existir una lectura `moderate` y ninguna `unhealthy`, se aplica la regla del peor veredicto.
- `final_verdict = moderate`.
- Se ejecuta la ruta `AI - Moderate Verdict`.
- Se devuelve una respuesta final en texto plano.

**Resultado obtenido:**

### Primera ejecución

- HTTP `200 OK`.
- Open Food Facts devolvió correctamente `nutriscore_grade = c`.
- `concern_score = 0`.
- `nutrient_verdict = healthy`.
- `nutriscore_verdict = moderate`.
- Sin embargo, el workflow devolvió incorrectamente `final_verdict = healthy`.
- La respuesta final mostró `SnackCheck Verdict: HEALTHY`.
- Tiempo registrado en Hoppscotch: `1013 ms`.

Se identificó que el nodo `Check - Any Verdict Moderate?` estaba configurado con una condición `AND`:

```text
nutrient_verdict = moderate
AND
nutriscore_verdict = moderate
```

Esta condición solo era verdadera si **ambos** veredictos eran `moderate`. En este caso, `nutrient_verdict = healthy` y `nutriscore_verdict = moderate`, por lo que el ítem seguía incorrectamente la rama `false`.

### Corrección aplicada

Se modificó la combinación de condiciones del nodo `Check - Any Verdict Moderate?` de `AND` a `OR`:

```text
nutrient_verdict = moderate
OR
nutriscore_verdict = moderate
```

De esta forma, basta con que cualquiera de las dos lecturas sea `moderate` para que el veredicto final sea `moderate`, siempre que previamente se haya descartado un resultado `unhealthy`.

### Segunda ejecución

- HTTP `200 OK`.
- Producto: `Cuajada`.
- Marca: `Hacendado`.
- Azúcar: `6.4 g/100g (medium)`.
- Sal: `0.16 g/100g (not_high)`.
- Grasa: `5.1 g/100g (not_high)`.
- Nutri-Score: `c`.
- Concern score: `0/3`.
- `nutrient_verdict = healthy`.
- `nutriscore_verdict = moderate`.
- Veredicto final: `MODERATE`.
- Se ejecutó correctamente la ruta `AI - Moderate Verdict`.
- Se generó un párrafo explicativo coherente con el veredicto moderado.
- Tiempo registrado en Hoppscotch: `876 ms`.
- El workflow finalizó correctamente.

**Estado:**  
`PASS`

**Notas:**  

La primera ejecución permitió detectar un error lógico en la combinación de condiciones del nodo `Check - Any Verdict Moderate?`. La regla del proyecto establece que debe utilizarse el peor de los dos veredictos, por lo que una lectura `moderate` debe ser suficiente para producir un resultado final `moderate` si no existe previamente una lectura `unhealthy`.

Tras cambiar la condición de `AND` a `OR` y repetir el mismo caso de prueba, el workflow produjo el resultado esperado.

Este caso documenta el ciclo de prueba, detección del error, corrección y re-ejecución.

## TC-004 - Producto con Nutri-Score D/E

**Objetivo:**  
Comprobar que un producto con Nutri-Score `D` o `E` recibe una lectura `unhealthy` y que esta prevalece en el veredicto final.

**Tipo de prueba:**  
Funcional / Clasificación

**Precondiciones:**  

- El workflow `SnackCheck - Nutrition Verdict` está activo.
- La instancia local de n8n está disponible.
- Open Food Facts está disponible.
- Las credenciales de Groq están configuradas.
- El endpoint `/nutrition-check` está disponible.
- Se dispone de un producto real con Nutri-Score `D` o `E` y datos nutricionales completos.

**Entrada:**

```json
{
  "barcode": "8402001041945"
}
```

**Producto utilizado:**  
Dátiles Hacendado

**Pasos de prueba:**

1. Enviar una petición `POST` a `/nutrition-check` desde Hoppscotch.
2. Comprobar que Open Food Facts encuentra el producto.
3. Revisar las clasificaciones de azúcar, sal y grasa.
4. Comprobar `concern_score`.
5. Verificar `nutrient_verdict`.
6. Confirmar que `nutriscore_grade = d`.
7. Verificar que `nutriscore_verdict = unhealthy`.
8. Comprobar que se aplica la regla del peor veredicto.
9. Confirmar que `final_verdict = unhealthy`.
10. Verificar que se ejecuta `AI - Unhealthy Verdict`.
11. Revisar la respuesta final.

**Resultado esperado:**

- HTTP `200`.
- Producto: `Dátiles`.
- Marca: `Hacendado`.
- Azúcar: `68 g/100g`.
- `sugar_level = high`.
- Sal: `0 g/100g`.
- `salt_level = not_high`.
- Grasa: `0.5 g/100g`.
- `fat_level = not_high`.
- `concern_score = 1`.
- `nutrient_verdict = moderate`.
- `nutriscore_grade = d`.
- `nutriscore_verdict = unhealthy`.
- Se aplica la regla del peor veredicto.
- `final_verdict = unhealthy`.
- Se ejecuta la ruta `AI - Unhealthy Verdict`.
- Se devuelve una respuesta final en texto plano.

**Resultado obtenido:**

- HTTP `200 OK`.
- Producto: `Dátiles`.
- Marca: `Hacendado`.
- Azúcar: `68 g/100g (high)`.
- Sal: `0 g/100g (not_high)`.
- Grasa: `0.5 g/100g (not_high)`.
- Nutri-Score: `d`.
- Concern score: `1/3`.
- `nutrient_verdict = moderate`.
- `nutriscore_verdict = unhealthy`.
- Veredicto final: `UNHEALTHY`.
- Se ejecutó correctamente la ruta `AI - Unhealthy Verdict`.
- El texto generado por IA fue coherente con los datos del producto y con el veredicto final.
- Tiempo registrado en Hoppscotch: `641 ms`.
- Tiempo registrado en n8n: aproximadamente `850 ms`.
- El workflow finalizó correctamente.

**Estado:**  
`PASS`

**Notas:**  

Durante la revisión inicial se confundieron temporalmente los datos de esta ejecución con los del TC-003. El valor `6.4 g/100g` correspondía a la Cuajada Hacendado del caso anterior, mientras que el producto utilizado en este caso, Dátiles Hacendado, contiene `68 g/100g` de azúcar.

Tras revisar la ejecución correcta en n8n y la respuesta recibida en Hoppscotch, se confirmó que el workflow procesó correctamente los datos del producto.

Este caso verifica además dos niveles de la lógica de clasificación: el azúcar `high` genera `concern_score = 1` y `nutrient_verdict = moderate`, pero el Nutri-Score `d` produce `nutriscore_verdict = unhealthy`; por la regla del peor veredicto, el resultado final correcto es `unhealthy`.

## TC-005 - Producto sin Nutri-Score

**Objetivo:**  
Comprobar que, cuando Open Food Facts no proporciona un Nutri-Score válido, el workflow utiliza únicamente `nutrient_verdict` para determinar el resultado final.

**Tipo de prueba:**  
Funcional / Fallback

**Precondiciones:**  

- El workflow `SnackCheck - Nutrition Verdict` está activo.
- La instancia local de n8n está disponible.
- Open Food Facts está disponible.
- Las credenciales de Groq están configuradas.
- El endpoint `/nutrition-check` está disponible.
- Se dispone de un producto existente con datos completos de azúcar, sal y grasa, pero sin un Nutri-Score válido.

**Entrada:**

```json
{
  "barcode": "8410109115543"
}
```

**Producto utilizado:**  
100% cacao puro natural — CHOCOLATES VALOR S.A.

**Pasos de prueba:**

1. Enviar una petición `POST` a `/nutrition-check` desde Hoppscotch.
2. Comprobar que Open Food Facts encuentra el producto.
3. Verificar que están disponibles los valores de azúcar, sal y grasa.
4. Confirmar que el Nutri-Score está ausente o se devuelve como `unknown`.
5. Revisar las clasificaciones nutricionales.
6. Comprobar `concern_score`.
7. Verificar `nutrient_verdict`.
8. Confirmar que el workflow aplica el fallback y utiliza únicamente `nutrient_verdict` como `final_verdict`.
9. Verificar que se ejecuta la ruta de IA correspondiente al veredicto final.
10. Revisar la respuesta final.

**Resultado esperado:**

- HTTP `200`.
- Producto: `100% cacao puro natural`.
- Marca: `CHOCOLATES VALOR S.A.`.
- Azúcar: `1 g/100g`.
- `sugar_level = low`.
- Sal: `0.05 g/100g`.
- `salt_level = not_high`.
- Grasa: `11 g/100g`.
- `fat_level = not_high`.
- `concern_score = 0`.
- `nutrient_verdict = healthy`.
- Nutri-Score ausente o `unknown`.
- No se utiliza un `nutriscore_verdict` válido para decidir el resultado final.
- `final_verdict = nutrient_verdict = healthy`.
- Se ejecuta la ruta `AI - Healthy Verdict`.
- Se devuelve una respuesta final en texto plano.

**Resultado obtenido:**

- HTTP `200 OK`.
- Producto: `100% cacao puro natural`.
- Marca: `CHOCOLATES VALOR S.A.`.
- Azúcar: `1 g/100g (low)`.
- Sal: `0.05 g/100g (not_high)`.
- Grasa: `11 g/100g (not_high)`.
- Nutri-Score: `unknown`.
- Concern score: `0/3`.
- Veredicto final: `HEALTHY`.
- El resultado es coherente con el fallback previsto: al no existir un Nutri-Score válido, el workflow utiliza la lectura nutricional.
- Se generó correctamente un párrafo explicativo correspondiente a la ruta `healthy`.
- Tiempo registrado en Hoppscotch: `864 ms`.
- La respuesta final se devolvió correctamente en texto plano.

**Estado:**  
`PASS`

**Notas:**  

Este caso confirma que la ausencia de Nutri-Score no bloquea el análisis ni produce un error. Como ninguno de los nutrientes está clasificado como `high`, el `concern_score` es `0` y la lectura nutricional es `healthy`.

Al aparecer el Nutri-Score como `unknown`, el workflow aplica correctamente la ruta de fallback y utiliza únicamente `nutrient_verdict` para establecer el veredicto final.

El resultado final `HEALTHY` coincide con el comportamiento esperado.

## TC-006 - Datos nutricionales insuficientes

**Objetivo:**  
Comprobar que un producto existente al que le falta al menos uno de los nutrientes obligatorios no se clasifica y devuelve una respuesta controlada de datos nutricionales insuficientes.

**Tipo de prueba:**  
Manejo de errores / Datos incompletos

**Precondiciones:**  

- El workflow `SnackCheck - Nutrition Verdict` está activo.
- La instancia local de n8n está disponible.
- Open Food Facts está disponible.
- El endpoint `/nutrition-check` está disponible.
- Se dispone de un producto existente en Open Food Facts al que le falta al menos uno de estos campos:
  - `sugars_100g`
  - `salt_100g`
  - `fat_100g`

**Entrada:**

```json
{
  "barcode": "5400606990555"
}
```

**Pasos de prueba:**

1. Enviar una petición `POST` a `/nutrition-check` desde Hoppscotch.
2. Confirmar que Open Food Facts encuentra el producto.
3. Verificar que falta al menos uno de los nutrientes requeridos.
4. Confirmar que el workflow sigue la ruta de datos nutricionales insuficientes.
5. Comprobar que no se ejecutan las reglas de clasificación.
6. Comprobar que no se calcula `concern_score`.
7. Confirmar que no se genera `final_verdict`.
8. Verificar que no se ejecutan los nodos de IA.
9. Revisar el código HTTP y el cuerpo JSON devuelto.

**Resultado esperado:**

- HTTP `200`.
- El producto existe en Open Food Facts.
- Se detectan datos nutricionales insuficientes.
- No se clasifica el producto.
- No se calcula `concern_score`.
- No se genera `nutrient_verdict`.
- No se genera `final_verdict`.
- No se ejecuta Groq.
- Se devuelve una respuesta JSON estructurada con:
  - `status = error`
  - `code = INSUFFICIENT_NUTRITION_DATA`
  - un mensaje explicando que no hay suficiente información nutricional;
  - los campos nutricionales requeridos;
  - una indicación de cómo resolver el problema.

**Resultado obtenido:**

- HTTP `200 OK`.
- Se devolvió correctamente una respuesta JSON estructurada.
- `status = error`.
- `code = INSUFFICIENT_NUTRITION_DATA`.
- Mensaje: el producto fue encontrado, pero no existe suficiente información nutricional para clasificarlo.
- Se indicaron como campos requeridos:
  - `sugars_100g`
  - `salt_100g`
  - `fat_100g`
- No se generó ningún veredicto nutricional.
- No se ejecutó la generación de texto mediante IA.
- Tiempo registrado en Hoppscotch: `168 ms`.

**Estado:**  
`PASS`

**Notas:**  

Este caso confirma que el workflow finaliza de forma temprana cuando faltan datos esenciales y evita ejecutar innecesariamente la lógica de clasificación y la llamada a IA.

## TC-007 - Código de barras no numérico

**Objetivo:**  
Comprobar que el sistema rechaza un código de barras que contiene caracteres no numéricos antes de consultar Open Food Facts.

**Tipo de prueba:**  
Validación / Error

**Precondiciones:**  

- El workflow `SnackCheck - Nutrition Verdict` está activo.
- La instancia local de n8n está disponible.
- El endpoint `/nutrition-check` está disponible.
- La validación del campo `barcode` está configurada antes de la llamada a Open Food Facts.

**Entrada:**

```json
{
  "barcode": "MG006069905AB"
}
```

**Pasos de prueba:**

1. Enviar una petición `POST` a `/nutrition-check` desde Hoppscotch.
2. Introducir un valor de `barcode` que contenga letras y números.
3. Comprobar que la validación detecta los caracteres no numéricos.
4. Confirmar que la solicitud no continúa hacia Open Food Facts.
5. Revisar el código HTTP devuelto.
6. Revisar el cuerpo JSON de la respuesta.

**Resultado esperado:**

- HTTP `400`.
- Se devuelve una respuesta JSON estructurada de validación.
- Se identifica el campo `barcode`.
- El mensaje indica que el código de barras debe contener únicamente números.
- Se incluye información sobre qué ocurrió, por qué ocurrió y cómo corregirlo.
- Se proporciona un ejemplo de código de barras válido.
- Open Food Facts no es consultado.
- No se ejecuta la lógica de clasificación.
- No se ejecuta Groq.

**Resultado obtenido:**

- HTTP `400 Bad Request`.
- `status = error`.
- Se devuelve un array `errors`.
- `code = INPUT_INVALID_BARCODE`.
- `message = "The barcode must contain numbers only."`.
- `field = barcode`.
- La respuesta explica que el valor contiene caracteres inválidos.
- La respuesta indica que Open Food Facts utiliza códigos de barras numéricos.
- Se incluye un ejemplo de entrada válida.
- La validación detiene correctamente el flujo antes de procesar el producto.
- Tiempo registrado en Hoppscotch: `60 ms`.

**Estado:**  
`PASS`

**Notas:**  

El workflow rechaza correctamente un código de barras alfanumérico y devuelve un error HTTP `400` claro y estructurado.

Este caso confirma que la validación de entrada se realiza antes de cualquier llamada externa, evitando consultas innecesarias a Open Food Facts y a los nodos posteriores del workflow.

## TC-008 - Solicitud vacía / Autodocumentación

**Objetivo:**  
Comprobar que una solicitud completamente vacía devuelve información de uso del servicio con HTTP `200`, sin intentar consultar ningún producto ni continuar con la lógica de clasificación.

**Tipo de prueba:**  
Validación / Autodocumentación

**Precondiciones:**  

- El workflow `SnackCheck - Nutrition Verdict` está activo.
- La instancia local de n8n está disponible.
- El endpoint `/nutrition-check` está disponible.
- El nodo `Validate - Barcode` distingue entre una solicitud completamente vacía y un campo `barcode` presente pero inválido.
- El nodo de respuesta utiliza dinámicamente `responseCode` y `response`.

**Entrada:**

```json
{}
```

**Pasos de prueba:**

1. Enviar una petición `POST` a `/nutrition-check` desde Hoppscotch.
2. Utilizar un cuerpo JSON completamente vacío: `{}`.
3. Comprobar que el workflow detecta la ausencia total de datos.
4. Verificar que no se consulta Open Food Facts.
5. Verificar que no se ejecuta la lógica de clasificación nutricional.
6. Verificar que no se ejecutan los nodos de IA.
7. Comprobar el código HTTP.
8. Revisar el JSON de autodocumentación devuelto.

**Resultado esperado:**

- HTTP `200`.
- Se devuelve una respuesta JSON informativa.
- `status = info`.
- Se identifica el servicio como `SnackCheck - Nutrition Verdict`.
- Se indica que debe enviarse un código de barras.
- Se informa del método HTTP `POST`.
- Se informa del endpoint `/nutrition-check`.
- Se indica que el campo requerido es `barcode`.
- Se incluye un ejemplo de petición válida.
- No se consulta Open Food Facts.
- No se calcula ningún veredicto.
- No se ejecuta Groq.

**Resultado obtenido:**

- HTTP `200 OK`.
- `status = info`.
- `service = SnackCheck - Nutrition Verdict`.
- `message = "Send a product barcode to receive a nutritional verdict."`.
- `method = POST`.
- `endpoint = /nutrition-check`.
- `required_field = barcode`.
- Se devolvió correctamente el ejemplo:

```json
{
  "barcode": "3017620422003"
}
```

- El workflow no intentó procesar ningún producto.
- Tiempo registrado en Hoppscotch: `56 ms`.

**Estado:**  
`PASS`

**Notas:**  

Este caso confirma que una solicitud completamente vacía se trata como una petición de ayuda/autodocumentación y no como un error de validación.

Tras introducir esta lógica, se realizaron además dos comprobaciones de regresión para asegurar que los errores de validación existentes siguieran funcionando:

```json
{
  "barcode": ""
}
```

Resultado: HTTP `400 Bad Request` con `code = INPUT_MISSING_BARCODE`.

```json
{
  "barcode": "ABC123"
}
```

Resultado: HTTP `400 Bad Request` con `code = INPUT_INVALID_BARCODE`.

Por tanto, el workflow distingue correctamente entre:

- solicitud vacía `{}` → HTTP `200` + autodocumentación;
- campo `barcode` vacío → HTTP `400`;
- código de barras no numérico → HTTP `400`.

## TC-009 - Producto no encontrado

**Objetivo:**  
Comprobar que un código de barras numérico y válido en formato, pero inexistente en Open Food Facts, supera la validación inicial y devuelve una respuesta controlada HTTP `404`.

**Tipo de prueba:**  
Manejo de errores / Integración

**Precondiciones:**  

- El workflow `SnackCheck - Nutrition Verdict` está activo.
- La instancia local de n8n está disponible.
- Open Food Facts está disponible.
- El endpoint `/nutrition-check` está disponible.
- Se utiliza un código de barras numérico que no corresponde a ningún producto registrado.

**Entrada:**

```json
{
  "barcode": "0000000000000"
}
```

**Pasos de prueba:**

1. Enviar una petición `POST` a `/nutrition-check` desde Hoppscotch.
2. Comprobar que el código de barras supera la validación por ser numérico.
3. Confirmar que se realiza la consulta a Open Food Facts.
4. Verificar que no se encuentra ningún producto asociado al código.
5. Confirmar que el workflow sigue la ruta `Product Not Found`.
6. Revisar el código HTTP y el cuerpo JSON devuelto.
7. Confirmar que no se ejecuta la clasificación nutricional ni la generación mediante IA.

**Resultado esperado:**

- HTTP `404`.
- Se devuelve una respuesta JSON estructurada.
- `status = error`.
- `code = PRODUCT_NOT_FOUND`.
- El mensaje indica que no se encontró ningún producto para el código proporcionado.
- Se identifica el campo `barcode`.
- Se incluye información sobre qué ocurrió, por qué pudo ocurrir y cómo corregirlo.
- Se proporciona un código de barras válido como ejemplo.
- No se ejecuta la clasificación nutricional.
- No se calcula `concern_score`.
- No se genera `final_verdict`.
- No se ejecuta Groq.

**Resultado obtenido:**

- HTTP `404 Not Found`.
- `status = error`.
- `code = PRODUCT_NOT_FOUND`.
- `message = "No product was found for the provided barcode."`.
- `field = barcode`.
- La respuesta indica que Open Food Facts no encontró un producto asociado al código.
- Se informa de que el código podría ser incorrecto o no existir en la base de datos.
- Se incluye como ejemplo el código `3017620422003`.
- El workflow finaliza correctamente por la ruta de producto no encontrado.
- Tiempo registrado en Hoppscotch: `256 ms`.

**Estado:**  
`PASS`

**Notas:**  

Este caso confirma que un código de barras puede ser válido desde el punto de vista de la validación de entrada y, aun así, no corresponder a ningún producto registrado en Open Food Facts.

El workflow diferencia correctamente este escenario de los errores de validación y devuelve un HTTP `404` específico sin continuar con la clasificación nutricional ni con la generación de texto mediante IA.

## TC-010 - Integración con Open Food Facts

**Objetivo:**  
Comprobar que el workflow se integra correctamente con la API real de Open Food Facts y recupera los campos necesarios para realizar el análisis nutricional.

**Tipo de prueba:**  
Integración

**Precondiciones:**  

- El workflow `SnackCheck - Nutrition Verdict` está activo.
- La instancia local de n8n tiene conexión a Internet.
- Open Food Facts está disponible.
- Se utilizan productos reales con información registrada en la API.

**Entradas utilizadas:**  

Este caso se valida reutilizando las ejecuciones realizadas en TC-001 a TC-006, que incluyen distintos productos y situaciones reales de la API:

```text
5449000000996 -> Coca-Cola
8480000171511 -> Tomate frito Hacendado
8480000683229 -> Cuajada Hacendado
8402001041945 -> Dátiles Hacendado
8410109115543 -> 100% cacao puro natural
5400606990555 -> Producto con datos nutricionales insuficientes
```

**Pasos de prueba:**

1. Enviar distintos códigos de barras reales al endpoint `/nutrition-check`.
2. Confirmar que el nodo HTTP Request consulta Open Food Facts.
3. Verificar que la API devuelve información del producto cuando existe.
4. Comprobar que el workflow extrae correctamente los campos utilizados.
5. Confirmar que los datos recuperados llegan a las etapas posteriores de clasificación.
6. Verificar también los escenarios de producto sin Nutri-Score y datos nutricionales insuficientes.

**Resultado esperado:**

- La API responde correctamente para productos existentes.
- El workflow recupera y utiliza, cuando están disponibles:
  - `product_name`
  - `brands`
  - `nutriscore_grade`
  - `sugars_100g`
  - `salt_100g`
  - `fat_100g`
  - `energy-kcal_100g`
- Los productos encontrados continúan hacia el análisis nutricional.
- Un Nutri-Score ausente o `unknown` se gestiona mediante fallback.
- Los productos con datos nutricionales incompletos se detienen antes de la clasificación.
- Los productos inexistentes se gestionan mediante la ruta HTTP `404`.

**Resultado obtenido:**

- Los productos reales utilizados en TC-001 a TC-005 fueron recuperados correctamente desde Open Food Facts.
- Los nombres, marcas, Nutri-Score y valores nutricionales se utilizaron correctamente en las etapas posteriores del workflow.
- TC-005 confirmó que la integración gestiona correctamente un Nutri-Score `unknown`.
- TC-006 confirmó que el workflow detecta información nutricional incompleta procedente de la API.
- TC-009 confirmó que un producto inexistente genera correctamente la ruta `PRODUCT_NOT_FOUND`.
- No se detectaron errores de integración con Open Food Facts durante las pruebas funcionales.

**Estado:**  
`PASS`

**Notas:**  

No fue necesario realizar una ejecución adicional exclusiva para este caso, ya que la integración con Open Food Facts quedó probada repetidamente durante los casos funcionales anteriores con diferentes productos y escenarios.

## TC-011 - Generación y coherencia del mensaje de IA

**Objetivo:**  
Comprobar que Groq genera una explicación coherente con el veredicto previamente calculado y que no modifica ni contradice las clasificaciones deterministas del workflow.

**Tipo de prueba:**  
Integración / IA

**Precondiciones:**  

- Las credenciales de Groq están configuradas correctamente.
- El workflow puede alcanzar las rutas `healthy`, `moderate` y `unhealthy`.
- El `final_verdict` se calcula antes de ejecutar la IA.
- Los prompts reciben los datos reales del producto y sus clasificaciones.

**Entradas utilizadas:**  

Se reutilizan las ejecuciones de varios casos anteriores:

```text
TC-001 -> Coca-Cola -> UNHEALTHY
TC-002 -> Tomate frito Hacendado -> HEALTHY
TC-003 -> Cuajada Hacendado -> MODERATE
TC-004 -> Dátiles Hacendado -> UNHEALTHY
TC-005 -> 100% cacao puro natural -> HEALTHY
```

Para la comprobación final de coherencia se volvió a utilizar:

```json
{
  "barcode": "5449000000996"
}
```

**Pasos de prueba:**

1. Revisar el `final_verdict` calculado antes de la llamada a IA.
2. Confirmar que se ejecuta únicamente la ruta de IA correspondiente.
3. Leer el párrafo generado.
4. Comparar el texto con:
   - `sugar_level`
   - `salt_level`
   - `fat_level`
   - Nutri-Score
   - `final_verdict`
5. Comprobar que la IA no cambia el veredicto.
6. Verificar que no describe como `high` un nutriente clasificado como `medium` o `not_high`.

**Resultado esperado:**

- Se ejecuta únicamente el nodo de IA correspondiente a `final_verdict`.
- La IA no modifica el veredicto.
- El tono es adecuado para `healthy`, `moderate` o `unhealthy`.
- El mensaje es breve y comprensible.
- El texto respeta las clasificaciones nutricionales calculadas.
- Un nutriente `medium` no debe describirse como `high`.
- Un nutriente `not_high` no debe describirse como `high`.

**Resultado obtenido:**

- HTTP `200 OK`.
- Producto probado: `Coca-Cola`.
- `final_verdict = unhealthy`.
- Se ejecutó correctamente `AI - Unhealthy Verdict`.
- Azúcar: `10.6 g/100g`.
- `sugar_level = medium`.
- La IA respetó esta clasificación y describió el azúcar como `"a medium amount of sugar"`.
- `salt_level = not_high` y `fat_level = not_high`; el texto no los presentó como altos.
- Nutri-Score: `e`.
- El texto mantuvo el veredicto `UNHEALTHY` y lo relacionó correctamente con el perfil nutricional general y el Nutri-Score `e`.
- Tiempo registrado en Hoppscotch: `897 ms`.
- La respuesta final se devolvió correctamente en texto plano.

**Estado:**  
`PASS`

**Notas:**  

Durante una ejecución anterior se observó que el modelo podía describir un nivel `medium` de azúcar como relativamente alto.

Se reforzaron los prompts para exigir que la IA respete exactamente las clasificaciones nutricionales calculadas por el workflow y que no cambie ni reinterprete el veredicto suministrado.

Tras repetir la prueba con Coca-Cola, el mensaje generado fue coherente con `sugar_level = medium`, con el resto de categorías nutricionales y con `final_verdict = unhealthy`.

La IA actúa únicamente como capa explicativa; la clasificación y el veredicto se calculan previamente mediante reglas deterministas.
## TC-012 - Tiempo de respuesta

**Objetivo:**  
Registrar el tiempo de respuesta observado durante ejecuciones completas del workflow y comprobar que el sistema finaliza correctamente sin errores inesperados ni timeouts.

**Tipo de prueba:**  
Rendimiento

**Precondiciones:**  

- El workflow está activo.
- Open Food Facts está disponible.
- Groq está disponible.
- Se utilizan ejecuciones completas que incluyen consulta externa, clasificación y generación de IA.

**Entradas utilizadas:**  

Se reutilizan los tiempos registrados durante las pruebas funcionales completas.

**Pasos de prueba:**

1. Recopilar los tiempos de respuesta registrados en Hoppscotch para varias ejecuciones completas.
2. Utilizar únicamente casos que recorran el flujo normal hasta una respuesta final.
3. Calcular el tiempo medio aproximado.
4. Comprobar que ninguna ejecución termina por timeout o error inesperado.

**Resultado esperado:**

- Las ejecuciones completas terminan correctamente.
- No se producen timeouts.
- Los tiempos quedan documentados.
- Se obtiene una referencia aproximada del rendimiento actual del workflow.

**Resultado obtenido:**

| Caso | Tiempo Hoppscotch |
|---|---:|
| TC-001 | 1365 ms |
| TC-002 | 934 ms |
| TC-003 | 876 ms |
| TC-004 | 641 ms |
| TC-005 | 864 ms |

Tiempo medio observado:

```text
(1365 + 934 + 876 + 641 + 864) / 5 = 936 ms
```

**Tiempo medio aproximado:** `936 ms` (`0.94 s`)

- Todas las ejecuciones incluidas finalizaron correctamente.
- No se observaron timeouts.
- Los tiempos variaron entre `641 ms` y `1365 ms`.
- Las respuestas que terminan antes de la clasificación completa, como errores de validación o autodocumentación, fueron considerablemente más rápidas y no se incluyeron en esta media.

**Estado:**  
`PASS`

**Notas:**  

El proyecto no define un umbral máximo obligatorio de rendimiento, por lo que este caso documenta el comportamiento observado en lugar de evaluar el workflow frente a un límite arbitrario.

El tiempo final depende también de servicios externos como Open Food Facts y Groq, por lo que puede variar entre ejecuciones.
