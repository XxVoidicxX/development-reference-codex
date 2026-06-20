# Discord Components V2 — Complete Reference

**Source:** https://docs.discord.com/developers/components/reference  
**Last Verified:** June 2026  
**Purpose:** AI coding assistant context file (Copilot, Claude, etc.) and developer reference. Prevents hallucination by documenting exact types, flags, nesting rules, field constraints, and gotchas.

---

## Activation Flag

To use Components V2, set `flags: 1 << 15` (`32768`) on the message payload.

```json
{
  "flags": 32768,
  "components": [...]
}
```

**Critical rules when IS_COMPONENTS_V2 is set:**
- `content` and `embeds` fields are **disabled** — use `TextDisplay` and `Container` instead
- `poll` and `stickers` fields are **disabled**
- Attachments are hidden by default — must be referenced explicitly via `Thumbnail`, `MediaGallery`, or `File` components
- Max **40 total components** per message (nested components count toward this limit)
- Max **4000 characters** across all `TextDisplay` components
- Once sent with this flag, **it cannot be removed** from that message
- Editing a message into V2: explicitly set `content`, `poll`, `embeds`, `stickers` to `null`

---

## Component Types — Full Table

| Type | Name              | Category    | Available In     |
|------|-------------------|-------------|------------------|
| 1    | Action Row        | Layout      | Message          |
| 2    | Button            | Interactive | Message          |
| 3    | String Select     | Interactive | Message, Modal   |
| 4    | Text Input        | Interactive | Modal            |
| 5    | User Select       | Interactive | Message, Modal   |
| 6    | Role Select       | Interactive | Message, Modal   |
| 7    | Mentionable Select| Interactive | Message, Modal   |
| 8    | Channel Select    | Interactive | Message, Modal   |
| 9    | Section           | Layout      | Message          |
| 10   | Text Display      | Content     | Message, Modal   |
| 11   | Thumbnail         | Content     | Message          |
| 12   | Media Gallery     | Content     | Message          |
| 13   | File              | Content     | Message          |
| 14   | Separator         | Layout      | Message          |
| 17   | Container         | Layout      | Message          |
| 18   | Label             | Layout      | Modal            |
| 19   | File Upload       | Interactive | Modal            |
| 21   | Radio Group       | Interactive | Modal            |
| 22   | Checkbox Group    | Interactive | Modal            |
| 23   | Checkbox          | Interactive | Modal            |

> **Types 15, 16, and 20 do not exist.** Never generate them.

---

## Base Component Fields

Every component shares these two fields:

| Field | Type    | Required | Description                                                        |
|-------|---------|----------|--------------------------------------------------------------------|
| type  | integer | Yes      | Component type integer (see table above)                           |
| id?   | integer | No       | 32-bit integer identifier. Auto-assigned if omitted. `0` = treated as empty. Must be unique per message. |

---

## Nesting Rules — Message

```
Message
└── [top-level, up to 10]
    ├── Container (type 17)
    │   └── [children]
    │       ├── Section (type 9)
    │       │   ├── components[]: TextDisplay (type 10), up to 3
    │       │   └── accessory?: Thumbnail (type 11) | Button (type 2)
    │       ├── TextDisplay (type 10)
    │       ├── MediaGallery (type 12)
    │       ├── File (type 13)
    │       ├── Separator (type 14)
    │       └── ActionRow (type 1)
    │           └── components[]
    │               ├── Button ×5 (type 2)
    │               └── OR one Select (type 3/5/6/7/8)
    ├── Section (type 9)
    ├── TextDisplay (type 10)
    ├── MediaGallery (type 12)
    ├── File (type 13)
    ├── Separator (type 14)
    └── ActionRow (type 1)
```

**Forbidden nesting:**
- Container inside Container
- Section inside Section
- Button outside ActionRow or Section accessory
- TextDisplay as Section accessory (only Thumbnail or Button)
- Any modal-only component (Label, TextInput, RadioGroup, etc.) in a message

