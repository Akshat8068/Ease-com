# Entity: template_languages

## Purpose
Har template ke multiple language variants store karta hai. Ek template ka English, Hindi, Telugu, Marathi — alag alag variant ho sakta hai. Har language variant Meta se ALAG review hota hai. English approve hone ka matlab Hindi approve nahi — dono alag submit hote hain aur alag status rakhte hain. Journey sirf us language mein bhejta hai jo approved ho.

---

## Table Definition

```sql
CREATE TABLE template_languages (
    id               UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id      UUID            NOT NULL REFERENCES templates(id),
    language         VARCHAR(10)     NOT NULL,

    -- HEADER (optional)
    header_type      VARCHAR(20)     CHECK (header_type IN ('none', 'text', 'image', 'video', 'document')),
    header_text      VARCHAR(255),
    header_media_url VARCHAR(500),

    -- BODY (required)
    body             TEXT            NOT NULL,
    variable_count   INTEGER         NOT NULL DEFAULT 0,

    -- FOOTER (optional)
    footer           VARCHAR(255),

    -- META SUBMISSION STATUS
    status           VARCHAR(20)     NOT NULL DEFAULT 'draft'
                                     CHECK (status IN ('draft', 'submitted', 'approved', 'rejected', 'paused')),
    rejection_reason TEXT,
    submitted_at     TIMESTAMP,
    approved_at      TIMESTAMP,
    meta_template_id VARCHAR(255),

    created_at       TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMP       NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column           | Type         | Nullable | Default           | Description                                                                 |
|------------------|--------------|----------|-------------------|-----------------------------------------------------------------------------|
| id               | UUID         | NO       | gen_random_uuid() | Primary key                                                                 |
| template_id      | UUID         | NO       | —                 | FK → templates. Kis template ka variant hai                                 |
| language         | VARCHAR(10)  | NO       | —                 | ISO 639-1 language code. e.g. "en" / "hi" / "te" / "mr"                   |

### Header (optional)

| Column           | Type         | Nullable | Default | Description                                                                 |
|------------------|--------------|----------|---------|-----------------------------------------------------------------------------|
| header_type      | VARCHAR(20)  | YES      | NULL    | ENUM: none / text / image / video / document                                |
| header_text      | VARCHAR(255) | YES      | NULL    | Header text — only when header_type = text                                  |
| header_media_url | VARCHAR(500) | YES      | NULL    | Media URL — only when header_type = image / video / document                |

### Body (required)

| Column           | Type         | Nullable | Default | Description                                                                 |
|------------------|--------------|----------|---------|-----------------------------------------------------------------------------|
| body             | TEXT         | NO       | —       | Message body with variables e.g. "Hi {{1}}, your order {{2}} is confirmed"  |
| variable_count   | INTEGER      | NO       | 0       | How many variables in body e.g. 2 for {{1}} and {{2}}                       |

### Footer (optional)

| Column           | Type         | Nullable | Default | Description                                                                 |
|------------------|--------------|----------|---------|-----------------------------------------------------------------------------|
| footer           | VARCHAR(255) | YES      | NULL    | Small text at bottom of message                                             |

### Meta Submission Status

| Column           | Type         | Nullable | Default | Description                                                                 |
|------------------|--------------|----------|---------|-----------------------------------------------------------------------------|
| status           | VARCHAR(20)  | NO       | draft   | ENUM: draft / submitted / approved / rejected / paused. Per language, Meta reviews separately |
| rejection_reason | TEXT         | YES      | NULL    | Meta's reason if rejected                                                   |
| submitted_at     | TIMESTAMP    | YES      | NULL    | When submitted to Meta for review                                           |
| approved_at      | TIMESTAMP    | YES      | NULL    | When Meta approved this language variant                                    |
| meta_template_id | VARCHAR(255) | YES      | NULL    | Meta's own ID returned after submission                                     |
| created_at       | TIMESTAMP    | NO       | NOW()   | Record creation time                                                        |
| updated_at       | TIMESTAMP    | NO       | NOW()   | Last update time                                                            |

---

## Status Flow

```
draft
  ↓ agent submits to Meta
submitted
  ↓ Meta reviews
  ↓
approved  → can be used in journey steps and chat window
rejected  → cannot be used, agent must fix and resubmit
paused    → Meta paused due to quality issues, cannot send
```

---

## Per Language Review — Key Rule

```
Each language variant is reviewed by Meta SEPARATELY.

Example:
template: "cart_reminder"
→ English  → submitted → approved   ✓ can send
→ Hindi    → submitted → pending    ✗ cannot send yet
→ Telugu   → draft     → not submitted ✗ cannot send

Journey checks language status before sending:
→ customer language = Hindi
→ Hindi status = submitted (not approved)
→ fallback to English (approved)
→ send English version
```

---

## Business Rules

- One row per language per template
- template_id + language combination must be UNIQUE
- body is always required
- header, footer, buttons are optional
- variable_count must match actual {{n}} count in body
- status starts as 'draft' — agent must explicitly submit to Meta
- Only 'approved' status allows sending in journeys and chat
- meta_template_id stored after Meta confirms submission
- rejection_reason populated from Meta webhook response

---

## Indexes

```sql
-- Template ke saare language variants fetch karne ke liye
CREATE INDEX idx_tpl_lang_template_id ON template_languages(template_id);

-- Uniqueness: one variant per language per template
CREATE UNIQUE INDEX idx_tpl_lang_template_language ON template_languages(template_id, language);

-- Journey engine: fetch approved variants quickly
CREATE INDEX idx_tpl_lang_status ON template_languages(template_id, status);
```

---

## Relationships

| Related Entity     | Type     | Via                                  |
|--------------------|----------|--------------------------------------|
| templates          | Many-One | template_languages.template_id       |
| template_variables | One-Many | template_variables.template_language_id |
| template_buttons   | One-Many | template_buttons.template_language_id   |
| template_sends     | One-Many | template_sends.template_language_id     |
