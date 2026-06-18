# AGENTS.md

## Project Overview

Firelink Frontend is a React single-page application that provides a web GUI for Bonfire, an
OpenShift namespace management tool. It enables developers to reserve and release ephemeral
namespaces, browse and deploy applications, monitor cluster resource usage, and save deployment
configurations as reusable recipes. The frontend communicates with a separate Firelink backend API
over REST and WebSocket, and is deployed as a container served by Caddy on OpenShift.

## Dependencies

**Runtime:** React, React Router, Redux Toolkit, redux-persist, PatternFly (React core, table,
styles), socket.io-client, react-spring, react-linkify, react-svg

**Dev/Build:** react-scripts (Create React App), http-proxy-middleware, @testing-library/react,
@testing-library/jest-dom, @testing-library/user-event, web-vitals

## Development Commands

See [Development Setup][readme-dev] in the README for the full command reference.

```bash
npm install       # Install dependencies
npm start         # Start dev server on localhost:3000
npm run build     # Production build to build/
npm test          # Run tests (Jest via react-scripts, watch mode)
```

CI runs Tekton pipelines (`.tekton/`) that build the container image and run security scans (Clair,
Snyk SAST, ClamAV, shell-check, ecosystem cert preflight). There is no separate lint or test step
in CI — the build step compiles the app, which catches syntax errors and lint violations configured
in the ESLint config.

## Architecture

The app is organized into feature modules under `src/`: `namespaces/` (namespace lifecycle),
`apps/` (app catalog), `deploy/` (deployment wizard), `recipes/` (saved deployment configs),
`cluster/` (cluster metrics), and `shared/` (reusable components). State lives in four Redux
slices under `store/`, with only user preferences persisted to localStorage. Routes are defined
in `src/index.js`; `src/App.js` provides the shell layout with navigation.

See [ARCHITECTURE.md][architecture] for detailed design decisions, API endpoint catalog, WebSocket
protocol, authentication flow, and deployment pipeline.

## Code Style

- ESLint with the `react-app` and `react-app/jest` presets (configured in `package.json` under
  `eslintConfig`)
- No explicit formatter (Prettier or similar) is configured
- Node.js >= 22.0.0
- Functional components with hooks throughout; no class components
- PatternFly components for all UI elements — custom HTML/CSS is minimal
- State mutations use Redux Toolkit's Immer-powered reducers

## Common Mistakes

- **Dual deploy components.** Two `AppDeploy.js` files exist: `src/apps/AppDeploy.js` (dead code —
  imports a non-existent component and would crash if rendered) and `src/deploy/AppDeploy.js`
  (current wizard). The wizard at `src/deploy/AppDeploy.js` is the primary deploy flow. No route
  points to the legacy file.
- **Polling vs. WebSocket confusion.** Resource usage components (`ResourceUsageProgress.js`,
  `TopPodsCard.js`) use `setInterval`-based polling (10-second cycles), not WebSocket. Only the
  deployment flow in `AppDeployModal.js` uses socket.io. Adding WebSocket subscriptions for
  resource data would require backend changes.
- **Authentication header dependency.** The app reads user identity from a `gap-auth` HTTP header
  injected by the upstream OAuth proxy (an implicit behavior of the `-pass-user-headers=true` flag).
  During local development, `src/setupProxy.js` fakes this header. If the proxy middleware is
  misconfigured or removed, the app will fall back to the default requester `"firelink-user"` and
  requests will lack a real user identity.
- **Redux persistence scope.** Only `appSlice` is persisted (dark mode, favorites, recipes). The
  other three slices (`listSlice`, `appDeploySlice`, `paramSelectorSlice`) are blacklisted. Adding
  new state to `appSlice` automatically persists it — adding transient data there will cause stale
  state bugs across sessions.

## Deployment

The app is containerized with a multi-stage Dockerfile: a Node.js build stage compiles the React
app, then a Caddy UBI image serves the static assets on port 8000. Caddy handles client-side
routing fallback (`try_files`) only — it does not proxy API requests. In production, a separate
firelink-proxy component (OpenShift OAuth Proxy + Caddy reverse proxy) handles authentication and
routes `/api/*` to the backend.

OpenShift deployment uses the Frontend Operator via templates in `deploy/`. Tekton pipelines in
`.tekton/` build and push images to Quay on every push to `master`, with security scans gating
the release.

[readme-dev]: ./README.md#development-setup
[architecture]: ./ARCHITECTURE.md
