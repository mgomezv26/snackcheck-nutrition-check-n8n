# SnackCheck - Nutrition Verdict

Automatización en **n8n** que analiza un alimento envasado a partir de su código de barras, consulta sus datos reales en **Open Food Facts**, aplica reglas nutricionales deterministas y genera un veredicto final acompañado de una explicación breve mediante **IA con Groq**.

---

## Propósito

SnackCheck está pensado como el motor de una pequeña aplicación de consumo capaz de responder, de forma clara y amigable, a una pregunta sencilla: **¿es este producto una opción adecuada para el día a día o sería mejor reservarlo para un consumo ocasional?**

El workflow:

- recibe un código de barras;
- valida la entrada;
- consulta el producto en Open Food Facts;
- comprueba que existan los datos nutricionales necesarios;
- clasifica azúcar, sal y grasa;
- calcula una puntuación de preocupación;
- interpreta el Nutri-Score;
- combina ambas lecturas utilizando siempre la peor de las dos;
- genera un texto explicativo con IA adaptado al nivel del veredicto;
- devuelve una respuesta final fácil de leer.

Los tres posibles niveles son:

- `healthy`
- `moderate`
- `unhealthy`

> La IA no decide la clasificación. El veredicto se calcula previamente mediante reglas deterministas.

---

## Tecnologías utilizadas

- **n8n** — construcción y orquestación del workflow.
- **Open Food Facts API v2** — obtención de datos reales del producto.
- **Groq** — generación del texto explicativo final.
- **JavaScript** — validación de entrada y cálculo de `concern_score`.
- **Excalidraw** — diseño y documentación visual del flujo.

---

## Arquitectura del flujo

El proyecto sigue las cinco capas trabajadas durante la formación.

### 1. Capa de entrada

El workflow comienza con un webhook HTTP `POST`:

```text
/nutrition-check
```

La solicitud debe incluir un código de barras en el cuerpo JSON:

```json
{
  "barcode": "3017620422003"
}
```

La entrada se valida antes de realizar cualquier llamada externa. El código de barras debe:

- existir;
- no estar vacío;
- contener únicamente números.

Si la validación falla, el workflow devuelve una respuesta JSON estructurada con un error HTTP `400`.

---

### 2. Capa de datos

Con el código de barras validado, el workflow consulta Open Food Facts:

```text
GET https://world.openfoodfacts.org/api/v2/product/{barcode}.json?fields=product_name,brands,nutriscore_grade,nutriments
```

Se utilizan los siguientes campos:

- `product_name`
- `brands`
- `nutriscore_grade`
- `nutriments.sugars_100g`
- `nutriments.salt_100g`
- `nutriments.fat_100g`
- `nutriments['energy-kcal_100g']`

El campo de energía utiliza notación de corchetes porque su nombre contiene un guion.

Antes de continuar, el workflow comprueba:

1. que el producto exista;
2. que estén disponibles los valores de azúcar, sal y grasa necesarios para clasificarlo.

Si Open Food Facts no encuentra el producto, se devuelve una respuesta HTTP `404`.

Si el producto existe pero no contiene suficientes datos nutricionales, el workflow devuelve una respuesta controlada de datos insuficientes y **no genera ningún veredicto**.

---

### 3. Capa lógica

#### Clasificación nutricional

##### Azúcar

La cantidad de azúcar por 100 g se clasifica como:

| Valor | Clasificación |
|---|---|
| `< 5 g` | `low` |
| `5 - 22.5 g` | `medium` |
| `> 22.5 g` | `high` |

Los valores límite `5` y `22.5` pertenecen a la categoría `medium`.

##### Sal

| Valor | Clasificación |
|---|---|
| `<= 1.5 g` | `not_high` |
| `> 1.5 g` | `high` |

##### Grasa

| Valor | Clasificación |
|---|---|
| `<= 17.5 g` | `not_high` |
| `> 17.5 g` | `high` |

---

#### Puntuación de preocupación

El campo `concern_score` indica cuántos de los tres nutrientes han sido clasificados como `high`.

| Nutrientes `high` | `concern_score` |
|---:|---:|
| 0 | 0 |
| 1 | 1 |
| 2 | 2 |
| 3 | 3 |

El cálculo se realiza en un nodo **Code** sobre un único ítem.

