# SnackCheck - Nutrition Verdict

**Language:** English | [Español](README.md)

Automation built in **n8n** that analyzes a packaged food product from its barcode, retrieves real product data from **Open Food Facts**, applies deterministic nutrition rules, and generates a final verdict accompanied by a short explanation using **Groq AI**.

---

## Purpose

SnackCheck is designed as the engine of a small consumer application that answers a simple question in a clear and friendly way: **is this product suitable for everyday consumption, or would it be better as an occasional choice?**

The workflow:

- receives a barcode;
- validates the input;
- queries the product in Open Food Facts;
- checks that the required nutrition data is available;
- classifies sugar, salt, and fat;
- calculates a concern score;
- interprets the Nutri-Score;
- combines both readings using the most restrictive result;
- generates an AI explanation adapted to the verdict level;
- returns an easy-to-read final response.

The three possible levels are:

- `healthy`
- `moderate`
- `unhealthy`

> AI does not decide the classification. The verdict is calculated beforehand using deterministic rules.

---

## Technologies Used

- **n8n** — workflow construction and orchestration.
- **Open Food Facts API v2** — retrieval of real product data.
- **Groq** — generation of the final explanatory text.
- **JavaScript** — input validation and `concern_score` calculation.
- **Excalidraw** — visual design and workflow documentation.

---

## Workflow Architecture

The project follows the five layers covered during the training.

### 1. Input Layer

The workflow starts with an HTTP `POST` webhook:

```text
/nutrition-check
```

The request must include a barcode in the JSON body:

```json
{
  "barcode": "3017620422003"
}
```

The input is validated before any external API call is made. The barcode must:

- exist;
- not be empty;
- contain digits only.

If validation fails, the workflow returns a structured JSON response with HTTP `400`.

A completely empty request body `{}` is handled separately as a self-documentation request and returns HTTP `200` with usage instructions.

---

### 2. Data Layer

Once the barcode has been validated, the workflow queries Open Food Facts:

```text
GET https://world.openfoodfacts.org/api/v2/product/{barcode}.json?fields=product_name,brands,nutriscore_grade,nutriments
```

The following fields are used:

- `product_name`
- `brands`
- `nutriscore_grade`
- `nutriments.sugars_100g`
- `nutriments.salt_100g`
- `nutriments.fat_100g`
- `nutriments['energy-kcal_100g']`

Bracket notation is used for the energy field because its name contains a hyphen.

Before continuing, the workflow checks:

1. that the product exists;
2. that sugar, salt, and fat values are available for classification.

If Open Food Facts does not find the product, the workflow returns HTTP `404`.

If the product exists but does not contain enough nutrition data, the workflow returns a controlled insufficient-data response and **does not generate a verdict**.

---

### 3. Logic Layer

#### Nutrient Classification

##### Sugar

Sugar per 100 g is classified as:

| Value | Classification |
|---|---|
| `< 5 g` | `low` |
| `5 - 22.5 g` | `medium` |
| `> 22.5 g` | `high` |

The boundary values `5` and `22.5` belong to the `medium` category.

##### Salt

| Value | Classification |
|---|---|
| `<= 1.5 g` | `not_high` |
| `> 1.5 g` | `high` |

##### Fat

| Value | Classification |
|---|---|
| `<= 17.5 g` | `not_high` |
| `> 17.5 g` | `high` |

---

#### Concern Score

The `concern_score` field indicates how many of the three nutrients are classified as `high`.

| Nutrients classified as `high` | `concern_score` |
|---:|---:|
| 0 | 0 |
| 1 | 1 |
| 2 | 2 |
| 3 | 3 |

The score is calculated in a **Code** node on a single item.

---

#### Nutrient-Based Verdict

The `concern_score` is interpreted as follows:

| `concern_score` | `nutrient_verdict` |
|---:|---|
| 0 | `healthy` |
| 1 | `moderate` |
| 2 or 3 | `unhealthy` |

---

#### Nutri-Score Classification

The `nutriscore_grade` field is transformed into a second independent verdict:

| Nutri-Score | `nutriscore_verdict` |
|---|---|
| `a` or `b` | `healthy` |
| `c` | `moderate` |
| `d` or `e` | `unhealthy` |

If `nutriscore_grade` is missing or has the value `unknown`, this second verdict is ignored and the final result relies only on `nutrient_verdict`.

---

#### Final Verdict

The workflow compares:

- `nutrient_verdict`
- `nutriscore_verdict`

and always uses the most restrictive result:

```text
unhealthy > moderate > healthy
```

Therefore:

- if either verdict is `unhealthy` → `final_verdict = unhealthy`;
- if neither is `unhealthy` but one is `moderate` → `final_verdict = moderate`;
- only if both are `healthy` → `final_verdict = healthy`;
- if Nutri-Score is unavailable → `final_verdict = nutrient_verdict`.

This separation prevents cases such as a soft drink with `concern_score = 0` but an unfavorable Nutri-Score from being incorrectly classified as healthy.

---

### 4. Intelligence Layer

Once `final_verdict` has been calculated, the workflow routes the product to one of three Groq prompts.

#### Healthy

Generates a message that is:

- positive;
- encouraging;
- brief;
- non-medical;
- non-alarmist.

#### Moderate

Generates a message that is:

- positive but balanced;
- brief;
- focused on the nutrition aspect that deserves attention.

#### Unhealthy

Generates a message that is:

- cautious;
- friendly;
- non-alarmist;
- non-judgmental;
- oriented toward presenting the product as more appropriate for occasional consumption.

The AI receives the real product data and the already calculated verdict as context, but **it does not modify or reinterpret the verdict**.

The prompts explicitly instruct the model to respect the calculated nutrient classifications, so a `medium` nutrient is not described as `high`, and a `not_high` nutrient is not described as high.

---

### 5. Output Layer

The three AI routes are merged again and a final plain-text message is built.

The message uses visual markers to make the result easier to interpret:

- 🟢 `healthy`, `low`, or `not_high`
- 🟡 `moderate` or `medium`
- 🔴 `unhealthy` or `high`

Example:

```text
🥗 🔴 SnackCheck Verdict: UNHEALTHY

Product: Coca-Cola
Brand: COCA-COLA SERVICES SA/NV

🟡 Sugar: 10.6 g/100g (medium)
🟢 Salt: 0 g/100g (not_high)
🟢 Fat: 0 g/100g (not_high)

Nutri-Score: e
Concern score: 0/3

[Short AI-generated explanation]
```

The final response uses:

```text
HTTP 200
Content-Type: text/plain; charset=utf-8
```

---

## Error Handling

SnackCheck is designed to respond in a controlled way to invalid input or incomplete data.

| Situation | Behavior |
|---|---|
| Empty request `{}` | HTTP `200` — returns service self-documentation |
| Missing `barcode` | HTTP `400` — structured validation response |
| `barcode` contains non-numeric characters | HTTP `400` — structured validation response |
| Product not found in Open Food Facts | HTTP `404` |
| Missing sugar, salt, or fat | HTTP `200` — insufficient-data response; product is not classified |
| Nutri-Score missing or `unknown` | HTTP `200` — only `nutrient_verdict` is used |

Validation responses include information such as:

- `status`
- `code`
- `message`
- `field`
- what happened;
- why it happened;
- how to fix it;
- an example of a valid request.

---

## Configuration

### Requirements

- n8n
- Internet connection
- valid Groq credentials

Open Food Facts is an open API and does not require:

- registration;
- API key;
- authentication.

### Credentials

The only external credential required is Groq.

The key must be configured through n8n's credential system and **must not be stored directly in the exported workflow or in the repository**.

---

## Quick Start

1. Import the file:

```text
workflow/snackcheck-workflow.json
```

into n8n.

2. Configure valid Groq credentials and assign them to the AI nodes.

3. Check that the webhook uses the path:

```text
/nutrition-check
```

4. Publish or activate the workflow.

5. Send an HTTP `POST` request with a body such as:

```json
{
  "barcode": "3017620422003"
}
```

---

## Usage

In a local n8n installation, the endpoint may be:

```text
http://localhost:5678/webhook/nutrition-check
```

> `localhost` only applies to a local setup. In another installation, replace it with the URL of the relevant n8n instance.

Example request:

```json
{
  "barcode": "3017620422003"
}
```

Barcodes used during development:

```text
3017620422003 -> Nutella
5449000000996 -> Coca-Cola
```

Additional products used in testing are documented in:

```text
tests/test-cases.md
```

---

## Workflow Documentation

The project uses several documentation layers:

- descriptive node names following the `[Action] - [Purpose]` pattern;
- inline notes in the main nodes;
- comments inside Code nodes;
- workflow diagram in Excalidraw;
- this README;
- separate test-case documentation;
- CHANGELOG for version history.

The main workflow notes are included in both Spanish and English.

---

## Testing

The complete test cases are documented in:

```text
tests/test-cases.md
```

The tests cover, among other scenarios:

- valid barcode;
- missing barcode;
- non-numeric barcode;
- product not found;
- product with incomplete nutrition data;
- `healthy` classification;
- `moderate` classification;
- `unhealthy` classification;
- disagreement between nutrient verdict and Nutri-Score;
- product without Nutri-Score;
- AI-generated explanation;
- final webhook response;
- Open Food Facts integration;
- response time.

After any relevant correction or modification, the affected tests should be executed again.

---

## Limitations

- SnackCheck depends on the information available in Open Food Facts.
- Some products may not be registered in the database.
- Some products may contain incomplete nutrition information.
- The system applies only the nutrition rules defined for this project.
- The AI-generated explanation depends on the availability and behavior of the Groq model.
- The verdict is a simplified rule-based assessment and does not replace professional medical or nutritional advice.
- The workflow analyzes one product per request and does not process collections or batches of products.

---

## Maintenance

If the nutrition thresholds change, the sugar, salt, and fat classification nodes must be reviewed.

If the Open Food Facts response structure changes, review:

- the HTTP Request node;
- field extraction;
- data-availability validations.

If the AI provider or model changes, review:

- credentials;
- Groq nodes;
- prompts;
- the structure of the model output field.

After any important modification, the relevant test suite should be run again.

---

## Project Structure

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

## Author

**Mónica Gómez Vadillo**

Final prework project for the **AI Engineering program at 4Geeks Academy**, developed as part of the **n8n automation module**.

The project applies concepts related to input validation, API integration, conditional logic, AI-powered content generation, error handling, documentation, and workflow testing.