---

## Nesting Rules — Modal

```
Modal
└── components[] [top-level]
    └── Label (type 18) [wraps one interactive component]
        └── component: (one of)
            ├── TextInput (type 4)
            ├── StringSelect (type 3)
            ├── UserSelect (type 5)
            ├── RoleSelect (type 6)
            ├── MentionableSelect (type 7)
            ├── ChannelSelect (type 8)
            ├── FileUpload (type 19)
            ├── RadioGroup (type 21)
            ├── CheckboxGroup (type 22)
            └── Checkbox (type 23)
```

> Action Row with Text Inputs in modals is **deprecated**. Use Label instead.

---

## Component Reference

---

### Action Row (type: 1)

| Field      | Type      | Required | Description                                              |
|------------|-----------|----------|----------------------------------------------------------|
| type       | integer   | Yes      | `1`                                                      |
| id?        | integer   | No       | Optional identifier                                      |
| components | array     | Yes      | Up to 5 Buttons OR one Select (string/user/role/mentionable/channel) |

---

### Button (type: 2)

| Field      | Type    | Required        | Description                                      |
|------------|---------|-----------------|--------------------------------------------------|
| type       | integer | Yes             | `2`                                              |
| id?        | integer | No              | Optional identifier                              |
| style      | integer | Yes             | See styles table                                 |
| label?     | string  | Conditional     | Max 80 chars. Required unless emoji is set.      |
| emoji?     | object  | No              | `{ name, id, animated }`                         |
| custom_id? | string  | Conditional     | 1–100 chars. Required for styles 1–4. Not for 5/6. |
| sku_id?    | snowflake | Conditional  | Required for style 6 (Premium). Not for others.  |
| url?       | string  | Conditional     | Max 512 chars. Required for style 5 (Link).      |
| disabled?  | boolean | No              | Defaults to `false`                              |

**Button Styles:**

| Name      | Value | Required Field | Sends Interaction |
|-----------|-------|----------------|-------------------|
| Primary   | 1     | `custom_id`    | Yes               |
| Secondary | 2     | `custom_id`    | Yes               |
| Success   | 3     | `custom_id`    | Yes               |
| Danger    | 4     | `custom_id`    | Yes               |
| Link      | 5     | `url`          | No                |
| Premium   | 6     | `sku_id`       | No                |

**Placement:** Inside ActionRow `components[]` OR Section `accessory`.

---

### String Select (type: 3)

| Field        | Type    | Required | Description                                              |
|--------------|---------|----------|----------------------------------------------------------|
| type         | integer | Yes      | `3`                                                      |
| id?          | integer | No       | Optional identifier                                      |
| custom_id    | string  | Yes      | 1–100 chars                                              |
| options      | array   | Yes      | Max 25 options (see Select Option below)                 |
| placeholder? | string  | No       | Max 150 chars                                            |
| min_values?  | integer | No       | Default 1. Min 0. Max 25. Must be ≥1 if `required` is true (modal). |
| max_values?  | integer | No       | Default 1. Max 25.                                       |
| required?    | boolean | No       | Modal only. Defaults to `true`. Ignored in messages.     |
| disabled?    | boolean | No       | Message only. Defaults to `false`. Error if used in modal. |

**Select Option fields:**

| Field        | Type    | Required | Description              |
|--------------|---------|----------|--------------------------|
| label        | string  | Yes      | Max 100 chars            |
| value        | string  | Yes      | Max 100 chars            |
| description? | string  | No       | Max 100 chars            |
| emoji?       | object  | No       | `{ id, name, animated }` |
| default?     | boolean | No       | Shows as pre-selected    |

---

### Text Input (type: 4) — Modal Only

