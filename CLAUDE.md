# My Library Account

Spring Boot 3 backend (Java 21, Maven) that authenticates with Cognito, proxies the Alma
Users API, and serves a React + Material UI frontend built with Vite and pnpm.
Full specification: `PROMPT.md`. Setup and configuration: `README.md`.

## Commands

```bash
mvn clean package                      # frontend + backend + tests, produces target/*.jar
mvn -Dskip.frontend=true test          # backend tests only
java -jar target/*.jar
cd frontend && pnpm install && pnpm dev   # hot reload on :5173, proxies /api to :8080
```

Configuration comes from environment variables (see `.env.example`). Keep real values in a
gitignored file such as `env.sh` and source it before running.

## Layout

- `src/main/java/<base package>/alma` Alma REST client and error mapping
- `src/main/java/<base package>/config` Security (Cognito login, logout, entry points), properties
- `src/main/java/<base package>/web` `/api` controller, SPA fallback, error handler
- `frontend/src` React app; one component per table under `components/`
- Frontend build output goes to `target/classes/static` and is packaged into the jar

## Rules

- The Alma API key never reaches the browser. All Alma calls go through the backend.
- The Alma user ID comes only from the configured claim (`alma.user-id-claim`). Never fall
  back to the Cognito username.
- `user-name-attribute` stays `sub`. Other claims are not guaranteed in the userinfo response.
- Alma JSON is passed through unchanged; shape columns in the frontend, not the backend.
- Keep the SPA fallback limited to dot-free path segments; it must never answer `/assets/**`
  or `/api/**`.
- Every security or Alma client change needs a test in `src/test`. Run `mvn test` before
  reporting work as done.
- Do not commit `env.sh`, `.env`, or anything under `.pnpm-store`.
