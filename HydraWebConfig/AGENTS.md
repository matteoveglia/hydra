# HydraWebConfig coding rules

This file applies to `HydraWebConfig/` and supplements the repository root guide.

## Purpose and boundaries

HydraWebConfig is a React 19 + TypeScript config editor built with Vite. Its serialized output is consumed by the .NET Hydra config loader, so UI convenience must not change config semantics silently.

- `src/types.ts` defines the editor's config model.
- `src/defaults.ts` owns new-item defaults.
- `src/utils/serializer.ts` and `deserializer.ts` are the compatibility boundary with `hydra.conf`.
- `src/utils/validation.ts` must stay aligned with .NET config validation.
- `src/hooks/useHydraConfig.ts` owns editor state transitions.
- `src/components/` should remain presentation-focused; put reusable transforms in hooks/utilities.

When adding or changing a config field, update the TypeScript model, defaults, serialization/deserialization, validation, focused tests, and `../docs/CONFIGURATION.md`. Confirm the .NET loader accepts the generated JSON.

## Commands

Use the committed npm lockfile and Node 22-compatible tooling:

```bash
npm ci
npm test
npm run lint
npm run build
```

Run commands from `HydraWebConfig/`. Do not substitute another package manager or rewrite `package-lock.json` without an intentional dependency change.

## Implementation rules

- Keep strict TypeScript types; do not use `any` to bypass config modeling.
- Preserve round-trip behavior: deserialize then serialize without dropping supported fields.
- Keep generated configuration deterministic so diffs remain readable.
- Add or update Vitest/Testing Library coverage in `src/__tests__/` for behavior changes.
- Prefer accessible labels/roles and user-level interaction tests over implementation-detail assertions.
- Do not commit `dist/`, local environment files, or generated assets.
- React Compiler is intentionally disabled; do not enable it as incidental cleanup.
