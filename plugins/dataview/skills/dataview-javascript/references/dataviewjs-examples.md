# DataviewJS Code Examples

Real-world patterns for common use cases.

## Basic: Display and Filter

### "Show all high-rated books"
```javascript
let books = dv.pages("#book")
  .where(b => b.rating >= 4)
  .sort(b => b.rating, true);

dv.header(1, "Books I Love");
dv.table(
  ["Book", "Author", "Rating"],
  books.map(b => [b.file.link, b.author, b.rating])
);
```

### "Count tasks by priority"
```javascript
let tasks = dv.pages()
  .where(p => p.file.tasks && p.file.tasks.length > 0);

let byPriority = {};
for (let page of tasks) {
  for (let task of page.file.tasks) {
    let priority = task.priority || "none";
    byPriority[priority] = (byPriority[priority] || 0) + 1;
  }
}

dv.paragraph("**Tasks by Priority**");
for (let [priority, count] of Object.entries(byPriority)) {
  dv.paragraph(`${priority}: ${count}`);
}
```

---

## Grouping and Aggregation

### "Books grouped by author with count"
```javascript
let books = dv.pages("#book");
let byAuthor = books.groupBy(b => b.author);

let results = [];
for (let [author, pages] of byAuthor) {
  results.push([author, pages.length, pages.map(p => p.file.link)]);
}

dv.table(
  ["Author", "Books", "Titles"],
  results.sort((a, b) => b[1] - a[1])
);
```

### "Calculate average rating per genre"
```javascript
let books = dv.pages("#book").where(b => b.genre && b.rating);
let byGenre = books.groupBy(b => b.genre);

for (let [genre, pages] of byGenre) {
  let avgRating = pages.map(p => p.rating)
    .reduce((sum, r) => sum + r, 0) / pages.length;

  dv.paragraph(`${genre}: ${avgRating.toFixed(1)} ⭐`);
}
```

---

## Conditional Rendering

### "Show different content based on current date"
```javascript
let today = dv.date.now().toDate();
let dayOfWeek = today.getDay();

if (dayOfWeek === 0) {
  dv.header(1, "Happy Sunday!");
} else if (dayOfWeek === 5) {
  dv.header(1, "TGIF!");
} else {
  dv.header(1, "Keep going!");
}
```

### "Show upcoming deadlines"
```javascript
let today = dv.date.now().toDate();
let upcomingDeadline = dv.pages('"Projects"')
  .where(p => p.deadline && p.deadline.toDate() > today)
  .sort(p => p.deadline.toDate(), false);

if (upcomingDeadline.length > 0) {
  dv.header(2, "⏰ Upcoming Deadlines");
  dv.table(
    ["Project", "Deadline", "Days Left"],
    upcomingDeadline.map(p => [
      p.file.link,
      p.deadline.toDateString(),
      Math.floor((p.deadline.toDate() - today) / (1000 * 60 * 60 * 24))
    ])
  );
} else {
  dv.paragraph("No upcoming deadlines!");
}
```

---

## Date Calculations

### "Reading pace calculator"
```javascript
let books = dv.pages("#book").where(b => b.completed && b.started);

for (let book of books) {
  let days = Math.floor((book.completed.toDate() - book.started.toDate()) / (1000 * 60 * 60 * 24));
  let pages = book.pages || 300;
  let pagesPerDay = (pages / days).toFixed(1);

  dv.paragraph(
    `${book.file.link}: ${pagesPerDay} pages/day over ${days} days`
  );
}
```

### "Days since last review"
```javascript
let notes = dv.pages("#learning").where(n => n.last-review);

for (let note of notes) {
  let daysSince = Math.floor(
    (dv.date.now().toDate() - note["last-review"].toDate()) / (1000 * 60 * 60 * 24)
  );

  dv.paragraph(`${note.file.link}: reviewed ${daysSince} days ago`);
}
```

---

## Building Custom Lists

