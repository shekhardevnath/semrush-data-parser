# Repository Guidelines

## Project Structure & Module Organization
This repository is a Vue 3 app built with Vite.

- `src/`: application code.
- `src/main.js`: app bootstrap and global CSS import.
- `src/App.vue`: root component.
- `src/components/`: reusable Vue single-file components (SFCs), including `icons/`.
- `src/assets/`: global styles and static assets used by components.
- `public/`: static files served as-is (for example `public/favicon.ico`).
- Root config files: `vite.config.js`, `jsconfig.json`, `package.json`.

Keep new feature code under `src/` and colocate component-specific logic/styles inside the same `.vue` file when practical.

## Build, Test, and Development Commands
Use npm scripts defined in `package.json`:

- `npm install`: install dependencies.
- `npm run dev`: start local dev server with hot reload.
- `npm run build`: create production build in `dist/`.
- `npm run preview`: preview the production build locally.

Example workflow: `npm install && npm run dev`.

## Coding Style & Naming Conventions
- Follow existing Vue SFC style with `<script setup>`, `<template>`, and scoped styles when needed.
- Use 2-space indentation in `.vue`, `.js`, and `.css` files.
- Component files use PascalCase (for example `HelloWorld.vue`, `TheWelcome.vue`).
- JS variables/functions use camelCase.
- Prefer clear, small components over large monolithic templates.

No formatter or linter is currently configured. Keep style consistent with surrounding code, and avoid introducing a new style pattern in isolated files.

## Testing Guidelines
There is no test framework configured yet (`test` script is not present). For now:

- Validate changes with `npm run build` before opening a PR.
- Manually verify key flows in `npm run dev`.
- If you add non-trivial logic, include a note in the PR describing how it was validated.

If tests are added later, place them under `src/` (for unit scope) or a dedicated `tests/` directory.

## Commit & Pull Request Guidelines
Git history is not available in this workspace snapshot, so no existing commit convention could be verified. Use clear, imperative commit messages, e.g.:

- `feat: add SEMrush CSV parser view`
- `fix: handle empty keyword rows`

For pull requests, include:

- concise summary of changes,
- linked issue/task (if available),
- validation steps (`npm run build`, manual checks),
- screenshots for UI changes.
