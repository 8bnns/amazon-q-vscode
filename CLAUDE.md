# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

This is the source for **Amazon Q for Visual Studio Code**, a VS Code extension (published as
`AmazonWebServices.amazon-q-vscode`) that connects VS Code to Amazon Q Developer (inline code
suggestions, chat, security scanning, Java upgrades, etc). It is a fork/derivative of
`aws/aws-toolkit-vscode` narrowed to just Amazon Q.

## Monorepo structure

This is an npm-workspaces TypeScript monorepo:

- `packages/core/` — almost all extension functionality (auth, CodeWhisperer/Q services, chat,
  shared utilities, webviews). Most tests live here.
- `packages/amazonq/` — the actual publishable extension. It is a thin wrapper that imports and
  activates `packages/core`; running, packaging, and most test-launching happens from this
  subproject. Import code from core via its `index.ts` exports and the `exports` map in
  `packages/core/package.json`.
- `plugins/eslint-plugin-aws-toolkits/` — a local ESLint plugin defining project-specific lint rules
  (registered in `.eslintrc.js`).
- `src.gen/` — vendored/generated service clients (`codewhisperer-streaming`,
  `amazon-q-developer-streaming-client`, `glue-catalog-client`, `sagemaker-client`), installed via
  the root `preinstall` script.
- `docs/` — architecture docs (`arch_*.md`, `CODE_GUIDELINES.md`, `TESTPLAN.md`, `web.md`,
  `telemetry.md`, `lsp.md`, `memory-bank-implementation.md`, etc). Note: `docs/arch_develop.md`
  still references a `packages/toolkit/` subproject from the upstream `aws-toolkit-vscode` repo;
  that package does not exist here — this repo only has `packages/core` and `packages/amazonq`.

You must open the project via the `amazon-q-vscode.code-workspace` workspace file (not the bare
folder) for VS Code run/debug/test launch configs to work correctly.

### `web` / `node` / `common` / `shared` convention

The extension runs both in desktop VS Code (Node.js) and in web/browser mode (vscode.dev), so
Node-only APIs (e.g. `fs`) must be isolated:

```
src/
├── myTopic/
│   ├── {file}.ts      # common code, works in any environment
│   ├── node/{file}.ts # Node.js-only code
│   └── web/{file}.ts  # web-only code
└── shared/            # general-purpose reusable utilities (common unless in node/ or web/)
```

This convention is not fully applied everywhere yet (ongoing migration); if you find "common" code
that actually depends on Node, move it into a `node/` subfolder.

### Key `packages/core/src/` areas

- `shared/` — cross-cutting utilities: `errors.ts` (`ToolkitError`), `vscode/commands2.ts`
  (`Commands` abstraction), settings, clients, telemetry, credentials, filesystem helpers.
- `auth/` — AWS/Builder ID/SSO authentication, connections.
- `codewhisperer/` — inline code suggestions engine, tracker, service clients.
- `codewhispererChat/` — chat backend (controllers, editors, storages).
- `amazonq/` — chat apps, onboarding, explorer, webview glue for the Q panel.
- `amazonqGumby/` — Java (and other language) upgrade/transform feature.
- `amazonqScan/` — security scanning feature.
- `notifications/`, `login/`, `feedback/`, `dev/` — supporting features (in-IDE notifications,
  login webview, feedback form, dev-mode/beta config).
- `webviews/` — shared webview client/server plumbing (Vue-based webviews live alongside their
  feature, e.g. `codewhisperer/vue/`).
- `web/` — package.json/README specific to the web build target.
- `test/`, `testInteg/`, `testE2E/`, `testWeb/`, `testLint/`, `testFixtures/` — see Testing below.

## Commands

Run from the repo root unless noted. Most root scripts fan out to workspaces via
`npm run <script> -w packages/ --if-present`.

```bash
npm install                 # installs deps; also runs preinstall (src.gen clients) and
                             # postinstall (builds the local eslint plugin)

npm run compile              # compile all packages (tsc + webpack) once
npm run watch                 # from within a package (e.g. packages/core) to build & watch

npm run lint                  # lint all packages (npm run lint -w packages/)
npm run lintfix                # eslint --fix across packages and plugins

npm run test                   # unit tests, all packages (npm run test -w packages/)
npm run testInteg              # integration tests
npm run testE2E                # end-to-end tests
npm run testWeb                # web-mode tests

npm run package                # build .vsix artifacts for packages/toolkit + packages/amazonq
                                 # (heap-heavy; may need: export NODE_OPTIONS=--max-old-space-size=8192)

npm run scan-licenses           # regenerate LICENSE-THIRD-PARTY / licenses-full.json
```

### Running a single test

Unit tests are Mocha-based. From `packages/amazonq` (or `packages/core`), the standard way from a
terminal:

```bash
# one file (Unix/macOS)
TEST_FILE=../core/src/test/foo.test.ts npm run test
# one directory
TEST_DIR=../core/src/test/foo npm run test
```

From VS Code (after opening via `amazon-q-vscode.code-workspace`): use the **Extension Tests
(current file)** launch config, or add `.only()` to a `describe()`/`it()`.

Test logs are written to `./.test-reports/testLog.log`. Coverage report (after running tests):
`./coverage/amazonq/lcov-report/index.html`.

### Copy-paste detection (jscpd)

```bash
npx jscpd --config .github/workflows/jscpd.json --pattern packages/…/src/foo.ts
```

## Architecture conventions

