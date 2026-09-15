# Entity: template_variables

## Purpose
Har template language variant ke body mein jo variables hain ({{1}}, {{2}}...{{n}}) unka mapping store karta hai. Do kaam karta hai:
1. **sample_value** — Meta ko submit karte waqt example data deta hai (Meta review ke liye zaroori)
2. **data_key** — Runtime par client API se actual data fetch karne ka mapping (e.g. customer.name, order.order_number)

---

## Table Definition

```sql
CREATE TABLE template_variables (
    id                   UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    template_language_id UUID            NOT NULL REFERENCES template_languages(id),
    variable_index       INTEGER         NOT NULL,
    sample_value         VARCHAR(255)    NOT NULL,
    data_key             VARCHAR(255)
);
```

---

## Columns

| Column               | Type         | Nullable | Default           | Description                                                                 |
|----------------------|--------------|----------|-------------------|-----------------------------------------------------------------------------|
| id                   | UUID         | NO       | gen_random_uuid() | Primary key                                                                 |
| template_language_id | UUID         | NO       | —                 | FK → template_languages. Kis language variant ke variables hain             |
| variable_index       | INTEGER      | NO       | —                 | Variable number — 1, 2, 3...n. Matches {{1}} {{2}} {{n}} in body           |
| sample_value         | VARCHAR(255) | NO       | —                 | Sample value shown to Meta during review. e.g. "Rahul" for {{1}}, "1078812" for {{2}} |
| data_key             | VARCHAR(255) | YES      | NULL              | Client API field path that fills this variable at send time. e.g. "customer.name" / "order.order_number" / "order.awb" |

---

## Why Two Fields — sample_value vs data_key

```
sample_value:
→ Used ONLY during Meta submission
→ Meta needs to see example data to review the template
→ e.g. body = "Hi {{1}}, your order {{2}} is confirmed"
→ sample for {{1}} = "Rahul"
→ sample for {{2}} = "1078812"
→ Meta sees: "Hi Rahul, your order 1078812 is confirmed"
→ After approval — sample_value is never used again

data_key:
→ Used at SEND TIME (runtime)
→ Maps variable to actual client API data
→ e.g. data_key = "customer.name"
→ At send time: fetch customer.name from event payload
→ Replace {{1}} with actual customer name
→ e.g. "Hi Priya, your order 1079001 is confirmed"
```

---

## Example — Order Confirmation Template

```
body = "Hi {{1}}, your order {{2}} worth {{3}} has been confirmed. 
        Expected delivery: {{4}}."

variable_index | sample_value        | data_key
───────────────────────────────────────────────────────
1              | "Rahul"             | "customer.name"
2              | "1078812"           | "order.order_number"
3              | "₹2,497"            | "order.total"
4              | "30 Aug 2026"       | "order.expected_delivery"
```

---

## Example — Cart Reminder Template

```
body = "Hi {{1}}, you left {{2}} in your cart. 
        Complete your order and get {{3}} off."

variable_index | sample_value        | data_key
───────────────────────────────────────────────────────
1              | "Priya"             | "customer.name"
2              | "Vitamin C Serum"   | "cart.item_name"
3              | "10%"               | "journey_step.offer_value"
```

---

## data_key Sources

```
Data comes from three sources at send time:

1. Client API event payload
   e.g. "customer.name", "order.order_number",
        "order.awb", "order.total", "order.courier"

2. Journey step config
   e.g. "journey_step.offer_value"
        (agent set this in journey editor)

3. Our system data
   e.g. "ticket.id", "agent.name"
```

---

## Business Rules

- One row per variable per template_language
- variable_index must be sequential starting from 1
- variable_count in template_languages must match max variable_index here
- sample_value is required — Meta rejects submission without samples
- data_key is nullable — if null, variable must be filled manually at send time
- template_language_id + variable_index combination must be UNIQUE

---

## Indexes

```sql
-- Language variant ke saare variables fetch karne ke liye
CREATE INDEX idx_tpl_vars_language_id ON template_variables(template_language_id);

-- Uniqueness: one row per variable per language variant
CREATE UNIQUE INDEX idx_tpl_vars_language_index ON template_variables(template_language_id, variable_index);
```

---

## Relationships

| Related Entity     | Type     | Via                                      |
|--------------------|----------|------------------------------------------|
| template_languages | Many-One | template_variables.template_language_id  |
