# Deploy FAST INVESTIMENTOS on Vercel (multi-service)

## Services and routing

- `backend`: FastAPI service rooted at `backend/`, ASGI entry point `server:app` (the `app` object defined in `backend/server.py`).
- `frontend`: Vite service rooted at `frontend/`, with an SPA fallback to `index.html` for React Router deep links.
- `/api` and `/api/*` route to the backend; all other paths route to the frontend.

The FastAPI app already mounts its routers under `/api`, and the frontend API client uses relative `/api` URLs. Therefore the public path prefix is preserved and no client-side absolute backend URL is needed.

## Bindings

No service bindings are added at this stage. The frontend is a browser-delivered Vite app and calls `/api/*` through the public same-origin rewrite. Vercel service bindings are runtime environment variables for server-side service-to-service calls; they are not available to browser code or at Vite build time. The inspected backend does not make calls to a second internal application service.

If another private backend service is introduced later, declare its binding on the backend service that calls it, then read the injected environment variable at runtime.

## Required Vercel environment variables

Configure these in Vercel for the backend service (never commit actual secrets):

- `MONGO_URL`: connection string to a remotely reachable MongoDB deployment. A localhost MongoDB instance from the development container is not available to Vercel.
- `DB_NAME`: database name.
- `CORS_ORIGINS`: the deployed origin(s), comma-separated if needed. Since frontend and API share an origin, use the production site's origin unless other trusted clients need access.

Add any other credentials required by the application's email, payment, or external integrations after inspecting the corresponding feature configuration. Do not copy development `.env` secrets into source control.

## Before production

1. Confirm the service names and routing in `vercel.json`.
2. Set backend environment variables in Vercel.
3. Ensure MongoDB network access allows the Vercel backend to connect securely.
4. Deploy a preview and verify `/api/`, login, and direct navigation/refresh on `/login`, `/cadastro`, `/termos`, and `/privacidade`.
5. Verify scheduled/cron functionality separately; platform-specific cron configuration may need adaptation.
6. Run backend tests and the frontend build in an environment with the required dependencies.

This configuration has been statically reviewed against the current Vercel Services routing model, but it has not been deployed or validated with the Vercel CLI. Confirm the service build and routing in a preview deployment before production.
