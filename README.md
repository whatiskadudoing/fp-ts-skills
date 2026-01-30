# fp-ts Skills for Claude Code

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0+-blue.svg)](https://www.typescriptlang.org/)
[![fp-ts](https://img.shields.io/badge/fp--ts-2.x-purple.svg)](https://github.com/gcanti/fp-ts)

> Practical functional programming for TypeScript. No jargon, just patterns that work.

## Quick Install

```bash
# Clone and copy essentials (quick-refs only - minimal tokens)
git clone https://github.com/whatiskadudoing/fp-ts-skills.git
cp fp-ts-skills/skills/quick-ref/*.md ~/.claude/skills/

# Add core patterns too
cp fp-ts-skills/skills/fp-pragmatic.md ~/.claude/skills/
cp fp-ts-skills/skills/fp-errors.md ~/.claude/skills/
```

## Skill Tiers

| Tier | Lines | Tokens | When Loaded |
|------|-------|--------|-------------|
| **quick-ref** | 50-100 | ~400 | Always cheap - use freely |
| **core** | 300-600 | ~2,500 | On invocation |
| **advanced** | 600-1500 | ~6,000 | Opt-in for deep dives |

### Quick References (Always Cheap)

| Skill | Lines | Use For |
|-------|-------|---------|
| [fp-types-ref](skills/quick-ref/fp-types-ref.md) | 75 | "Which type should I use?" |
| [fp-pipe-ref](skills/quick-ref/fp-pipe-ref.md) | 70 | pipe() and flow() patterns |
| [fp-option-ref](skills/quick-ref/fp-option-ref.md) | 70 | Handling nullable values |
| [fp-either-ref](skills/quick-ref/fp-either-ref.md) | 75 | Error handling basics |
| [fp-taskeither-ref](skills/quick-ref/fp-taskeither-ref.md) | 85 | Async error handling |

### Core Patterns

| Skill | Lines | Use For |
|-------|-------|---------|
| [fp-pragmatic](skills/fp-pragmatic.md) | 598 | 80/20 of FP, when NOT to use it |
| [fp-errors](skills/fp-errors.md) | 857 | Error handling patterns |
| [fp-async](skills/fp-async.md) | 964 | Async operations |
| [fp-react](skills/fp-react.md) | 790 | React state, forms, fetching |
| [fp-compose](skills/fp-compose.md) | 837 | Function composition |
| [fp-validation](skills/fp-validation.md) | 912 | Form/API validation |

### Advanced (Opt-in)

| Skill | Lines | Use For |
|-------|-------|---------|
| [fp-data-transforms](skills/fp-data-transforms.md) | 1516 | Complex data manipulation |
| [fp-backend](skills/fp-backend.md) | 1335 | Service layer, DI |
| [fp-refactor](skills/fp-refactor.md) | 1781 | Migration from imperative |
| [fp-algebraic-types](skills/fp-algebraic-types.md) | 1445 | Semigroup, Monoid, etc. |
| [fp-side-effects](skills/fp-side-effects.md) | 2042 | Effect management |

## Bundles

### Essentials (375 lines)
Just quick-refs. Minimal token cost.
```bash
cp fp-ts-skills/skills/quick-ref/*.md ~/.claude/skills/
```

### Recommended (1,830 lines)
Quick-refs + practical patterns.
```bash
cp fp-ts-skills/skills/quick-ref/*.md ~/.claude/skills/
cp fp-ts-skills/skills/fp-pragmatic.md ~/.claude/skills/
cp fp-ts-skills/skills/fp-errors.md ~/.claude/skills/
```

### React Developer (1,095 lines)
```bash
cp fp-ts-skills/skills/quick-ref/fp-{types,option,either,taskeither}-ref.md ~/.claude/skills/
cp fp-ts-skills/skills/fp-react.md ~/.claude/skills/
```

### Backend Developer (2,529 lines)
```bash
cp fp-ts-skills/skills/quick-ref/fp-{types,either,taskeither}-ref.md ~/.claude/skills/
cp fp-ts-skills/skills/fp-async.md ~/.claude/skills/
cp fp-ts-skills/skills/fp-backend.md ~/.claude/skills/
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

## Resources

- [fp-ts](https://github.com/gcanti/fp-ts) - The library
- [Functional-Light JS](https://github.com/getify/Functional-Light-JS) - Philosophy
- [Effect](https://github.com/Effect-TS/effect) - Future of FP in TS

## License

MIT

---

**Keywords**: functional programming, typescript, fp-ts, claude code, ai skills, option type, either type, taskeither, react hooks, node.js, error handling, validation, pipe, composition
