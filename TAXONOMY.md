# Blog Taxonomy

Reference for consistent categorization across posts.

## Categories

One category per post. Title Case in the front matter.

| Category | Use for |
|---|---|
| `Tutorials` | Step-by-step walkthroughs: "Your first entity", "Building a REST API with Trysil.Http", "Multi-tenant from scratch" |
| `Deep Dives` | How a feature works internally: "How the identity map works", "Anatomy of a JOIN query", "Inside `TTResolver`" |
| `Releases` | Release notes, changelogs, migration guides between Trysil versions |
| `Design Notes` | Architectural decisions and their tradeoffs: "Why full-clone in `TTSession`", "No interfaces on generics — and why it's fine", "Soft delete vs hard delete" |
| `Announcements` | Events, milestones, external coverage: Delphi Day talks, Show HN, new YouTube videos, GetIt publication |

## Tags

Multiple tags per post. Kebab-case, lowercase. Pick from the list below; add new ones only when a concept recurs.

### Core features

`orm`, `entity-mapping`, `filter-builder`, `identity-map`, `session`, `lazy-loading`, `transactions`, `events`, `validation`, `raw-sql`, `joins`, `soft-delete`, `change-tracking`, `version-column`, `sequences`, `relations`, `nullable`

### Modules

`trysil-json`, `trysil-http`, `multi-tenant`, `jwt`, `cors`, `authentication`, `load-balancing`

### Databases

`sqlite`, `postgresql`, `firebird`, `sqlserver`, `firedac`

### Delphi language

`delphi`, `rtti`, `generics`, `attributes`, `fireDAC`

### Content type

`tutorial`, `internals`, `release-notes`, `benchmark`, `case-study`

## Front matter template

```yaml
---
title: "Post title"
date: 2026-04-20 12:00:00 +0100
categories: [Deep Dives]
tags: [identity-map, orm, internals]
---
```

## Conventions

- **One category**, even if the post spans multiple themes — pick the primary one.
- **3–5 tags** per post is the sweet spot. More than 7 dilutes the taxonomy.
- If introducing a new tag, check whether an existing one covers it.
- Avoid `trysil` as a tag — the whole blog is about Trysil.
- Category names are stable; renaming requires updating every post.