| Field        | Type    | Required | Description                             |
|--------------|---------|----------|-----------------------------------------|
| type         | integer | Yes      | `4`                                     |
| id?          | integer | No       | Optional identifier                     |
| custom_id    | string  | Yes      | 1–100 chars                             |
| style        | integer | Yes      | `1` = Short (single-line), `2` = Paragraph (multi-line) |
| min_length?  | integer | No       | Min 0, max 4000                         |
| max_length?  | integer | No       | Min 1, max 4000                         |
| required?    | boolean | No       | Defaults to `true`                      |
| value?       | string  | No       | Pre-filled value. Max 4000 chars.       |
| placeholder? | string  | No       | Max 100 chars                           |

> The `label` field on Text Input is **deprecated**. Use the `Label` component wrapper instead.

---

### User Select (type: 5) / Role Select (type: 6) / Mentionable Select (type: 7) / Channel Select (type: 8)

All four share this structure:

| Field              | Type    | Required | Description                                                 |
|--------------------|---------|----------|-------------------------------------------------------------|
| type               | integer | Yes      | 5, 6, 7, or 8                                               |
| id?                | integer | No       | Optional identifier                                         |
| custom_id          | string  | Yes      | 1–100 chars                                                 |
| placeholder?       | string  | No       | Max 150 chars                                               |
| default_values?    | array   | No       | Array of `{ id, type }` pre-selected values                 |
| min_values?        | integer | No       | Defaults to 1                                               |
| max_values?        | integer | No       | Defaults to 1                                               |
| disabled?          | boolean | No       | Message only. Defaults to `false`.                          |
| channel_types?     | array   | No       | **Channel Select only.** Filter by channel type integers.   |

---

### Section (type: 9)

| Field      | Type    | Required | Description                                                           |
|------------|---------|----------|-----------------------------------------------------------------------|
| type       | integer | Yes      | `9`                                                                   |
| id?        | integer | No       | Optional identifier                                                   |
| components | array   | Yes      | 1–3 `TextDisplay` components                                          |
| accessory? | object  | No       | A `Thumbnail` (type 11) or `Button` (type 2). Displayed to the right. |

---

### Text Display (type: 10)

| Field   | Type    | Required | Description                              |
|---------|---------|----------|------------------------------------------|
| type    | integer | Yes      | `10`                                     |
| id?     | integer | No       | Optional identifier                      |
| content | string  | Yes      | Markdown text. Contributes to 4000-char total. |

Supports standard Discord markdown: bold, italic, underline, strikethrough, code blocks, headers, lists, links, mentions.

---

### Thumbnail (type: 11)

| Field       | Type    | Required | Description                                                     |
|-------------|---------|----------|-----------------------------------------------------------------|
| type        | integer | Yes      | `11`                                                            |
| id?         | integer | No       | Optional identifier                                             |
| media       | object  | Yes      | `{ url: string }` — attachment URL (`attachment://filename`) or external URL |
| description?| string  | No       | Alt text                                                        |
| spoiler?    | boolean | No       | Defaults to `false`                                             |

**Placement:** Section `accessory` field only.

---

### Media Gallery (type: 12)

| Field | Type  | Required | Description                           |
|-------|-------|----------|---------------------------------------|
| type  | integer | Yes    | `12`                                  |
| id?   | integer | No     | Optional identifier                   |
| items | array | Yes      | 1–10 media items (see below)          |

**Media Gallery Item fields:**

| Field        | Type    | Required | Description                             |
|--------------|---------|----------|-----------------------------------------|
| media        | object  | Yes      | `{ url: string }`                       |
| description? | string  | No       | Alt text. Max 1024 chars.               |
| spoiler?     | boolean | No       | Defaults to `false`                     |

---

### File (type: 13)

| Field    | Type    | Required | Description                                   |
|----------|---------|----------|-----------------------------------------------|
| type     | integer | Yes      | `13`                                          |
| id?      | integer | No       | Optional identifier                           |
| file     | object  | Yes      | `{ url: "attachment://filename" }`            |
| spoiler? | boolean | No       | Defaults to `false`                           |

---

### Separator (type: 14)

