---
description: "Detects project tech stack and conventions. Used at review start to configure agents appropriately."
---

# Stack Detection Skill

Automatically detect project configuration to tailor review to existing conventions.

## Detection Process

### 1. Package Manager

```bash
# Check for lock files
ls package-lock.json  # npm
ls yarn.lock          # yarn
ls pnpm-lock.yaml     # pnpm
ls bun.lockb          # bun
```

### 2. Framework Detection

Check `package.json` dependencies:

| Dependency | Framework |
|------------|-----------|
| `react` | React |
| `next` | Next.js |
| `vue` | Vue |
| `@angular/core` | Angular |
| `svelte` | Svelte |
| `express` | Express.js |
| `fastify` | Fastify |
| `nestjs` | NestJS |

### 3. TypeScript Configuration

Check for `tsconfig.json`:
- Strict mode enabled?
- Path aliases configured?
- Target ES version?

### 4. Linter/Formatter

| File | Tool |
|------|------|
| `.eslintrc*` | ESLint |
| `biome.json` | Biome |
| `.prettierrc*` | Prettier |
| `deno.json` | Deno |

### 5. Testing Framework

| Dependency/File | Framework |
|-----------------|-----------|
| `jest` | Jest |
| `vitest` | Vitest |
| `@testing-library/*` | Testing Library |
| `cypress` | Cypress |
| `playwright` | Playwright |

### 6. Code Conventions

Analyze existing code for patterns:

**Import Style:**
```typescript
// Relative
import { X } from '../utils';
// Alias
import { X } from '@/utils';
// Absolute
import { X } from 'src/utils';
```

**Component Style:**
```tsx
// Arrow function
const Component = () => {};
// Function declaration
function Component() {}
// Default export
export default function Component() {}
```

**Naming Conventions:**
- Files: `PascalCase.tsx` vs `kebab-case.tsx` vs `camelCase.tsx`
- Components: `PascalCase`
- Hooks: `useCamelCase`
- Utils: `camelCase`

## Output Format

```yaml
Stack:
  package_manager: npm | yarn | pnpm | bun
  framework: React | Next.js | Vue | Angular | Node
  typescript: true | false
  strict_mode: true | false
  path_aliases: ["@/*", "~/"]

Testing:
  framework: Jest | Vitest | none
  coverage: true | false

Linting:
  eslint: true | false
  prettier: true | false
  biome: true | false

Conventions:
  import_style: relative | alias | absolute
  component_style: arrow | declaration
  file_naming: PascalCase | kebab-case | camelCase
  export_style: named | default

Commands:
  test: "npm test"
  typecheck: "tsc --noEmit"
  lint: "eslint ."
  format: "prettier --write ."
```

## Agent Configuration

Based on detected stack, configure agents:

### For React/Next.js:
- Enable React-specific patterns in all agents
- Check for hooks rules violations
- Verify component composition patterns

### For TypeScript:
- Enforce type safety in all suggestions
- Check for `any` usage reduction
- Verify generic constraints

### For Strict Projects:
- Higher standards for all agents
- More aggressive simplification
- Stricter consistency requirements
