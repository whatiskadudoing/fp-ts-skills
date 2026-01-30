# fp-ts Skills for AI Agents

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0+-blue.svg)](https://www.typescriptlang.org/)
[![fp-ts](https://img.shields.io/badge/fp--ts-2.x-purple.svg)](https://github.com/gcanti/fp-ts)

> Practical functional programming for TypeScript. No jargon, just patterns that work.

## Quick Install

```bash
# List available skills
npx skills add whatiskadudoing/fp-ts-skills --list

# Install all skills
npx skills add whatiskadudoing/fp-ts-skills --all

# Install specific skill
npx skills add whatiskadudoing/fp-ts-skills --skill fp-react
```

Works with Claude Code, Cursor, GitHub Copilot, Gemini CLI, and any agent supporting the SKILL.md format.

## Skill Tiers

| Tier | Lines | Tokens | When Loaded |
|------|-------|--------|-------------|
| **quick-ref** | 50-100 | ~400 | Always cheap - use freely |
| **core** | 300-600 | ~2,500 | On invocation |
| **advanced** | 600-1500 | ~6,000 | Opt-in for deep dives |

### Quick References (Always Cheap)

| Skill | Lines | Use For |
|-------|-------|---------|
| [fp-types-ref](skills/quick-ref/fp-types-ref/SKILL.md) | 75 | "Which type should I use?" |
| [fp-pipe-ref](skills/quick-ref/fp-pipe-ref/SKILL.md) | 70 | pipe() and flow() patterns |
| [fp-option-ref](skills/quick-ref/fp-option-ref/SKILL.md) | 70 | Handling nullable values |
| [fp-either-ref](skills/quick-ref/fp-either-ref/SKILL.md) | 75 | Error handling basics |
| [fp-taskeither-ref](skills/quick-ref/fp-taskeither-ref/SKILL.md) | 85 | Async error handling |

### Core Patterns

| Skill | Lines | Use For |
|-------|-------|---------|
| [fp-pragmatic](skills/fp-pragmatic/SKILL.md) | 598 | 80/20 of FP, when NOT to use it |
| [fp-errors](skills/fp-errors/SKILL.md) | 857 | Error handling patterns |
| [fp-async](skills/fp-async/SKILL.md) | 964 | Async operations |
| [fp-react](skills/fp-react/SKILL.md) | 790 | React state, forms, fetching |
| [fp-compose](skills/fp-compose/SKILL.md) | 837 | Function composition |
| [fp-validation](skills/fp-validation/SKILL.md) | 912 | Form/API validation |

### Advanced (Opt-in)

| Skill | Lines | Use For |
|-------|-------|---------|
| [fp-data-transforms](skills/fp-data-transforms/SKILL.md) | 1516 | Complex data manipulation |
| [fp-backend](skills/fp-backend/SKILL.md) | 1335 | Service layer, DI |
| [fp-refactor](skills/fp-refactor/SKILL.md) | 1781 | Migration from imperative |
| [fp-algebraic-types](skills/fp-algebraic-types/SKILL.md) | 1445 | Semigroup, Monoid, etc. |
| [fp-side-effects](skills/fp-side-effects/SKILL.md) | 2042 | Effect management |

## Bundles

Install multiple related skills at once:

```bash
# Essentials - quick refs only (375 lines)
npx skills add whatiskadudoing/fp-ts-skills --skill fp-types-ref
npx skills add whatiskadudoing/fp-ts-skills --skill fp-pipe-ref
npx skills add whatiskadudoing/fp-ts-skills --skill fp-option-ref
npx skills add whatiskadudoing/fp-ts-skills --skill fp-either-ref
npx skills add whatiskadudoing/fp-ts-skills --skill fp-taskeither-ref

# React Developer
npx skills add whatiskadudoing/fp-ts-skills --skill fp-react

# Backend Developer
npx skills add whatiskadudoing/fp-ts-skills --skill fp-async
npx skills add whatiskadudoing/fp-ts-skills --skill fp-backend
```

## Philosophy

- **Token-efficient**: Small focused skills that compose well
- **No jargon**: "chain operations" not "monadic bind"
- **When NOT to use FP**: We tell you when a for loop is clearer
- **Pragmatic**: The 80/20 of functional programming

## Writing Good Descriptions

Claude uses **LLM reasoning** (not keywords) to decide when to invoke skills. Write descriptions that cover:

```yaml
---
name: fp-validation
description: Form and API validation with error accumulation. Use when validating user input, handling multiple errors, or building type-safe validators.
---
```

## Resources & Sources

These skills were created by studying and synthesizing patterns from:

### Core Libraries

- [fp-ts](https://github.com/gcanti/fp-ts) - The functional programming library for TypeScript
- [Effect](https://github.com/Effect-TS/effect) - Next-generation FP in TypeScript

### Books & Guides

- [Functional-Light JavaScript](https://github.com/getify/Functional-Light-JS) - Kyle Simpson's pragmatic approach to FP
- [Mostly Adequate Guide to FP](https://github.com/MostlyAdequate/mostly-adequate-guide) - Professor Frisby's FP fundamentals
- [Functional Programming Guide](https://github.com/enricopolanski/functional-programming) - Enrico Polanski's comprehensive guide

### Articles

- [Level Up React: Functional Programming in React](https://www.56kode.com/posts/level-up-react-functional-programming-in-react/) - 56kode
- [Fundamentals of Functional Programming in React](https://blog.logrocket.com/fundamentals-functional-programming-react/) - LogRocket
- [Hooked on React: React 19's Function Component Superpowers](https://dev.to/philipjohnbasile/hooked-on-react-the-complete-guide-to-react-19s-function-component-superpowers-1hdj) - Dev.to
- [Functional Programming in React](https://blog.saeloun.com/2024/07/25/functional-programming-in-react/) - Saeloun

## License

MIT

---

**Keywords**: functional programming, typescript, fp-ts, claude code, ai skills, option type, either type, taskeither, react hooks, node.js, error handling, validation, pipe, composition
