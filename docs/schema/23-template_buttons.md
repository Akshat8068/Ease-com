# Entity: template_buttons

## Purpose
Har template language variant ke interactive buttons store karta hai. Meta WhatsApp templates mein maximum 3 buttons allowed hain. Teen types hote hain — quick reply (user ek tap mein reply karta hai), url (ek link khulta hai), phone (call hoti hai). Buttons template ke saath Meta ko submit hote hain aur approve hone ke baad hi send hote hain.

---

## Table Definition

```sql
CREATE TABLE template_buttons (
    id                   UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    template_language_id UUID            NOT NULL REFERENCES template_languages(id),
    button_index         INTEGER         NOT NULL,
    button_type          VARCHAR(20)     NOT NULL CHECK (button_type IN ('quick_reply', 'url', 'phone')),
    label                VARCHAR(255)    NOT NULL,
    value                VARCHAR(500)
);
```

---

## Columns

| Column               | Type         | Nullable | Default           | Description                                                                 |
|----------------------|--------------|----------|-------------------|-----------------------------------------------------------------------------|
| id                   | UUID         | NO       | gen_random_uuid() | Primary key                                                                 |
| template_language_id | UUID         | NO       | —                 | FK → template_languages. Kis language variant ke buttons hain               |
| button_index         | INTEGER      | NO       | —                 | Button order — 1, 2, 3. Controls display order on message                   |
| button_type          | VARCHAR(20)  | NO       | —                 | ENUM: quick_reply / url / phone                                             |
| label                | VARCHAR(255) | NO       | —                 | Button text shown to user. e.g. "Track Order" / "Call Us" / "Yes, confirm" |
| value                | VARCHAR(500) | YES      | NULL              | url: the link / phone: the number / quick_reply: the reply payload          |

---

## Button Types

| Type        | label example    | value example                        | What happens when user taps          |
|-------------|------------------|--------------------------------------|--------------------------------------|
| quick_reply | "Yes, confirm"   | "CONFIRM_ORDER"                      | Sends that text back as a reply      |
| quick_reply | "No, cancel"     | "CANCEL_ORDER"                       | Sends that text back as a reply      |
| url         | "Track Order"    | "https://track.delhivery.com/{{1}}"  | Opens the URL in browser             |
| url         | "View Cart"      | "https://shop.lavos.in/cart"         | Opens the URL in browser             |
| phone       | "Call Us"        | "+91 98110 00000"                    | Initiates a phone call               |

---

## value field by button_type

```
quick_reply:
→ value = the payload text sent back when user taps
→ e.g. "CONFIRM_ORDER" or "YES" or "NO"
→ This payload comes back as an inbound message
→ Our system reads it to trigger next journey step

url:
→ value = the full URL
→ Can contain variables e.g. "https://track.delhivery.com/{{1}}"
→ {{1}} replaced with AWB at send time

phone:
→ value = phone number with country code
→ e.g. "+91 98110 00000"
→ value is nullable for phone if label is enough
```

---

## Business Rules

- Maximum 3 buttons per template language variant (Meta limit)
- button_index must be 1, 2 or 3 — sequential
- label is always required
- value is nullable only for quick_reply if payload not needed
- value is required for url type
- value is required for phone type
- template_language_id + button_index must be UNIQUE
- Buttons are part of the template submitted to Meta
- Any change to button label or value = new Meta submission required

---

## Example — COD Confirmation Template Buttons

```
template: "COD confirmation"
body: "Hi {{1}}, please confirm your COD order {{2}} worth {{3}}."

button_index | button_type | label          | value
──────────────────────────────────────────────────────
1            | quick_reply | "Yes, confirm" | "CONFIRM_COD"
2            | quick_reply | "No, cancel"   | "CANCEL_COD"
```

---

## Example — Shipping Update Template Buttons

```
template: "Order status update"
body: "Hi {{1}}, your order {{2}} has been shipped via {{3}}. AWB: {{4}}"

button_index | button_type | label         | value
──────────────────────────────────────────────────────────────
1            | url         | "Track Order" | "https://track.shiprocket.in/{{4}}"
2            | phone       | "Call Us"     | "+91 98110 00000"
```

---

## Indexes

```sql
-- Language variant ke saare buttons fetch karne ke liye
CREATE INDEX idx_tpl_btns_language_id ON template_buttons(template_language_id);

-- Uniqueness: one row per button position per language variant
CREATE UNIQUE INDEX idx_tpl_btns_language_index ON template_buttons(template_language_id, button_index);
```

---

## Relationships

| Related Entity     | Type     | Via                                     |
|--------------------|----------|-----------------------------------------|
| template_languages | Many-One | template_buttons.template_language_id   |