### "Books with emoji rating"
```javascript
let books = dv.pages("#book").sort(b => b.rating, true);

let list = books.map(b => {
  let stars = "⭐".repeat(b.rating);
  return `${b.file.link} ${stars}`;
});

dv.list(list);
```

### "Active projects status board"
```javascript
let projects = dv.pages('"Projects"').where(p => p.status === "active");

for (let project of projects.sort(p => p.priority)) {
  let progressBar = "█".repeat(project.progress || 0) + "░".repeat(100 - (project.progress || 0));

  dv.paragraph(`**${project.file.link}** [${progressBar}] ${project.progress || 0}%`);
  dv.paragraph(`Due: ${project.deadline}`);
  dv.paragraph("---");
}
```

---

## Working with Arrays

### "Flatten nested lists"
```javascript
let books = dv.pages("#book");

// Flatten all tags from all books
let allTags = [];
for (let book of books) {
  if (book.tags) {
    allTags = allTags.concat(book.tags);
  }
}

// Count tag frequency
let tagCounts = {};
for (let tag of allTags) {
  tagCounts[tag] = (tagCounts[tag] || 0) + 1;
}

// Sort by frequency
let sorted = Object.entries(tagCounts)
  .sort((a, b) => b[1] - a[1]);

dv.paragraph("**Top Tags**");
for (let [tag, count] of sorted.slice(0, 10)) {
  dv.paragraph(`${tag}: ${count}`);
}
```

### "Combine multiple fields into one display"
```javascript
let books = dv.pages("#book");

for (let book of books) {
  let info = [];
  if (book.author) info.push(book.author);
  if (book.genre) info.push(book.genre);
  if (book.rating) info.push(`${book.rating}⭐`);

  dv.paragraph(`${book.file.link} — ${info.join(" • ")}`);
}
```

---

## Statistics and Summaries

### "Reading statistics"
```javascript
let books = dv.pages("#book").where(b => b.status === "completed");

let stats = {
  total: books.length,
  avgRating: books.map(b => b.rating || 0).reduce((a, b) => a + b, 0) / books.length,
  totalPages: books.map(b => b.pages || 0).reduce((a, b) => a + b, 0),
  genres: new Set(books.map(b => b.genre)).size
};

dv.paragraph(`📚 **Reading Stats**`);
dv.paragraph(`Total books: ${stats.total}`);
dv.paragraph(`Average rating: ${stats.avgRating.toFixed(2)}⭐`);
dv.paragraph(`Total pages: ${stats.totalPages}`);
dv.paragraph(`Genres: ${stats.genres}`);
```

### "Completion rate by category"
```javascript
let projects = dv.pages('"Projects"');

let byStatus = projects.groupBy(p => p.status);

for (let [status, pages] of byStatus) {
  dv.paragraph(`**${status}**: ${pages.length} projects`);
}
```

---

## Custom HTML/Markdown Rendering

### "Card-style layout"
```javascript
let books = dv.pages("#book").where(b => b.status === "completed").slice(0, 5);

for (let book of books) {
  dv.paragraph(
    `<div style="border: 1px solid #ccc; padding: 10px; margin: 10px 0; border-radius: 5px;">` +
    `<strong>${book.file.link}</strong><br/>` +
    `by ${book.author}<br/>` +
    `${book.rating}⭐ (${book.pages} pages)<br/>` +
    `</div>`
  );
}
```

### "Timeline view"
```javascript
let events = dv.pages("#event").sort(e => e.date, false);

for (let event of events) {
  dv.paragraph(
    `**${event.date.toDateString()}** — ${event.file.link}`
  );
  if (event.description) {
    dv.paragraph(`_${event.description}_`);
  }
}
```

---

## Accessing Different Field Types

```javascript
// Text field
let author = book.author;

// Number field
let rating = book.rating;

// Date field (convert to JS Date)
let date = book.completed.toDate();

// Link field (convert to string)
let relatedLink = book.related.path;

// Array field
let tags = book.tags || [];
if (tags.includes("favorite")) { ... }

// Check if field exists
if (book.notes) { ... }
```

