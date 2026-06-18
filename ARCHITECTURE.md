# Architecture

## System Overview

Firelink Frontend is a single-page application that provides a web GUI for [Bonfire][bonfire], an OpenShift namespace management tool used by Red Hat's engineering productivity teams. The application enables developers to manage ephemeral environment namespaces on an OpenShift cluster: reserving and releasing namespaces, browsing and deploying applications, monitoring cluster and namespace resource usage, and saving deployment configurations as reusable recipes.

The frontend communicates with a separate Firelink backend API server over HTTP REST endpoints and WebSocket connections. It is deployed on OpenShift using the [Frontend Operator][frontend-operator] and served by a Caddy reverse proxy.

### Firelink Ecosystem

```
Browser  -->  OpenShift Route (:8888 TLS)
                |
          OAuth Proxy sidecar (:8888)          ── firelink-proxy pod ──
                |                              |                      |
          Caddy reverse proxy (:8000)          |  two containers in   |
              /   \                            |  a single pod        |
             /     \                           ── (sidecar pattern) ──
            v       v
  /api/*  -->  firelink-backend service  -->  OpenShift / Bonfire
  /*      -->  firelink-frontend service      (this repo, Caddy serving static files)
                        |
                   React SPA
```

The frontend is a purely client-side application. Its own Caddy instance serves the built static
assets only. In production, [firelink-proxy][firelink-proxy] is a single pod with two containers
(sidecar pattern): an OpenShift OAuth Proxy listening on port 8888 (HTTPS) and a Caddy reverse
proxy listening on port 8000 (HTTP). The OAuth Proxy handles authentication and forwards to Caddy
over localhost. Caddy routes `/api/*` to the backend service and all other requests to this
frontend service (it does not serve static files — it reverse-proxies to the frontend's own pod).
See the [firelink-proxy][firelink-proxy] repository for full proxy configuration details.

During local development, the backend's Caddy dev proxy (`make run-proxy` on port 8000) handles
the same routing: `/api/*` to the Flask backend on port 5001, everything else to the React dev
server on port 3000. `src/setupProxy.js` injects a mock `gap-auth` header to simulate the OAuth
proxy's user identity injection. The backend wraps Bonfire library functions and the Kubernetes
Python client, exposing them as REST and WebSocket endpoints.

## Technology Stack

- **UI framework**: React (functional components with hooks)
- **Design system**: PatternFly (Red Hat's open source design system)
- **State management**: Redux Toolkit with redux-persist
- **Routing**: React Router (BrowserRouter, nested routes)
- **Real-time communication**: socket.io-client (WebSocket over polling transport)
- **Animations**: react-spring (fade and slide transitions)
- **Build tooling**: Create React App (react-scripts)
- **Reverse proxy**: Caddy
- **Container runtime**: Node.js >= 22.0.0 (build stage), Caddy UBI (serve stage)
- **CI/CD**: Tekton pipelines via Konflux

## Module Structure

```
src/
  index.js              # Entry point: Redux Provider, PersistGate, BrowserRouter, route definitions
  App.js                # Shell layout: Masthead, sidebar navigation, dark mode, user identity
  Root.js               # Landing page with "Manage / Deploy / Develop" feature cards
  App.css               # Global CSS overrides (logo theming, page height)
  setupProxy.js         # Dev-only middleware: injects gap-auth header for local authentication

  store/                # Redux state management
  namespaces/           # Namespace lifecycle views (list, describe, reserve)
  apps/                 # App catalog views (browse, favorites)
  deploy/               # Deployment wizard and execution
  recipes/              # Saved deployment recipe management
  cluster/              # Cluster-wide metrics and node status
  shared/               # Reusable UI components
```

### `store/` -- State Management

Contains the Redux store configuration and four slices:

| File | Slice name | Purpose | Persisted |
|------|-----------|---------|-----------|
| `Store.js` | -- | Configures the store, combines reducers, sets up redux-persist | -- |
| `AppSlice.js` | `appSlice` | User identity (requester), dark mode preference, favorite apps, saved deploy recipes | Yes |
| `ListSlice.js` | `listSlice` | Namespace list, app list, namespace resource metrics, top pods data | No |
| `AppDeploySlice.js` | `appDeploySlice` | All deployment options (apps, namespace, duration, pool, image tags, parameters, etc.) | No |
| `ParamSelectorSlice.js` | `paramSelectorSlice` | Template parameter tree-view options and selected parameter overrides | No |

Only `appSlice` is persisted to `localStorage` (via redux-persist with `autoMergeLevel2` reconciliation). The other three slices are blacklisted from persistence since they hold transient server-fetched or session-scoped data.

### `namespaces/` -- Namespace Management

Handles the ephemeral namespace lifecycle:

- **`NamespaceList.js`** -- Fetches and displays all namespaces in a filterable table. Supports "My Reservations" toggle and namespace release.
- **`NamespaceListTable.js`** -- Renders the namespace table with column filters (reserved, status, requester, pool type), resource usage mini-bars, expiration timers, and per-row actions (extend, release).
- **`NamespaceDescribe.js`** -- Detail view for a single namespace. Takes a namespace name from the URL parameter or text input and shows description, resource usage, and pod metrics.
- **`NamespaceDescribeCard.js`** -- Fetches and renders namespace description data (requester, logins, routes, project URL) from the backend.
- **`NamespaceReserve.js`** -- Form to reserve a new namespace with pool, duration, and force (multi-reservation) options.
- **`NamespaceResourcesCard.js`** -- Displays CPU and memory progress bars for a namespace.
- **`ResourceUsageMini.js`** -- Compact progress bar for CPU/memory usage, reads from the Redux store.
- **`ResourceUsageProgress.js`** -- Self-fetching progress bar that polls `/api/firelink/namespace/resource_metrics/{namespace}` every 10 seconds.
- **`TopPodsCard.js`** -- Sortable, filterable table of pod resource consumption within a namespace, with links to the OpenShift console. Auto-refreshes every 10 seconds.

### `apps/` -- App Catalog

- **`AppList.js`** -- Gallery view of all deployable applications. Supports text filtering and a favorites toggle.
- **`AppListItem.js`** -- Card component for a single app with a generated gradient icon, favorite star toggle, links to the Developer Portal and resource templates, and a "Deploy" button that navigates to the deploy wizard.
- **`AppDeploy.js`** -- (Dead code) Three-column deploy page from an older implementation. Imports a non-existent `AppDeployControllerCard` component and would crash if rendered. No route points to this file — the primary deploy flow lives in `deploy/AppDeploy.js`.

### `deploy/` -- Deployment Wizard

The deployment workflow is implemented as a multi-step PatternFly Wizard:

- **`AppDeploy.js`** -- Wizard container with steps: Apps, Namespace, Options, Preserve Resources, Omit Dependencies, Set Parameters, Image Tag Overrides, Review & Deploy. "Advanced Options" toggle shows/hides the advanced steps.
- **`AppMenuCard.js`** -- Searchable, favoritable app selection menu with quick-deploy capability.
- **`AppDeployNamespaceSelector.js`** -- Choose between reserving a new namespace or using an existing reservation.
- **`AppDeployOptionsCard.js`** -- Deployment option toggles: frontends, release-on-fail, dependencies, single replicas, pool, duration, environments, optional deps method.
- **`ResourceSelector.js`** -- Dual-list selector for choosing which ClowdApps/ResourceTemplates to preserve resources for or omit dependencies from.
- **`SetParameters.js`** -- Tree-view parameter browser that lets users select and override template parameters per component.
- **`ParamInput.js`** -- Individual parameter value input field.
- **`ImageTagOverrides.js`** -- Dynamic list of image tag override inputs (`quay.io/org/repo=tag` format).
- **`ImageTagInput.js`** -- Single image tag override input with validation.
- **`AppDeployReview.js`** -- Summary view of all deployment options with deploy and save-as-recipe buttons.
- **`AppDeployModal.js`** -- Triggers the deployment via WebSocket and displays real-time progress in a modal dialog.
- **`AppDeploySaveRecipeModal.js`** -- Saves current deployment options as a named recipe to the Redux store.
- **`AppDeployRemoveSelector.js`** -- (Dead code) Tree-based dual-list selector for choosing resources to remove. Superseded by `ResourceSelector.js` and not imported anywhere.

### `recipes/` -- Deployment Recipes

- **`Recipes.js`** -- Recipe management page. Lists saved recipes, allows viewing/deploying/downloading/deleting/uploading recipes. Recipes are JSON files with a validated schema containing deployment options.
- **`RecipeList.js`** -- Dropdown selector for choosing among saved recipes.

### `cluster/` -- Cluster Metrics

- **`ClusterCard.js`** -- Dashboard showing cluster-wide CPU usage, memory usage, and per-node metrics. Links to the OpenShift monitoring dashboard.
- **`ClusterResourceUsage.js`** -- Progress bar for cluster-level CPU or RAM usage.
- **`ClusterResourceUsageMini.js`** -- Compact progress bar with tooltip showing usage/capacity/percentage for a single node metric.
- **`ClusterTopNodes.js`** -- Table of cluster nodes with CPU, memory, and storage usage bars, linked to the OpenShift console.

### `shared/` -- Reusable Components

- **`Loading.js`** -- Centered spinner card with an optional loading message.
- **`ErrorCard.js`** -- Error display card with retry button.
- **`FadeInFadeOut.js`** -- react-spring fade-in animation wrapper.
- **`SlideInSlideOut.js`** -- react-spring slide-in animation wrapper.
- **`FilterDropdown.js`** -- Dynamic column filter dropdown that extracts unique values from a data array.
- **`CustomSelects.js`** -- Reusable select list components for pool, duration, optional deps method, reference environment, and target environment options.
- **`DescribeLink.js`** -- React Router `Link` to the namespace describe page.
- **`HelpTip.js`** -- Question-mark icon with a tooltip for contextual help.
- **`SelectedAppsChips.js`** -- Label group showing currently selected apps as chips.
- **`RecipieViewer.js`** -- Read-only summary view of a deploy recipe's options, apps, namespace, resources, dependencies, parameters, and image tag overrides.

## Routing Architecture

All routes are defined in `src/index.js` using React Router's `BrowserRouter`. The `App` component serves as the layout wrapper (masthead + sidebar), and all page components render into its `<Outlet />`.

| Path | Component | Description |
|------|-----------|-------------|
| `/` | `Root` | Landing page |
| `/namespace/list` | `NamespaceList` | Namespace table |
| `/namespace/describe/:namespaceParam` | `NamespaceDescribe` | Namespace detail (with param) |
| `/namespace/describe` | `NamespaceDescribe` | Namespace detail (manual input) |
| `/namespace/reserve` | `NamespaceReserve` | Reserve a namespace |
| `/apps/list` | `AppList` | App catalog gallery |
| `/apps/deploy/:appParam` | `AppDeploy` | Deploy wizard (pre-selected app) |
| `/apps/deploy` | `AppDeploy` | Deploy wizard |
| `/recipes` | `Recipes` | Saved recipe management |
| `/cluster` | `ClusterCard` | Cluster metrics dashboard |

Caddy is configured with `try_files {path} /index.html` to support client-side routing -- all non-file requests fall through to the SPA.

## API Communication Patterns

The frontend communicates with the Firelink backend through two mechanisms:

### REST API

All REST calls go through the browser's `fetch` API to relative paths under `/api/firelink/`. In production, the proxy's Caddy reverse-proxies these to the backend service. Key endpoints:

| Method | Endpoint | Used by |
|--------|----------|---------|
| GET | `/api/firelink/namespace/list` | `ListSlice.loadNamespaces` |
| POST | `/api/firelink/namespace/reserve` | `ListSlice.reserveNamespace` |
| POST | `/api/firelink/namespace/release` | `NamespaceList.releaseNamespace` |
| GET | `/api/firelink/namespace/describe/{ns}` | `NamespaceDescribeCard` |
| GET | `/api/firelink/namespace/resource_metrics` | `ListSlice.loadNamespaceResources` |
| GET | `/api/firelink/namespace/resource_metrics/{ns}` | `ResourceUsageProgress` |
| POST | `/api/firelink/namespace/top_pods` | `ListSlice.loadNamespaceTopPods` |
| GET | `/api/firelink/apps/list` | `ListSlice.loadApps` |
| GET | `/api/firelink/cluster/top_nodes` | `ClusterCard` |
| GET | `/api/firelink/cluster/cpu_usage` | `ClusterCard` |
| GET | `/api/firelink/cluster/memory_usage` | `ClusterCard` |

Async thunks in `ListSlice.js` handle namespace, app, and resource data fetching with Redux Toolkit's `createAsyncThunk`, providing built-in pending/fulfilled/rejected state tracking. Some components (like `ClusterCard` and `ResourceUsageProgress`) fetch data directly with `fetch` rather than going through the store, particularly when the data is component-local or needs polling.

### WebSocket (socket.io)

WebSocket communication is used exclusively for the deployment flow (`AppDeployModal.js`). The client connects to the backend's socket.io endpoint at `/api/firelink/socket.io` using polling transport, emits a `deploy-app` event with the deployment options payload, and listens for:

- `monitor-deploy-app` -- Progress updates during deployment
- `error-deploy-app` -- Deployment errors
- `end-deploy-app` -- Deployment completion

The connection is established on-demand when the user clicks "Deploy" and includes a 60-second timeout for server responsiveness. The WebSocket URL is constructed dynamically from `window.location` to match the current protocol and host.

## Authentication

Authentication is handled externally by the [firelink-proxy][firelink-proxy], which runs an
OpenShift OAuth Proxy. The proxy's `-pass-user-headers=true` flag causes it to inject a `gap-auth`
HTTP header containing the user's email address (a legacy header name from oauth2-proxy's origin as
the Google Auth Proxy). The frontend reads this header by fetching `/index.html` and extracting the
`gap-auth` response header value, splitting on `@` to get the username. This username becomes the
`requester` identity stored in the Redux `appSlice` (defaulting to `"firelink-user"` if the header
is absent).

The backend has no authentication logic of its own — it trusts whatever `requester` value the
frontend sends in API request payloads. All access control relies on the OAuth proxy layer in front
of the application.

During local development, `src/setupProxy.js` injects a mock `gap-auth` header
(`addrew@localhost`) into all responses so the authentication flow works without the proxy. Note
that `setupProxy.js` only sets this header — it does not proxy API requests. API routing during
development is handled by the backend's Caddy dev proxy (`make run-proxy`).

## State Management

The Redux store uses four slices combined with `combineReducers`. Only the `appSlice` is persisted to `localStorage` via redux-persist, preserving user preferences (dark mode, favorites) and saved deploy recipes across sessions.

The deployment workflow stores all options in `appDeploySlice`, which maps directly to the Bonfire parameter structure. The `getDeploymentOptions` selector in `AppDeploySlice.js` assembles the full options object sent to the backend during deployment.

State flow for a typical deployment:

1. User selects apps -- `addOrRemoveApp` / `addOrRemoveAppName` actions update `appDeploySlice`
2. User configures options -- individual setter actions update `appDeploySlice` fields
3. User clicks Deploy -- `getDeploymentOptions` selector reads the full state
4. `AppDeployModal` emits the options via WebSocket
5. Backend streams progress back via socket events

## Build and Deployment Pipeline

### Multi-Stage Docker Build

The `Dockerfile` defines a two-stage build:

1. **Build stage** (`registry.access.redhat.com/ubi9/nodejs-22`) -- Installs npm dependencies, runs `npm run build` to produce optimized static assets in `/app/build`.
2. **Serve stage** (`caddy-ubi`) -- Copies the Caddyfile and built assets into the Caddy image. Serves on port 8000.

### Caddy Configuration

`config/Caddyfile` configures the frontend's own Caddy instance to:

- Serve static files from `/opt/app-root/src/build`
- Fall through to `index.html` for client-side routing (`try_files`)
- Expose Prometheus metrics via the `metrics` server directive
- Listen on port 8000

This Caddy instance serves only the frontend's static assets. API routing and authentication are handled by the separate [firelink-proxy][firelink-proxy] component.

### OpenShift Deployment

Three OpenShift templates in `deploy/` define the deployment:

- **`deploy.yaml`** -- Standard Kubernetes Deployment and Service. Runs the container on port 8000 with parameterized image, tag, and service name.
- **`frontend.yaml`** -- Frontend Operator `Frontend` custom resource. Declares the app's API versions, paths, and module configuration.
- **`frontend-environment.yaml`** -- `FrontendEnvironment` custom resource. Configures the hostname, SSO endpoint, SSL, and ingress class for the Firelink instance.

### CI/CD with Tekton

Tekton pipelines in `.tekton/` automate the build on push to `master`:

1. `init` -- Pipeline initialization
2. `clone-repository` -- Git clone
3. `prefetch-dependencies` -- Optional dependency prefetching
4. `build-container` -- Buildah container image build
5. `build-image-index` -- OCI image index creation
6. Security scans: Clair vulnerability scan, Snyk SAST, ClamAV, ecosystem cert preflight checks, shell check, unicode check, RPM signature scan
7. `push-dockerfile` -- Publishes the Dockerfile as an artifact
8. `apply-tags` -- Tags the built image

Images are pushed to `quay.io/redhat-user-workloads/hcm-eng-prod-tenant/firelink/firelink-frontend`.

## Key Design Decisions

- **No server-side rendering**: The app is a purely client-side SPA served as static files by Caddy. This simplifies deployment but means the initial load requires downloading the full JavaScript bundle.
- **Polling for live data**: Resource usage components (`ResourceUsageProgress`, `TopPodsCard`) use `setInterval`-based polling (10-second intervals) rather than WebSocket subscriptions. WebSocket is reserved for the deployment flow where streaming progress updates are essential.
- **Selective state persistence**: Only user preferences and saved recipes persist across sessions. Transient data (namespace lists, app catalogs, deployment options) is fetched fresh, avoiding stale data issues.
- **External authentication**: The frontend delegates authentication entirely to an upstream proxy, reading identity from HTTP headers. This avoids implementing a login flow but requires the proxy infrastructure to be in place.
- **Dual deploy interfaces**: Two `AppDeploy.js` files exist -- the legacy three-column layout (`src/apps/AppDeploy.js`, dead code with a broken import) and the current wizard-based flow (`src/deploy/AppDeploy.js`). Only the wizard is routed; the legacy file is unreachable.
- **PatternFly theming**: Dark mode is implemented by toggling the `pf-v6-theme-dark` CSS class on the document root, with custom CSS in `App.css` handling logo color adjustments for light mode.

[bonfire]: https://github.com/RedHatInsights/bonfire
[firelink-proxy]: https://github.com/RedHatInsights/firelink-proxy
[frontend-operator]: https://github.com/RedHatInsights/frontend-operator
