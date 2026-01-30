# fp-ts-skills

Practical functional programming skills for TypeScript. No academic jargon, just patterns that work.

## Quick Start

**New to FP?** Start here:
1. **fp-pragmatic** - The 80/20 of functional programming
2. **fp-compose** - Build things from smaller pieces
3. **fp-errors** - Handle errors without try/catch hell

## Installation

```bash
# Copy to Claude Code skills directory
cp skills/*.md ~/.claude/skills/

# Or install individual skills
cp skills/fp-pragmatic.md ~/.claude/skills/
```

## Skills Overview

### Start Here (Practical, No Jargon)

| Skill | What It Does |
|-------|--------------|
| [fp-pragmatic](skills/fp-pragmatic.md) | The 80/20 of FP - patterns that matter, when NOT to use FP |
| [fp-compose](skills/fp-compose.md) | pipe() and building reusable functions |
| [fp-errors](skills/fp-errors.md) | Handle errors as values, not exceptions |
| [fp-data-transforms](skills/fp-data-transforms.md) | map/filter/reduce, object transforms, null-safe access |
| [fp-async](skills/fp-async.md) | Async without try/catch hell |
| [fp-immutable](skills/fp-immutable.md) | Update data without mutation |

### Core Patterns

| Skill | What It Does |
|-------|--------------|
| [fp-fundamentals](skills/fp-fundamentals.md) | Pure functions, currying, composition basics |
| [fp-pipe-flow](skills/fp-pipe-flow.md) | Deep dive into pipe and flow |
| [fp-option-either](skills/fp-option-either.md) | Option and Either in detail |
| [fp-task-either](skills/fp-task-either.md) | TaskEither for async operations |
| [fp-validation](skills/fp-validation.md) | Form validation with error accumulation |
| [fp-do-notation](skills/fp-do-notation.md) | Readable async workflows |
| [fp-side-effects](skills/fp-side-effects.md) | Managing side effects |

### Advanced / Reference

| Skill | What It Does |
|-------|--------------|
| [fp-algebraic-types](skills/fp-algebraic-types.md) | Sum types, Semigroup, Monoid, Eq, Ord |
| [fp-react](skills/fp-react.md) | Using fp-ts with React |
| [fp-backend](skills/fp-backend.md) | Backend patterns, ReaderTaskEither, DI |
| [fp-refactor](skills/fp-refactor.md) | Converting imperative code to FP |

## The "Functional-Light" Philosophy

This skill set follows a pragmatic approach:

- **Readability first** - If FP makes code harder to read, don't use it
- **80/20 rule** - Focus on patterns that give the most benefit
- **No jargon** - "chain operations" not "monadic bind"
- **When NOT to use FP** - Sometimes a for loop is clearer

## Learning Paths

### "I just want cleaner code"
1. fp-pragmatic → fp-compose → fp-data-transforms

### "I'm tired of null checks"
1. fp-pragmatic → fp-errors → fp-option-either

### "Async code is a mess"
1. fp-async → fp-task-either → fp-do-notation

### "Building a full app"
1. fp-pragmatic → fp-errors → fp-async → fp-react or fp-backend

## Resources

Based on concepts from:
- [fp-ts](https://github.com/gcanti/fp-ts) - The library we use
- [Functional-Light JS](https://github.com/getify/Functional-Light-JS) - Pragmatic approach
- [Mostly Adequate Guide](https://github.com/MostlyAdequate/mostly-adequate-guide) - FP fundamentals

## Contributing

Keep it practical:
- No unnecessary jargon
- Show before/after examples
- Explain when NOT to use a pattern
- TypeScript examples that work in real projects

## License

MIT
