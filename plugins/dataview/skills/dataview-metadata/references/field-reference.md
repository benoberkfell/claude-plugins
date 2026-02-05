# Field Names and Data Types

## Field Naming Rules

Dataview normalizes all field names. Here's how:

| Input | Normalized | Notes |
|-------|-----------|-------|
| `Author` | `author` | Converted to lowercase |
| `book rating` | `book-rating` | Spaces become hyphens |
| `Book!` | `book` | Punctuation removed |
| `simple-field` | `simple-field` | Already normalized |
| `CAPS_FIELD` | `caps-field` | Caps + underscores normalized |

When referencing a field in queries, always use the normalized form:

```dataview
TABLE author, book-rating, published-date
```

## Supported Data Types

### Text
Raw strings. No special syntax needed.

```markdown
---
author: "Jane Smith"
genre: horror
---

Or inline: [notes:: This is a great book]
```

**In queries:** Treat as text for string operations.

### Number
Integers or decimals.

```markdown
---
rating: 4.5
page-count: 312
---

Or inline: [pages:: 450]
```

**In queries:** Use for arithmetic and comparisons.

### Date
Format: `YYYY-MM-DD`. Dataview auto-recognizes this format.

```markdown
---
published: 2024-03-15
started: 2024-01-10
---

Or inline: [finished:: 2024-03-20]
```

**In queries:** Use `date()` function to ensure proper comparison:
```dataview
WHERE completed >= date(2024-01-01)
```

### Duration
Time periods. Format: `dur(value unit)`.

```markdown
---
reading-time: dur(4 hours)
estimate: dur(2 days)
---

Or inline: [effort:: dur(5 hours)]
```

**Common units:** `ms`, `second`, `seconds`, `minute`, `minutes`, `hour`, `hours`, `day`, `days`, `week`, `weeks`, `month`, `months`, `year`, `years`

**In queries:** Add/subtract from dates:
```dataview
TABLE started, started + dur(1 month) AS "Checkpoint"
```

### Link
Reference to another note. Format: `[[Note Name]]` or `[[Note Name|Display Text]]`.

```markdown
---
related: [[Main Concept]]
---

Or inline: [mentor:: [[Jane Smith]]]
```

**In queries:** Access fields from the linked note:
```dataview
TABLE author, [[author]].role
```

### List
Multiple values. Use square brackets with comma separation.

```markdown
---
tags: [productivity, writing, self-improvement]
authors: [Stephen King, Neil Gaiman]
---

Or inline: [contributors:: [[Alice]], [[Bob]]]
```

**In queries:** Use `contains()` to check membership:
```dataview
WHERE contains(tags, "productivity")
```

### Object
Key-value pairs (advanced). Use `{key: value}` syntax.

```markdown
---
metadata:
  created: 2024-01-01
  version: 2
---
```

Access with dot notation:
```dataview
TABLE metadata.created, metadata.version
```

### Boolean
True/false values.

```markdown
---
published: true
archived: false
---

Or inline: [completed:: true]
```

**In queries:** Use in WHERE clauses:
```dataview
WHERE !archived AND published
```

## Type Conversion

Dataview is lenient with types in some contexts:

- **Text to number:** "5" becomes 5 in arithmetic
- **Date to text:** `date(2024-03-15)` displays as "March 15, 2024"
- **List to boolean:** Empty list is falsy, non-empty is truthy

For explicit conversion, use functions:
- `number(value)` - convert to number
- `string(value)` - convert to string
- `typeof(value)` - check type of value

## Null/Unset Fields

If a field is missing or empty, it's null. Handle with `default()`:

```dataview
TABLE author, default(completed, "not yet")
```

Or filter them out:

```dataview
WHERE completed != null
```

## Edge Cases

**Empty strings vs null:** `field: ""` is different from field being missing. Both are falsy but one is an empty string.

**Spaces in field names:** Use hyphens when inline:
```markdown
[book-rating:: 4.5]
```

But YAML handles spaces automatically:
```markdown
---
book rating: 4.5
---
```
Then refer to it as `book-rating` in queries.

**Special characters:** Avoid them. Dataview will strip most punctuation. Stick to letters, numbers, hyphens, underscores.
