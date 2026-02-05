---
name: dataview-javascript
description: Write DataviewJS code blocks for custom logic that DQL can't handle. Use this skill when you need to manipulate data with JavaScript, create custom renderings, perform complex transformations, or build interactive elements beyond what query language provides.
---

# DataviewJS - JavaScript API

When DQL isn't enough, use JavaScript. DataviewJS lets you run arbitrary code against the Dataview index with access to the full `dv` API.

## When to Use JavaScript vs DQL

**Use DQL when:**
- Querying and filtering data
- Displaying in standard format (list, table, task, calendar)
- Simple calculations

**Use JavaScript when:**
- Complex multi-step transformations
- Custom rendering (charts, cards, animations)
- Conditional logic beyond WHERE clauses
- Combining data from multiple sources
- Building interactive components

## DataviewJS Block Syntax

Code goes in a `dataviewjs` block (not `dataview`):

````markdown
```dataviewjs
// Your code here
// Always start with `dv` object
dv.paragraph("Hello!");
```
````

## The `dv` Object - Core Methods

### dv.pages(source)

Get all pages matching a source (tag, folder, file name pattern).

```javascript
// All pages with #book tag
let books = dv.pages("#book");

// All pages in "Projects" folder
let projects = dv.pages('"Projects"');

// Multiple sources
let work = dv.pages('#work OR "Active Projects"');

// Loop through results
for (let page of books) {
  dv.paragraph(page.file.link);
}
```

Returns an array of page objects. Each page has fields like `rating`, `author`, plus implicit fields like `file.link`, `file.name`, `file.path`, `file.folder`.

### dv.table(headers, rows)

Render a table programmatically.

```javascript
let books = dv.pages("#book").where(p => p.status = "completed");

// Map to table format: [file link, author, rating]
let tableData = books.map(b => [b.file.link, b.author, b.rating]);

dv.table(["Book", "Author", "Rating"], tableData);
```

### dv.list(array)

Render a bullet list.

```javascript
let books = dv.pages("#book");
let titles = books.map(b => b.file.link);

dv.list(titles);
```

### dv.paragraph(text)

Render text/markdown.

```javascript
dv.paragraph("**Bold text** and *italics*");
```

### dv.header(level, text)

Add a heading.

```javascript
dv.header(1, "My Title");
dv.header(2, "Subtitle");
```

## Data Manipulation

### .where() - Filter

```javascript
// Books with rating >= 4
let great = dv.pages("#book").where(b => b.rating >= 4);

// Multiple conditions
let active = dv.pages('"Projects"')
  .where(p => p.status = "active" && p.priority = "high");
```

### .sort() - Sort

```javascript
// Sort by rating (descending)
let sorted = books.sort(b => b.rating, true);

// Sort by date ascending
let chronological = books.sort(b => b.started, false);
```

### .map() - Transform

```javascript
// Extract just titles
let titles = books.map(b => b.file.name);

// Create new objects
let summary = books.map(b => ({
  title: b.file.link,
  rating: b.rating,
  status: b.status
}));
```

### .group() - Group by field

```javascript
// Group by author
let byAuthor = books.groupBy(b => b.author);

// Render each group
for (let [author, pages] of byAuthor) {
  dv.header(2, author);
  dv.list(pages.map(p => p.file.link));
}
```

### .length - Count

```javascript
let count = books.length;
dv.paragraph(`Total books: ${count}`);
```

## Common Patterns

See `references/dataviewjs-examples.md` for complete working examples:
- Counting and statistics
- Conditional rendering
- Building charts
- Custom formatting
- Date calculations
- Working with arrays/objects

## Important Notes

- **Immutability:** Dataview data is read-only. You can manipulate it in JavaScript, but changes won't save back to your notes.
- **Async:** The `dv` API is synchronous. No promises or async/await.
- **Access fields:** Use dot notation: `page.author`, `page.file.link`, `page.file.name`
- **Null handling:** Check for undefined fields: `if (page.rating) { ... }`

## Quick Debugging

```javascript
// Log to browser console
console.log(books);

// Display in note as plain text
dv.paragraph(JSON.stringify(books[0], null, 2));
```

## References

See `references/dataviewjs-examples.md` for real-world code examples by use case.
