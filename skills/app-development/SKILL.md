---
name: app-development
description: Builds, changes and deploys a Marcapony app, a small web app with its own backend that Marcapony hosts and connects to the business's account, using create-marcapony-app, the marcapony CLI and marcapony-sdk. Covers scaffolding from a template (blank, booking, shop, team tool), linking the project to the app, calling the Marcapony API from the app's server, signed-in customers and team members, test mode and deploying. Use when the user wants a custom tool, portal page, booking flow, shop flow or internal team tool on Marcapony that a section cannot do, or works in a project that has a marcapony.json, including when its API key or .dev.vars seems wrong. Load it before opening any file in such a project: .dev.vars holds the app's secret key and must never be opened. Needs a local project folder and a terminal, so it applies in Claude Code, not in a chat without file access.
---

# Building a Marcapony app

A Marcapony app is a small web app (a backend plus optional static assets) that the business registers under **My Apps** in the Marcapony dashboard and Marcapony hosts. It runs on the business's site, behind the customer portal login, or in the dashboard for the team, and it reaches the account's data through the Marcapony external API with a key Marcapony supplies at runtime. There is nothing to host or provision.

Use an app when the work needs its own backend logic, routes or signed-in users. For a visual block on a page, a custom code section is the right tool (see the custom-code skill).

The packages are `create-marcapony-app` (scaffolder), `marcapony` (CLI) and `marcapony-sdk` (typed API client). Their READMEs, and each template's `README.md`, are the reference; this skill is the order of work.

## 1. Start or pick up the project

- **Existing project**: every template ships a `marcapony.json`, so that file alone does not mean the project is linked. If there is no `AGENTS.md`, or `marcapony.json` has no `app` field, the project is not linked yet: go to step 2 first. Otherwise read `AGENTS.md` first. The CLI writes it at link time and it lists exactly which API operations the app's key may call. Then read `marcapony.json` and the template `README.md` if there is one. Do not plan features that need operations missing from `AGENTS.md`; tell the user which scope the app needs instead.
- **New project**: choose the template closest to the request and scaffold it:

  ```sh
  npx create-marcapony-app my-app --template booking
  cd my-app
  bun install
  ```

  | Template | What it is | Runs in |
  |---|---|---|
  | `blank` | A starting point that lists products | The site |
  | `booking` | Signed-in customers see, book and cancel appointments | Customer portal |
  | `shop` | Products, a cart and a checkout that sends the buyer to pay | The site |
  | `team-tool` | The team approves, rejects and completes the day's reservations | Dashboard |

  The template's `README.md` says which placement, data access, scopes and permissions to choose when the user creates the app under **My Apps**. Tell the user those settings; creating the app happens in the dashboard, not from the terminal.

## 2. Link the project

Linking exchanges a one-time pairing code from the app's page in the dashboard for a key to the app's **test** environment, writes it into `.dev.vars` (and adds that file to `.gitignore`), fills `marcapony.json` and writes `AGENTS.md`.

Ask the user to generate the pairing code and run the link step themselves, for example `bun run link -- --code mpl_…`. Never ask for the key, never open, read, grep, print or log `.dev.vars`, even to check whether the key looks right (a 401 is fixed by linking again, not by inspecting the key), never commit it, and never put the key in browser code: the SDK refuses to run with a secret key in a browser. If a key leaks, the user revokes it in the dashboard and links again.

## 3. Build

- Server code calls the API through the client the template builds in `src/server/marcapony.ts`: `marcaponyFromEnv(env)` per request, `identityOf()` for the signed-in person, `errorResponse()` for API errors. Do not call `createClient` anywhere else: the helper reads the key, the API address and the app id from the right place, and the app id is what lets `identity()` work. In a project without that file, pass `apiKey`, `baseUrl` and `appId` from `MARCAPONY_API_KEY`, `MARCAPONY_API_URL` and `MARCAPONY_APP_ID`.
- The SDK is 0.x, so a new minor version (0.7 to 0.8) may break the app. Do not move to a new minor version without reading the SDK `CHANGELOG.md`.
- **Signed-in people** (portal and dashboard apps): read who is signed in with `mp.identity(request)` and act for them through `mp.forRequest(request)`, created per request and never cached in module scope. Customers get the `me.*` calls; team members get the account groups. `identity()` is for display only; the platform enforces access on every call. Never read, log or forward the identity header or the portal cookie.
- **Money and writes**: amounts are numbers in the currency's major unit (`250` is 250.00 TRY). Pass an `idempotencyKey` to every create call: the SDK retries a POST only when it has one, and the server replays the first response for the same key. Derive it from what the call is for (an order, a booking request), so a retry never charges or books twice.
- **Errors**: every non-2xx response throws a `MarcaponyError` subclass. `tier_limit_exceeded` and `plugin_required` are permanent for the account's plan; do not retry them, tell the user what the plan or plugin needs.
- **No storage of its own**: an app may not declare storage bindings. Anything that persists lives in Marcapony through the API.
- `dataAccess` and `placement` in `marcapony.json` decide whose data the app works with. A deploy can only narrow them; widening is done in the dashboard.

## 4. Run and test

- `bun run dev` runs the app locally against the test environment. `bun run typecheck` before every deploy.
- A new app starts in **test mode**: reads return real data, writes are not saved and resolve to the input echoed back with `simulated: true` and without server-computed fields. Handle that case in the UI instead of treating it as a failure.
- Locally nobody is signed in, so portal and team flows (`identity()`, `forRequest()`) can only be tested on the deployed app, opened from the dashboard. While the owner previews, `who.preview` is true and writes are simulated.

## 5. Deploy

Ask the user before every deploy, and run `bun run typecheck` first. `bun run deploy` builds, uploads and waits until the deployment is live, then prints its address. A deploy still running when the wait ends prints its deployment id instead: give the user that id and do not deploy again until it has finished. The environment follows the key, so a test key deploys to test. If the CLI prints `Deploy failed:` the previous version keeps serving; fix the reported reason and deploy again. The owner turns test mode off in the dashboard when the app is ready for real data.

## 6. Finish

Tell the user what the app now does, which routes changed, anything they have to do in the dashboard (scopes, permissions, assigning permissions to people, turning test mode off), and the deployed address the CLI printed.
