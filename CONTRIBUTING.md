# Contributing to Open Work

Thanks for your interest in contributing! Open Work is a Next.js (App Router) +
[shadcn/ui](https://ui.shadcn.com) app that drives a real worker agent — Claude
Code or Codex — inside an isolated [Hiver](https://hiver.sh) sandbox. This guide
covers how to set up your environment, make changes, and submit them.

For the Hiver runtime itself, see the main Hiver repository's contributing guide.

## Prerequisites

- **Node.js 20.9+**, required by Next.js 16.
- **Docker**, which the local Hiver stack runs on.
- **Hiver CLI**:

  ```sh
  npm install -g @hiver.sh/cli
  ```

  Then bring up the gateway the app talks to, on `http://localhost:10000` by
  default (configurable in Settings):

  ```sh
  hiver up      # hiver down to stop it
  ```

- **At least one provider API key** (Anthropic and/or OpenAI), entered once in
  Settings. Keys are stored in your browser's local storage and never leave it
  except via the egress override described in the [README](README.md).

## Running it

```sh
npm install
npm run dev          # http://localhost:3000
```

The dev server expects a gateway to already be up (`hiver up`). Each task
provisions its own sandbox on demand, so you don't need to start one yourself.

## Project layout

| Path | Description |
| --- | --- |
| [`app/`](app/) | App Router entry: `layout.tsx`, the single client `page.tsx`, and `globals.css`. |
| [`app/api/`](app/api/) | Route handlers — `stream` (the one SSE endpoint every task's events flow through), `tasks` (POST a turn to a task's agent), plus `task`, `conversation`, `sandbox`, `file`, `files`, and `browser/{screen,input}`. |
| [`components/`](components/) | App components (conversation, composer, task sidebar, settings, viewers). |
| [`components/ui/`](components/ui/) | shadcn/ui primitives. Generated — prefer `npx shadcn@latest add <component>` over hand-writing these. |
| [`lib/`](lib/) | The engine room. See below. |
| [`lib/providers/`](lib/providers/) | Per-provider glue (`claude.ts`, `codex.ts`) behind a shared interface. |

The parts worth understanding before you change anything:

- [`lib/hiver.ts`](lib/hiver.ts) — sandbox provisioning: one sandbox per task
  (keyed `<taskId>-work`), the egress rules that inject provider API keys without
  ever exposing them to the agent, and the nested per-task browser sandbox.
- [`lib/session.ts`](lib/session.ts) — the persistent agent process: pumps
  `stream-json` events into SSE, watches the sandbox event stream for file
  writes, egress denials, and nested sandboxes, and resumes rather than restarts
  if the process dies.
- [`lib/browser.ts`](lib/browser.ts) — a minimal CDP client: page discovery,
  screencast streaming, input dispatch, clipboard bridging.
- [`lib/orchestration.ts`](lib/orchestration.ts) — the model list, CLI model ids,
  and default model. Adding a model usually means editing only this file.

## Verifying your change

There is no test suite in this repo yet, so the bar is that the app builds clean
and the flow you touched actually works against a live gateway:

```sh
npm run typecheck    # tsc --noEmit
npm run build        # also type-checks, and catches App Router / RSC errors
```

Then exercise it end to end — `hiver up`, `npm run dev`, and drive the path you
changed. A green build says very little about this app; most of its behavior only
shows up against a real sandbox:

- **Changed streaming, sessions, or the agent process?** Send a task, then
  refresh the page mid-stream. The conversation is rebuilt from the sandbox's own
  transcript, so a regression here can look fine until you reload.
- **Changed sandbox, egress, or provider wiring?** Confirm the API key still
  never lands in the agent's environment — it's applied by the proxy via an
  egress override, and keeping it that way is the point.
- **Changed the browser viewer?** Run a task that uses the browser skill and
  confirm frames stream in and input dispatches back out.

If you fix a bug, say in the PR how you reproduced it and how you confirmed it's
gone.

> **Note:** the `lint` script in `package.json` is currently broken — it runs
> `next lint`, which was removed in Next.js 16, and there's no ESLint config in
> the repo. Don't rely on it; `npm run typecheck` is the check that works today.

## Code style

- **TypeScript throughout**, with the `@/*` path alias for imports.
- **Tailwind + shadcn/ui.** Styling goes through Tailwind classes and the theme
  tokens in `app/globals.css`; the shadcn config lives in
  [`components.json`](components.json) (new-york style, zinc base, `lucide`
  icons). Add primitives with `npx shadcn@latest add <component>` rather than
  writing them by hand.
- **Light and dark mode both matter.** Theme is driven by `next-themes` and CSS
  variables — check any UI change in both.
- Match the surrounding code's naming and comment density. Comment the parts that
  are Hiver- or agent-specific, not the parts that are ordinary React.

## Submitting changes

1. Fork the repository and create a topic branch off `main`.
2. Make your change, keeping commits focused and descriptive.
3. Run `npm run typecheck` and `npm run build`, and exercise the flow you touched
   against a live gateway (see [Verifying your change](#verifying-your-change)).
4. Check the UI in both light and dark mode if you touched anything visual.
5. Push your branch and open a pull request against `main`.
6. Describe what changed and why. Link any related issues.

Please open an issue first for large or breaking changes so we can discuss the
approach before you invest significant effort.

## License

By contributing, you agree that your contributions will be licensed under the
project's [Apache 2.0](LICENSE) license.
