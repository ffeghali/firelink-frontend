# Firelink Frontend

Frontend React SPA for the Firelink project: a web GUI for [Bonfire][bonfire]. See the
[Firelink backend][firelink-backend] for the other half of this app.

Firelink enables developers to manage ephemeral environment namespaces on an OpenShift cluster —
reserving and releasing namespaces, browsing and deploying applications, monitoring cluster and
namespace resource usage, and saving deployment configurations as reusable recipes.

## Prerequisites

- Node.js >= 22.0.0
- npm
- The [Firelink backend][firelink-backend] running locally (required for full functionality)

## Development Setup

```bash
# Install dependencies
npm install

# Start the app
npm start
```

The frontend will be served on `localhost:5000`.

To work on the app you will need the backend running as well. Clone the
[Firelink backend][firelink-backend], run the app, and then run the dev proxy that ensures the
frontend and backend can talk to each other. The backend README contains instructions on how to get
up and running.

During local development, the proxy in `src/setupProxy.js` injects a mock authentication header so
the app works without the upstream OAuth proxy.

## Building

```bash
docker build -t firelink-frontend:latest .
docker run -p 8000:8000 firelink-frontend:latest
```

The Dockerfile uses a multi-stage build: Node.js compiles the React app, then the built assets are
served by Caddy on port 8000.

## Deploying

An OpenShift template with a Frontend resource is provided in `deploy/frontend.yaml`. Deploy it to
any cluster running the [Frontend Operator][frontend-operator]:

```bash
oc process -f deploy/frontend.yaml \
  -p IMAGE="quay.io/rh_ee_addrew/firelink-frontend" \
  -p IMAGE_TAG="86263a4" \
  -p ENV_NAME="env-ephemeral-bqzepn" \
  | oc apply -n ephemeral-bqzepn -f -
```

## Architecture

For details on the internal design, module structure, API endpoints, state management, and
deployment pipeline, see the [architecture documentation][architecture].

## License

This project is licensed under the MIT License. See [LICENSE][license] for details.

[bonfire]: https://github.com/RedHatInsights/bonfire
[firelink-backend]: https://github.com/RedHatInsights/firelink-backend
[frontend-operator]: https://github.com/RedHatInsights/frontend-operator
[architecture]: ./ARCHITECTURE.md
[license]: ./LICENSE
