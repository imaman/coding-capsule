# CLAUDE.md

## Package manager
Use `yarn` (not npm or pnpm).

## Build
```
yarn build
```

## Project
TypeScript ESM project. Build output goes to `dist/`.

## TypeScript style
No type coercion. No `as` casts, no non-null assertions (`!`). Use runtime narrowing or `demandOption`/defaults to let types flow naturally. Use `failMe()` (returns `never`) with `??` to replace non-null assertions or `||` to replace non-falsy assertions.
