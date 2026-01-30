# Functional Programming Skills for TypeScript & Claude Code

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0+-blue.svg)](https://www.typescriptlang.org/)
[![fp-ts](https://img.shields.io/badge/fp--ts-2.x-purple.svg)](https://github.com/gcanti/fp-ts)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skills-orange.svg)](https://claude.ai)

> **Practical functional programming patterns for TypeScript developers. No academic jargon, just code that works.**

A collection of 17 AI-powered skills for [Claude Code](https://claude.ai/claude-code) that help you write better TypeScript using functional programming patterns. Each skill teaches real-world patterns with before/after examples.

---

## Why This Exists

Functional programming resources often suffer from:
- **Too much theory** - Category theory and abstract math before practical examples
- **Academic jargon** - "Monadic bind" instead of "chain operations"
- **No guidance on when NOT to use FP** - Not everything needs to be functional

This skill set takes a **"Functional-Light"** approach:
- Learn patterns that give **80% of the benefit with 20% of the complexity**
- See **before/after examples** so the improvement is obvious
- Know **when a for loop is actually clearer** than a functional approach

---

## Quick Start

### Installation

```bash
# Clone the repository
git clone https://github.com/whatiskadudoing/fp-ts-skills.git

# Copy all skills to Claude Code
cp fp-ts-skills/skills/*.md ~/.claude/skills/

# Or copy just the essentials
cp fp-ts-skills/skills/fp-pragmatic.md ~/.claude/skills/
cp fp-ts-skills/skills/fp-errors.md ~/.claude/skills/
cp fp-ts-skills/skills/fp-compose.md ~/.claude/skills/
```

### First Skills to Learn

| Order | Skill | What You'll Learn |
|-------|-------|-------------------|
| 1 | [fp-pragmatic](skills/fp-pragmatic.md) | The 80/20 of FP - only patterns that matter |
| 2 | [fp-compose](skills/fp-compose.md) | `pipe()` - chain operations cleanly |
| 3 | [fp-errors](skills/fp-errors.md) | Handle errors without try/catch hell |

---

## All Skills

### Practical Skills (Start Here)

These skills focus on immediate improvements with no jargon:

| Skill | Description | Key Patterns |
|-------|-------------|--------------|
| [fp-pragmatic](skills/fp-pragmatic.md) | The 80/20 rule for FP | When to use FP, when NOT to |
| [fp-compose](skills/fp-compose.md) | Function composition | `pipe()`, `flow()`, reusable utilities |
| [fp-errors](skills/fp-errors.md) | Error handling | Either, Result pattern, no exceptions |
| [fp-data-transforms](skills/fp-data-transforms.md) | Data manipulation | map, filter, reduce, groupBy |
| [fp-async](skills/fp-async.md) | Async operations | TaskEither, parallel vs sequential |
| [fp-immutable](skills/fp-immutable.md) | Immutable updates | Spread patterns, Immer, nested updates |

### Core fp-ts Patterns

Deep dives into fp-ts library patterns:

| Skill | Description | Key Patterns |
|-------|-------------|--------------|
| [fp-fundamentals](skills/fp-fundamentals.md) | FP foundations | Pure functions, currying, closures |
| [fp-pipe-flow](skills/fp-pipe-flow.md) | Composition deep dive | pipe vs flow, debugging pipelines |
| [fp-option-either](skills/fp-option-either.md) | Nullable & errors | Option, Either, fromNullable |
| [fp-task-either](skills/fp-task-either.md) | Async + errors | TaskEither, tryCatch, parallel |
| [fp-validation](skills/fp-validation.md) | Form validation | Error accumulation, sequenceS |
| [fp-do-notation](skills/fp-do-notation.md) | Readable async | Do, bind, apS patterns |
| [fp-side-effects](skills/fp-side-effects.md) | Managing effects | IO, pure core / impure shell |

### Application Patterns

Full application architecture:

| Skill | Description | Use Case |
|-------|-------------|----------|
| [fp-react](skills/fp-react.md) | React integration | Hooks, state, RemoteData |
| [fp-backend](skills/fp-backend.md) | Backend services | ReaderTaskEither, DI, middleware |
| [fp-refactor](skills/fp-refactor.md) | Migration guide | Converting imperative to FP |
| [fp-algebraic-types](skills/fp-algebraic-types.md) | Type classes | Semigroup, Monoid, Eq, Ord |

---

## Learning Paths

### Path 1: "I just want cleaner code"

```
fp-pragmatic → fp-compose → fp-data-transforms → fp-immutable
```

### Path 2: "I'm tired of null checks and try/catch"

```
fp-pragmatic → fp-errors → fp-option-either → fp-validation
```

### Path 3: "My async code is a mess"

```
fp-async → fp-task-either → fp-do-notation
```

### Path 4: "Building a React app"

```
fp-pragmatic → fp-errors → fp-async → fp-react
```

### Path 5: "Building a Node.js/Deno backend"

```
fp-pragmatic → fp-errors → fp-async → fp-backend
```

---

## Examples

### Before: Nested Try/Catch Hell

```typescript
async function getUser(id: string) {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) {
      throw new Error('Failed to fetch');
    }
    try {
      const data = await response.json();
      if (!data.user) {
        throw new Error('User not found');
      }
      return data.user;
    } catch (e) {
      throw new Error('Invalid JSON');
    }
  } catch (e) {
    console.error(e);
    return null;
  }
}
```

### After: Clean Pipeline

```typescript
import { pipe } from 'fp-ts/function'
import * as TE from 'fp-ts/TaskEither'

const getUser = (id: string) => pipe(
  TE.tryCatch(
    () => fetch(`/api/users/${id}`),
    () => ({ type: 'NETWORK_ERROR' } as const)
  ),
  TE.filterOrElse(
    (res) => res.ok,
    () => ({ type: 'HTTP_ERROR' } as const)
  ),
  TE.flatMap((res) => TE.tryCatch(
    () => res.json(),
    () => ({ type: 'PARSE_ERROR' } as const)
  )),
  TE.filterOrElse(
    (data) => data.user != null,
    () => ({ type: 'NOT_FOUND' } as const)
  ),
  TE.map((data) => data.user)
)
```

**Why it's better:**
- Every error type is explicit in the type system
- No nested try/catch blocks
- Easy to add retry, logging, or fallbacks
- Composable with other operations

---

## Philosophy

### The "Functional-Light" Approach

This skill set is inspired by Kyle Simpson's [Functional-Light JavaScript](https://github.com/getify/Functional-Light-JS):

> "Functional-Light Programming is about using functional programming patterns where they help, not as a religion."

### Our Principles

1. **Readability over purity** - If FP makes code harder to understand, use something simpler
2. **Pragmatic adoption** - Start with the patterns that help most
3. **No jargon** - "chain operations" not "monadic bind"
4. **When NOT to use FP** - We tell you when a for loop is clearer
5. **Real TypeScript** - All examples are copy-paste ready

---

## Tech Stack

These skills work with:

- **TypeScript** 4.7+ (5.0+ recommended)
- **fp-ts** 2.x - The functional programming library for TypeScript
- **React** 17+ (for fp-react skill)
- **Node.js** / **Deno** / **Bun** (for fp-backend skill)
- **Claude Code** - AI-powered coding assistant

### Related Libraries

- [fp-ts](https://github.com/gcanti/fp-ts) - Core FP library
- [io-ts](https://github.com/gcanti/io-ts) - Runtime type validation
- [monocle-ts](https://github.com/gcanti/monocle-ts) - Optics (lenses, prisms)
- [Effect](https://github.com/Effect-TS/effect) - Next-gen FP (fp-ts successor)

---

## Contributing

Contributions welcome! Please follow these guidelines:

### Content Guidelines

- **No unnecessary jargon** - Explain concepts in plain English
- **Before/after examples** - Show the improvement
- **When NOT to use** - Be honest about trade-offs
- **Working TypeScript** - All code should compile

### How to Contribute

1. Fork the repository
2. Create a feature branch (`git checkout -b add-new-skill`)
3. Add your skill to `skills/`
4. Update README.md with the new skill
5. Submit a pull request

### Skill Format

```markdown
---
name: skill-name
description: Brief description
version: 1.0.0
author: Your Name
tags: [fp-ts, typescript, relevant-tags]
---

# Skill Title

[Content with examples]
```

---

## Resources

### Inspiration

- [fp-ts](https://github.com/gcanti/fp-ts) - The library these skills teach
- [Functional-Light JavaScript](https://github.com/getify/Functional-Light-JS) - Pragmatic FP philosophy
- [Mostly Adequate Guide](https://github.com/MostlyAdequate/mostly-adequate-guide) - FP fundamentals
- [Functional Programming Guide](https://github.com/enricopolanski/functional-programming) - Theory reference

### Further Learning

- [fp-ts Documentation](https://gcanti.github.io/fp-ts/)
- [Practical Guide to fp-ts](https://rlee.dev/practical-guide-to-fp-ts-part-1)
- [Effect Documentation](https://effect.website/) - The future of FP in TypeScript

---

## License

MIT License - Use these skills however you want.

---

## Keywords

functional programming, typescript, fp-ts, claude code, ai skills, functional typescript, error handling typescript, pipe function, option type, either type, taskeither, react functional programming, node.js functional, deno functional, immutable data, pure functions, function composition, practical fp, functional light

---

<p align="center">
  Made with fp-ts and Claude Code
</p>