---

#### Veredicto basado en nutrientes

El `concern_score` se interpreta de la siguiente forma:

| `concern_score` | `nutrient_verdict` |
|---:|---|
| 0 | `healthy` |
| 1 | `moderate` |
| 2 o 3 | `unhealthy` |

---

#### Clasificación del Nutri-Score

El campo `nutriscore_grade` se transforma en una segunda lectura independiente:

| Nutri-Score | `nutriscore_verdict` |
|---|---|
| `a` o `b` | `healthy` |
| `c` | `moderate` |
| `d` o `e` | `unhealthy` |

Si `nutriscore_grade` no existe o su valor es `unknown`, esta segunda lectura se ignora y el veredicto final se basa únicamente en `nutrient_verdict`.

---

#### Veredicto final

El workflow compara:

- `nutrient_verdict`
- `nutriscore_verdict`

y utiliza siempre el resultado más restrictivo:

```text
unhealthy > moderate > healthy
```

Por tanto:

- si cualquiera de las dos lecturas es `unhealthy` → `final_verdict = unhealthy`;
- si ninguna es `unhealthy` pero alguna es `moderate` → `final_verdict = moderate`;
- solo si ambas son `healthy` → `final_verdict = healthy`;
- si el Nutri-Score no está disponible → `final_verdict = nutrient_verdict`.

Esta separación permite detectar casos como una bebida con `concern_score = 0` pero Nutri-Score desfavorable.

---

### 4. Capa de inteligencia

Una vez calculado `final_verdict`, el workflow dirige el producto hacia uno de tres prompts de Groq.

#### Healthy

Genera un texto:

- positivo;
- alentador;
- breve;
- no médico;
- no alarmista.

#### Moderate

Genera un texto:

- positivo pero equilibrado;
- breve;
- destacando el aspecto nutricional que merece atención.

#### Unhealthy

Genera un texto:

- cauteloso;
- amable;
- no alarmista;
- no moralizante;
- orientado a presentar el producto como una opción más adecuada para consumo ocasional.

La IA recibe como contexto los datos reales del producto y el veredicto ya calculado, pero **no puede modificarlo ni reinterpretarlo**.

---

### 5. Capa de salida

Las tres rutas de IA se vuelven a unificar y se construye un mensaje final en texto plano.

El mensaje utiliza marcadores visuales para facilitar la interpretación:

- 🟢 `healthy`, `low` o `not_high`
- 🟡 `moderate` o `medium`
- 🔴 `unhealthy` o `high`

Ejemplo:

```text
🥗 🔴 SnackCheck Verdict: UNHEALTHY

Product: Coca-Cola
Brand: COCA-COLA SERVICES SA/NV

🟡 Sugar: 10.6 g/100g (medium)
🟢 Salt: 0 g/100g (not_high)
🟢 Fat: 0 g/100g (not_high)

Nutri-Score: e
Concern score: 0/3

[Explicación breve generada por IA]
```

La respuesta final utiliza:

```text
HTTP 200
Content-Type: text/plain; charset=utf-8
```

---

## Manejo de errores

SnackCheck está diseñado para responder de forma controlada ante entradas inválidas o datos incompletos.

| Situación | Comportamiento |
|---|---|
| Solicitud vacía `{}` | HTTP `200` — devuelve autodocumentación del servicio |
| Falta `barcode` | HTTP `400` — respuesta estructurada de validación |
| `barcode` contiene caracteres no numéricos | HTTP `400` — respuesta estructurada de validación |
| Producto no encontrado en Open Food Facts | HTTP `404` |
| Faltan azúcar, sal o grasa | HTTP `200` — respuesta de datos insuficientes; no se clasifica el producto |
| Nutri-Score ausente o `unknown` | HTTP `200` — se utiliza únicamente `nutrient_verdict` |

Las respuestas de validación incluyen información como:

- `status`
- `code`
- `message`
- `field`
- explicación de qué ocurrió;
- por qué ocurrió;
- cómo corregirlo;
- ejemplo de entrada válida.

---

## Configuración

### Requisitos

- n8n
- conexión a Internet
- credencial válida de Groq

Open Food Facts es una API abierta y no requiere:

- registro;
- API key;
- autenticación.