| Field    | Type    | Required | Description                                      |
|----------|---------|----------|--------------------------------------------------|
| type     | integer | Yes      | `14`                                             |
| id?      | integer | No       | Optional identifier                              |
| divider? | boolean | No       | Whether to show a visible line. Defaults to `true`. |
| spacing? | integer | No       | `1` = small, `2` = large. Defaults to `1`.       |

---

### Container (type: 17)

| Field         | Type    | Required | Description                                       |
|---------------|---------|----------|---------------------------------------------------|
| type          | integer | Yes      | `17`                                              |
| id?           | integer | No       | Optional identifier. Required if using component replacement. |
| components    | array   | Yes      | Child components (see nesting rules above)        |
| accent_color? | integer | No       | RGB hex integer (e.g., `0x7289DA`). Left border color accent. |
| spoiler?      | boolean | No       | Defaults to `false`                               |

---

### Label (type: 18) — Modal Only

| Field       | Type    | Required | Description                                         |
|-------------|---------|----------|-----------------------------------------------------|
| type        | integer | Yes      | `18`                                                |
| id?         | integer | No       | Optional identifier                                 |
| label       | string  | Yes      | Display label for the component                     |
| description?| string  | No       | Additional helper text shown under the label        |
| component   | object  | Yes      | One interactive component (TextInput, Select, RadioGroup, CheckboxGroup, Checkbox, FileUpload) |

---

### File Upload (type: 19) — Modal Only

| Field     | Type    | Required | Description                      |
|-----------|---------|----------|----------------------------------|
| type      | integer | Yes      | `19`                             |
| id?       | integer | No       | Optional identifier              |
| custom_id | string  | Yes      | 1–100 chars                      |
| required? | boolean | No       | Defaults to `true`               |

---

### Radio Group (type: 21) — Modal Only

| Field   | Type  | Required | Description                  |
|---------|-------|----------|------------------------------|
| type    | integer | Yes    | `21`                         |
| id?     | integer | No     | Optional identifier          |
| options | array | Yes      | Array of radio options       |

---

### Checkbox Group (type: 22) — Modal Only

| Field   | Type  | Required | Description                  |
|---------|-------|----------|------------------------------|
| type    | integer | Yes    | `22`                         |
| id?     | integer | No     | Optional identifier          |
| items   | array | Yes      | Array of Checkbox components |

---

### Checkbox (type: 23) — Modal Only

| Field     | Type    | Required | Description                          |
|-----------|---------|----------|--------------------------------------|
| type      | integer | Yes      | `23`                                 |
| id?       | integer | No       | Optional identifier                  |
| custom_id | string  | Yes      | 1–100 chars                          |
| label     | string  | Yes      | Display label                        |
| checked?  | boolean | No       | Default checked state                |
| required? | boolean | No       | Defaults to `false`                  |

---

## discord.js V2 Builders

```js
import {
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  StringSelectMenuBuilder,
  StringSelectMenuOptionBuilder,
  TextDisplayBuilder,
  SectionBuilder,
  ContainerBuilder,
  SeparatorBuilder,
  MediaGalleryBuilder,
  ThumbnailBuilder,
  FileBuilder,
  MessageFlags,
} from 'discord.js'

await interaction.reply({
  flags: MessageFlags.IsComponentsV2,
  components: [
    new ContainerBuilder()
      .addTextDisplayComponents(
        new TextDisplayBuilder().setContent('Hello World')
      )
      .addActionRowComponents(
        new ActionRowBuilder().addComponents(
          new ButtonBuilder()
            .setCustomId('confirm')
            .setLabel('Confirm')
            .setStyle(ButtonStyle.Primary)
        )
      )
  ]
})
```

**Raw payload equivalent for REST calls:**

```json
{
  "flags": 32768,
  "components": [
    {
      "type": 17,
      "components": [
        { "type": 10, "content": "Hello World" },
        {
          "type": 1,
          "components": [
            { "type": 2, "custom_id": "confirm", "label": "Confirm", "style": 1 }
          ]
        }
      ]
    }
  ]
}
```

