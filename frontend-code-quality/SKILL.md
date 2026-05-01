---
name: frontend-code-quality
description: "Frontend code quality rules for React/Next.js: component organization, sizing, styling with cn(), route-scoped utilities, and server actions. Use when writing, reviewing, or refactoring frontend components, organizing route files, applying className patterns, or auditing code against project conventions."
allowed-tools: Read, Grep, Glob, Bash, Task, Edit, Write
---

# Frontend Code Quality Rules

## Rules

### Component Organization

- If a component is only relevant to one page/route, place it in `app/<route>/_components/`.
- If a sub-component is only used by its parent or a related sibling, create a folder for it:
  ```
  _components/
    MyComponent/
      index.tsx
  ```

### Component Size

- Keep components small and modular. Break large components into a composed tree.
- Exception: Skip this if splitting would only create unnecessary prop-drilling. A slightly bigger component beats a chain of pass-through props.

### Styling

- Never use template literals in classNames. Use the `cn()` utility.
- For conditional classes, use object syntax:
  ```tsx
  // Bad
  className={`text-sm ${isActive ? "bg-blue-500" : ""}`}

  // Good
  className={cn("text-sm", { "bg-blue-500": isActive })}
  ```
- Never create custom CSS classes. Use Tailwind's utility classes directly. If you need reusability, extract a component instead of creating a shared class.

### String Escaping in JSX

- When text contains characters that would need HTML entities or escaping, use the `{``}` template literal format instead:
  ```tsx
  // Bad
  <p>Don&apos;t use &quot;HTML entities&quot;</p>

  // Good
  <p>{`Don't use "HTML entities"`}</p>
  ```

### Utilities

- Route-scoped utility functions go in `app/<route>/_lib/`.

### Server Actions

- All server actions live in `src/lib/actions/`.
- One file per action.

---

## Audit Workflow

When asked to audit or fix frontend code quality, follow these steps:

### 1. Ensure linting script

- Check `package.json` for a `lint` script.
- If missing, add one. For Next.js projects: `"lint": "next lint --max-warnings 0"`. For non-Next projects, use the appropriate eslint command with `--max-warnings 0`.
- If a lint script exists but does NOT include `--max-warnings 0`, append it. Zero warnings tolerance is mandatory.
- Run the lint script to verify it passes. Fix any warnings or errors that surface.

### 2. Ensure formatter setup

- Check if the project already has a formatter configured (look for `.prettierrc`, `.prettierrc.json`, `.prettierrc.js`, `prettier.config.*`, or a `prettier` key in `package.json`).
- If **no formatter is configured**, set up Prettier:
  1. Install dependencies:
     ```bash
     npm install -D prettier @ianvs/prettier-plugin-sort-imports prettier-plugin-tailwindcss prettier-plugin-packagejson
     ```
  2. Create `.prettierrc` with:
     ```json
     {
       "semi": false,
       "plugins": [
         "@ianvs/prettier-plugin-sort-imports",
         "prettier-plugin-tailwindcss",
         "prettier-plugin-packagejson"
       ]
     }
     ```
- Check `package.json` for `format:write` and `format:check` scripts. Add them if missing:
  ```json
  {
    "scripts": {
      "format:write": "prettier --write .",
      "format:check": "prettier --check ."
    }
  }
  ```
- Run `npm run format:check` to verify the setup works. If there are unformatted files, run `npm run format:write` and note the changes.

### 3. Discover project structure

- Glob for `app/**/page.tsx` to identify routes.
- Glob for `app/**/_components/**` and `src/components/**` to map component locations.
- Glob for `src/lib/actions/**` to find existing server actions.

### 4. Check className usage

- Grep for template literals in className attributes: patterns like `` className={` `` or `className={\"` with `${}` interpolation.
- For each violation, replace with `cn()` using object syntax for conditionals.
- Verify the file imports `cn` from the project's utility path (typically `@/lib/utils` or similar). Add the import if missing.

### 5. Check component placement

- Identify components defined in `src/components/` that are only imported by a single route. These should be moved to that route's `_components/` directory.
- Use Grep to count imports of each component across the codebase. If a component under `src/components/` has exactly one consumer inside `app/<route>/`, flag it for relocation.

### 6. Check component size

- Read through components in `_components/` directories. Flag any component file exceeding ~150 lines of JSX as a candidate for decomposition.
- Do NOT split if the only way to pass data to the child is prop-drilling through multiple layers. Note the exception and move on.

### 7. Check server actions

- Grep for `"use server"` directives. Any server action not located in `src/lib/actions/` should be moved there.
- Ensure one action per file. If a file exports multiple server actions, split them.

### 8. Check utility placement

- Look for utility/helper functions defined inline in route page files or components. If they are route-scoped, move them to `app/<route>/_lib/`.

### 9. Report and fix

- Fix all violations directly. For each change, briefly note what was wrong and what you did.
- After all fixes, run the project's type-check and lint commands (e.g., `npx tsc --noEmit`, `npx next lint`) to confirm nothing broke.
