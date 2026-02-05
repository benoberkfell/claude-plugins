# Obsidian-Specific Metadata Patterns

Real-world examples of how to structure metadata for common Obsidian use cases.

## Daily Notes / Life Tracking

**Use case:** Track sleep, mood, exercise, diet, focus quality over time.

### Setup

```markdown
---
date: 2024-03-15
mood: 7
sleep-hours: 7.5
exercise: 30
focus-quality: 8
---

# 2024-03-15

Slept [sleep-start:: 11:00 PM] to [sleep-end:: 6:30 AM].

Today's focus: [focus-areas:: [deep work, planning]]

Evening note: [notes:: Good day, productive afternoon session.]
```

### Query
```dataview
TABLE date, mood, sleep-hours, exercise, focus-quality
FROM "Daily Notes"
WHERE date >= date(today) - dur(30 days)
SORT date DESC
```

---

## Book Tracking

**Use case:** Build a searchable book database with ratings, status, and notes.

### Setup

```markdown
---
type: book
author: "N.K. Jemisin"
title: "The Fifth Season"
published: 2015-08-03
series: "Broken Earth"
series-order: 1
genre: [fantasy, sci-fi, speculative]
status: completed
rating: 5
started: 2024-01-15
completed: 2024-03-10
---

# The Fifth Season

From [author:: N.K. Jemisin], published [published:: 2015-08-03].

**Series:** [series:: Broken Earth] #[series-order:: 1]

**Status:** [status:: completed]

**My Rating:** [rating:: 5] / 5

**Key themes:** [themes:: [[Identity]], [[Power]], [[Survival]]]

**Notes:** [notes:: Brilliant worldbuilding, emotionally devastating, absolutely recommend.]

**Favorite quotes:**
- Essun's arc
- The Fulcrum introduction
- [quote:: "You are not going to like this story"]
```

### Query
```dataview
TABLE author, published, status, rating
FROM #book
WHERE status = "completed"
SORT rating DESC, completed DESC
```

---

## Project Management

**Use case:** Track project status, deadlines, deliverables, team members.

### Setup

```markdown
---
type: project
project-name: "Website Redesign"
status: in-progress
start-date: 2024-02-01
target-completion: 2024-04-30
priority: high
team-lead: [[Jane Smith]]
stakeholders: [[[Jane Smith]], [[Bob Johnson]]]
---

# Website Redesign

**Status:** [status:: in-progress]
**Priority:** [priority:: high]
**Target Completion:** [target:: 2024-04-30]
**Team Lead:** [lead:: [[Jane Smith]]]

## Deliverables
- [x] Design mockups [due:: 2024-03-01]
- [ ] Frontend implementation [due:: 2024-04-15]
- [ ] Testing & QA [due:: 2024-04-25]
- [ ] Launch [due:: 2024-04-30]

## Open Blockers
- [Waiting on design approval from stakeholders]
- [Database schema needs finalization]

**Progress:** [progress:: 40%]
```

### Query
```dataview
TABLE project-name, status, target-completion, priority, lead
WHERE status != "completed"
SORT priority DESC, target-completion ASC
```

---

## Task Management (GTD / Todo Tracking)

**Use case:** Centralized task collection with context, priority, and due dates.

### Setup

Note with tasks:
```markdown
# Q1 Goals

- [ ] Launch new feature [due:: 2024-03-31] [priority:: high] [context:: [[Frontend Team]]]
- [ ] Write documentation [due:: 2024-04-15] [priority:: medium]
  - [ ] API docs [due:: 2024-04-10]
  - [ ] User guide [due:: 2024-04-15]
- [x] Complete training [due:: 2024-02-28] [priority:: high]
```

### Query (using TASK type)
```dataview
TASK
WHERE !completed AND priority = "high"
GROUP BY file.link
SORT due ASC
```

---

## Learning Notes / Spaced Repetition

**Use case:** Track learning materials, review schedule, difficulty level.

### Setup

```markdown
---
type: learning
subject: JavaScript
topic: Async/Await
source: [[MDN Web Docs]]
difficulty: intermediate
date-learned: 2024-03-10
next-review: 2024-03-17
review-count: 1
---

# Async/Await in JavaScript

**Source:** [source:: [[MDN Web Docs]]]
**Difficulty:** [difficulty:: intermediate]
**First learned:** [learned:: 2024-03-10]
**Next review:** [next-review:: 2024-03-17]
**Review count:** [review-count:: 1]

## Key Points
- Async functions return promises
- Await pauses execution
- Error handling with try/catch

**Confidence:** [confidence:: 7/10]
```

### Query
```dataview
TABLE topic, difficulty, next-review, review-count
WHERE next-review <= date(today) AND type = "learning"
SORT next-review ASC
```

---

## Recipe Collection

**Use case:** Searchable recipe database with ingredients, time, difficulty, tags.

### Setup

```markdown
---
type: recipe
cuisine: Italian
difficulty: easy
prep-time: dur(15 minutes)
cook-time: dur(20 minutes)
servings: 4
rating: 4.5
last-made: 2024-03-10
---

# Pasta Carbonara

**Prep time:** [prep:: dur(15 minutes)]
**Cook time:** [cook:: dur(20 minutes)]
**Servings:** [servings:: 4]
**Difficulty:** [difficulty:: easy]
**Last made:** [made:: 2024-03-10]
**Rating:** [rating:: 4.5]

## Ingredients
- [ingredient:: 400g spaghetti]
- [ingredient:: 200g guanciale]
- [ingredient:: 4 eggs]
- [ingredient:: 100g Pecorino Romano]

**Tags:** [tags:: [italian, pasta, quick, favorite]]
```

### Query
```dataview
TABLE cuisine, prep-time, cook-time, difficulty, rating
FROM #recipe
WHERE contains(tags, "quick")
SORT rating DESC
```

---

## Map of Contents / Hub Pages

**Use case:** Organize notes by topic with summaries and relationship tracking.

### Setup

```markdown
---
type: moc
topic: "Machine Learning"
subtopics: [[[Neural Networks]], [[Supervised Learning]], [[Optimization]]]
related: [[[Python]], [[Statistics]]]
last-updated: 2024-03-15
completeness: 60
---

# Machine Learning Hub

**Topic:** Machine Learning
**Updated:** [updated:: 2024-03-15]
**Completeness:** [completeness:: 60%]

## Main Areas
- [Neural Networks]
- [Supervised Learning]
- [Optimization]

**Related Topics:** [[Python]], [[Statistics]]

## Key Concepts
- [[Backpropagation]]
- [[Gradient Descent]]
- [[Loss Functions]]
```

### Query
```dataview
TABLE topic, completeness, updated
WHERE type = "moc"
SORT updated DESC
```

---

## Decision Tree: Which Pattern?

- **Tracking daily habits?** → Daily Notes pattern
- **Building a database of items (books, movies, recipes)?** → Database pattern (Book/Recipe)
- **Managing work across multiple people/deadlines?** → Project Management pattern
- **Collecting scattered tasks?** → Task Management pattern with TASK queries
- **Organizing learning materials?** → Learning Notes pattern
- **Organizing by topic/concept?** → MOC pattern

Most vaults use **multiple patterns**. Your daily notes track habits. Your projects track work. Your books track consumption. Mix and match based on what you need to query.