- **Commands**: use the `Commands` abstraction in `packages/core/src/shared/vscode/commands2.ts`
  (`Commands.register`, `Commands.declare`, `Commands.from`) instead of raw
  `vscode.commands.registerCommand`, for consistent logging/error-handling and testability
  (`build().asCodeLens()`, `.asTreeNode()`).
- **Errors**: use `ToolkitError` (`packages/core/src/shared/errors.ts`), not raw `Error`. Chain
  errors with `ToolkitError.chain` when adding context as they bubble up. Errors thrown from code
  invoked via `Commands` are handled centrally by `handleError` in `extension.ts`. Do not swallow
  errors, do not show users error messages directly (let `ToolkitError`/handlers do it), and use
  `CancellationError` for explicit user cancellations.
- **Globals**: use the extension's `globals` object (e.g. `globals.clock.Date`,
  `globals.clock.setTimeout`) instead of calling `Date`/`setTimeout`/other clock-related JS globals
  directly, so tests can control time.
- **Wizards & Prompters**: multi-step UI flows use the `Wizard`/`CompositeWizard` classes and
  `Prompter` (quick pick / input box) abstractions described in `docs/arch_develop.md`; test them
  with `WizardTester` / `PrompterTester` / `createQuickPickTester`.
- **Webviews**: primarily built with Vue; see `docs/arch_develop.md` ("Webviews (Vue framework)")
  for bundling/client-server structure. In debug mode, use Command Palette → **Reload Webviews** to
  pick up `.vue` changes without restarting the debug session (backend/TS changes still require a
  restart).
- **Devtools**: developer-only behavior is gated by `aws.dev.*` VS Code settings (see
  `DevSettings` in `settings.ts`); `aws.dev.forceDevMode` toggles dev mode explicitly.

## Testing conventions

- Test categories (see `docs/TESTPLAN.md`): unit (`src/test/`, fast, no mocks preferred), lint
  (`src/testLint/`), integration (`src/testInteg/`, full activated extension, no mocks), E2E
  (`src/testE2E/`), performance (`src/testInteg/perf`).
- Use `function ()` / `async function ()` for `describe()`/`it()` callbacks, not arrow functions
  (required by Mocha's `this` binding).
- Never put `await` directly inside a `describe()` block — only inside `before`/`beforeEach`/
  `after`/`afterEach`/`it`. Initialize awaited values as `let` and assign in `before()`.
- Prefer testing real code paths over mocking (e.g. assert generated CLI strings rather than
  mocking subprocess internals).
- `vscode.window` interactions in unit tests go through `getTestWindow()` (inspect
  `shownMessages`, register `onDidShowQuickPick` handlers, etc).
- Stubbing/spying on `vscode` APIs only works for code inside `packages/core`; keep such tests
  there rather than in `packages/amazonq`.

## Code style & naming (see `docs/CODE_GUIDELINES.md` for full detail)

- Enforced by Prettier (4-space indent, single quotes, no semicolons, 120 col) and ESLint —
  run `npm run lintfix` rather than hand-formatting.
- Every source file requires the Apache-2.0 license header block (enforced by the `header/header`
  ESLint rule):
  ```
  /*!
   * Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
   * SPDX-License-Identifier: Apache-2.0
   */
  ```
- Do not use "AWS" in command names/IDs (the AWS brand isn't used in some regions).
- Prefer common verbs/names for parallel concepts (`get*` over `retrieve*`, "Info" over "Details"),
  and avoid over-specific names that just describe the implementation.
- Most code about topic "Foo" belongs in `foo.ts`; don't split closely-related
  classes/interfaces/symbols across files without a clear reason.
- Avoid `lodash` — prefer native ES6/TS array/object methods.
- Use module-qualified imports (`import * as foo from '../foo'; foo.Result`) over aliased named
  imports, to avoid naming collisions.
- New custom ESLint rules go in `plugins/eslint-plugin-aws-toolkits/lib/rules`, with tests in
  `plugins/eslint-plugin-aws-toolkits/test/rules`, registered in
  `plugins/eslint-plugin-aws-toolkits/index.ts`, and enabled in `.eslintrc.js`.

## Pull requests & changelog

- PR title format (enforced by `.github/workflows/lintcommit.js`): `type: subject` where `type` is
  one of `build`, `ci`, `config`, `deps`, `docs`, `feat`, `fix`, `perf`, `refactor`, `style`,
  `telemetry`, `test`, `types` (note: `chore` is intentionally rejected), subject < 100 chars.
- PR description should be a brief **Problem** / **Solution** pair.
- Any customer-impacting change needs a changelog entry:
  `npm run newChange -w packages/amazonq` (creates a file under
  `packages/amazonq/.changes/next-release/`). Write changelog text from the user's point of view,
  not implementation details, and don't start bug-fix entries with "Fixed" (redundant).
- Treat all branches as public; `feature/x` branches are not squash-merged at release, so don't
  push anything you wouldn't want in permanent history.

## Debugging / logging

- Use `getLogger()` for logging (e.g. `getLogger().error('topic: widget failed: %O', {...})`);
  combine multiple related pieces of info into one printf-style call rather than multiple log
  calls.
- Logs appear in the VS Code **Output** panel under "Amazon Q Logs"; `aws.dev.logfile` setting can
  redirect logs to a fixed file for `tail`/`grep`.
- Service endpoint overrides for dev/testing: `aws.dev.codewhispererService`,
  `aws.dev.amazonqLsp`, `aws.dev.amazonqWorkspaceLsp` settings (or corresponding
  `__CODEWHISPERER_*` / `__AMAZONQLSP_*` / `__AMAZONQWORKSPACELSP_*` env vars).