### Credenciales

La única credencial externa necesaria es Groq.

La clave debe configurarse dentro del sistema de credenciales de n8n y **no debe almacenarse directamente en el workflow exportado ni en el repositorio**.

---

## Inicio rápido

1. Importa el archivo:

```text
workflow/snackcheck-workflow.json
```

en n8n.

2. Configura una credencial válida de Groq y asígnala a los nodos de IA.

3. Comprueba que el webhook utilice la ruta:

```text
/nutrition-check
```

4. Publica o activa el workflow.

5. Envía una solicitud HTTP `POST` con un cuerpo como:

```json
{
  "barcode": "3017620422003"
}
```

---

## Uso

En una instalación local de n8n, el endpoint puede ser:

```text
http://localhost:5678/webhook/nutrition-check
```

> `localhost` corresponde únicamente a una ejecución local. En otra instalación debe sustituirse por la URL de la instancia de n8n utilizada.

Ejemplo de solicitud:

```json
{
  "barcode": "3017620422003"
}
```

Códigos utilizados durante el desarrollo:

```text
3017620422003 -> Nutella
5449000000996 -> Coca-Cola
```

Los productos adicionales utilizados en las pruebas se documentan en:

```text
tests/test-cases.md
```

---

## Documentación del workflow

El proyecto utiliza varias capas de documentación:

- nombres descriptivos siguiendo el patrón `[Acción] - [Propósito]`;
- notas inline en los nodos principales;
- comentarios en los nodos Code;
- diagrama del flujo en Excalidraw;
- este README;
- registro independiente de casos de prueba;
- CHANGELOG para registrar cambios de versión.

Las notas principales del workflow se incluyen tanto en español como en inglés.

---

## Pruebas

Los casos de prueba completos se documentan en:

```text
tests/test-cases.md
```

Las pruebas cubren, entre otros, los siguientes escenarios:

- código de barras válido;
- código de barras ausente;
- código de barras no numérico;
- producto no encontrado;
- producto con información nutricional incompleta;
- clasificación `healthy`;
- clasificación `moderate`;
- clasificación `unhealthy`;
- discrepancia entre lectura nutricional y Nutri-Score;
- producto sin Nutri-Score;
- generación del texto de IA;
- respuesta final del webhook;
- integración con Open Food Facts;
- tiempo de respuesta.

Después de cualquier corrección o modificación relevante, los casos afectados deben volver a ejecutarse.

---

## Limitaciones

- SnackCheck depende de la información disponible en Open Food Facts.
- Algunos productos pueden no estar registrados en la base de datos.
- Algunos productos pueden contener información nutricional incompleta.
- El sistema aplica exclusivamente las reglas nutricionales definidas para este proyecto.
- La explicación generada por IA depende de la disponibilidad y comportamiento del modelo de Groq.
- El veredicto es una simplificación basada en reglas y no sustituye asesoramiento médico o nutricional profesional.
- El workflow analiza un único producto por solicitud y no procesa colecciones ni lotes de productos.

---

## Mantenimiento

Si cambian los umbrales nutricionales, deben revisarse los nodos de clasificación de azúcar, sal y grasa.

Si cambia la estructura de Open Food Facts, deben revisarse:

- el nodo HTTP Request;
- la extracción de campos;
- las validaciones de existencia de datos.

Si se cambia el proveedor o modelo de IA, deben revisarse:

- las credenciales;
- los nodos Groq;
- los prompts;
- la estructura del campo de salida generado por el modelo.

Después de cualquier modificación importante debe volver a ejecutarse el conjunto de pruebas correspondiente.

---

## Estructura del proyecto

```text
SnackCheck/
├── README.md
├── README_EN.md
├── CHANGELOG.md
├── tests/
│   └── test-cases.md
├── diagrams/
│   └── snackcheck-workflow.excalidraw
└── workflow/
    └── snackcheck-workflow.json
```

---

## Autor

**Mónica Gómez Vadillo**

Proyecto final del prework de la formación de **AI Engineering de 4Geeks Academy**, desarrollado como parte del módulo de automatización con **n8n**.

El proyecto aplica conceptos de validación de entradas, integración de APIs, lógica condicional, generación de contenido con IA, manejo de errores, documentación y pruebas de workflows.
