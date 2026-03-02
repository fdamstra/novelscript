# NovelScript

A pipeline-oriented, pattern-driven programming language designed for AI agents to write, with human readability as a secondary goal.

## Project Structure

- `docs/specification.md` — Full language specification
- `docs/tutorial.md` — Tutorial for experienced programmers

## Language Key Points

- File extension: `.ns`
- Pipeline-first (`|>`), conditional pipeline (`|?>`) for error propagation
- Pattern matching is the **only** branching mechanism (no if/else)
- Contracts: `require` (preconditions), `ensure` (postconditions)
- `proof` blocks are inline tests, first-class language constructs
- `biject` defines bidirectional/reversible functions
- No null — uses `Option[T]` and `Result[T, E]`
- Immutable by default, `let mut` for mutable bindings, `<-` for mutation
- No loops — iteration via streams, pipelines, and recursion
- Shapes are named structural record types, updated with `with`
- `@intent` annotations for machine-readable documentation

## Status

- Language design: drafted (spec + tutorial), not yet reviewed/finalized
- Interpreter: not yet started
- Implementation language: not yet decided