---

## Component Replacement

Containers with an explicit `id` can be replaced in-place without resending the full message. Send a PATCH to the message endpoint targeting the container by `id`.

---

## Deferred Interaction Gotcha

When deferring an interaction and then following up with V2 components, set the `flags: 32768` on the **followup PATCH payload**, not the initial defer.

```json
// PATCH /webhooks/{app_id}/{token}/messages/@original
{
  "flags": 32768,
  "components": [...]
}
```

Setting the flag only on the defer POST does not carry through to the followup.

---

## Common Mistakes / Hallucination Traps

1. **Types 15, 16, 20 do not exist.** Never use them.
2. **Buttons cannot be top-level.** They must be inside ActionRow or Section `accessory`.
3. **Section accessory cannot be TextDisplay.** Only Thumbnail or Button.
4. **Container inside Container is invalid.**
5. **`content` and `embeds` silently fail** when `IS_COMPONENTS_V2` is set. Use TextDisplay instead.
6. **Attachments must be referenced in a component** or they won't show.
7. **`disabled: true` on a Select inside a modal causes an API error.**
8. **`required` on StringSelect is modal-only.** Ignored in messages.
9. **Link buttons (style 5) do not trigger interactions.** Don't listen for their custom_id.
10. **Premium buttons (style 6) do not trigger interactions.** Use sku_id, not custom_id.
11. **Text Input `label` field is deprecated.** Use the Label (type 18) wrapper.
12. **Action Row with TextInput in modals is deprecated.** Use Label.
13. **`id: 0` is treated as empty** and will be replaced by auto-assignment.
14. **40 total component limit counts nested components.** A Container with 5 children counts as 6.
15. **4000 char limit is across ALL TextDisplay components combined**, not per-component.
16. **Editing a message into IS_COMPONENTS_V2** requires explicitly nulling out `content`, `embeds`, `poll`, `stickers`.
17. **Once IS_COMPONENTS_V2 is set on a message, it cannot be removed.**
18. **Section must have 1–3 TextDisplay children.** Cannot be empty or exceed 3.

---

## Full Message Example

```json
{
  "flags": 32768,
  "components": [
    {
      "type": 17,
      "id": 1,
      "accent_color": 7559394,
      "components": [
        {
          "type": 9,
          "components": [
            { "type": 10, "content": "**Action Required**" },
            { "type": 10, "content": "A moderation event was detected in your server." }
          ],
          "accessory": {
            "type": 11,
            "media": { "url": "attachment://icon.png" }
          }
        },
        { "type": 14, "divider": true, "spacing": 1 },
        {
          "type": 1,
          "components": [
            { "type": 2, "custom_id": "action_approve", "label": "Approve", "style": 3 },
            { "type": 2, "custom_id": "action_deny", "label": "Deny", "style": 4 },
            { "type": 2, "label": "View Docs", "style": 5, "url": "https://example.com" }
          ]
        }
      ]
    }
  ]
}
```

---

## Full Modal Example

```json
{
  "type": 9,
  "data": {
    "custom_id": "report_modal",
    "title": "Submit Report",
    "components": [
      {
        "type": 18,
        "id": 1,
        "label": "Reason",
        "description": "Briefly describe the issue",
        "component": {
          "type": 4,
          "id": 2,
          "custom_id": "reason",
          "style": 2,
          "placeholder": "Describe what happened...",
          "min_length": 20,
          "max_length": 1000,
          "required": true
        }
      },
      {
        "type": 18,
        "id": 3,
        "label": "Severity",
        "component": {
          "type": 3,
          "id": 4,
          "custom_id": "severity",
          "placeholder": "Select severity",
          "options": [
            { "label": "Low", "value": "low" },
            { "label": "Medium", "value": "medium" },
            { "label": "High", "value": "high", "emoji": { "name": "🚨" } }
          ]
        }
      }
    ]
  }
}
```
