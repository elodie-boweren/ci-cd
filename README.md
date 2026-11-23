# CI/CD – SafeBase

![CI/CD](https://github.com/elodie-boweren/ci-cd/actions/workflows/ci-cd.yml/badge.svg)

This repository sets up a complete **CI/CD pipeline with GitHub Actions** on top of **SafeBase**, a web application that manages database backups and restores (Node.js / Express API + React frontend).

On every push to `main` and on every Git tag, the pipeline:

1. checks out the code;
2. runs the linter and the unit tests (and stops if one fails);
3. builds the Docker images for the development or production environment;
4. pushes them to the **GitHub Container Registry (GHCR)**;
5. sends a **Google Chat** notification with the result.

---

## Table of Contents

* [CI/CD Pipeline](#cicd-pipeline)
* [Docker Images](#docker-images)
* [Notifications](#notifications)
* [Configuration](#configuration)
* [Creating a Release](#creating-a-release)
* [The Application: SafeBase](#the-application-safebase)
* [Installation](#installation)
* [Testing](#testing)
* [Linting](#linting)
* [Project Structure](#project-structure)
* [Documentation](#documentation)

---

## CI/CD Pipeline

Two workflows live in [.github/workflows/](.github/workflows/):

| Workflow | Triggered on | Purpose |
| -------- | ------------ | ------- |
| [ci-cd.yml](.github/workflows/ci-cd.yml) | push on `main`, tag `v*.*.*`, Pull Request to `main` | Full pipeline: tests → build → push → notification |
| [lint.yml](.github/workflows/lint.yml)   | push on any other branch | Quick lint feedback while developing |

### Jobs of `ci-cd.yml`

```text
            ┌──────────────┐   ┌───────────────────┐
 push/tag → │  test-api    │   │  test-frontend    │
            │ lint + Jest  │   │ lint + Vitest     │
            └──────┬───────┘   └─────────┬─────────┘
                   └──────────┬──────────┘
                              ▼  (only if all tests pass, not on PRs)
                ┌───────────────────────────┐
                │ docker (api, frontend)    │
                │ build + push to GHCR      │
                └─────────────┬─────────────┘
                              ▼  (always)
                ┌───────────────────────────┐
                │ notify → Google Chat      │
                └───────────────────────────┘
```

| Job | Steps |
| --- | ----- |
| `test-api` | `npm ci` → `npm run lint` → `npm test` (Jest) in `api/` |
| `test-frontend` | `npm ci` → `npm run lint` → `npx vitest run` in `frontend/` |
| `docker` | Matrix on `api` and `frontend`: login to GHCR, compute tags, build and push the image |
| `notify` | Sends the pipeline status to Google Chat, even when a job failed |

* **Fail fast:** the first failing step stops its job, and `docker` depends on both test jobs (`needs`). No image is pushed if a test fails.
* **Pull Requests** only run the tests and the notification; no image is pushed.
* Tests are unit tests: databases and dump tools are mocked, so no container is needed in the CI.
* npm and Docker layer caches speed up the runs.

---

## Docker Images

Images are published on GHCR:

* `ghcr.io/elodie-boweren/ci-cd-api`
* `ghcr.io/elodie-boweren/ci-cd-frontend`

Tags depend on what triggered the pipeline:

| Trigger | Environment | Image tags |
| ------- | ----------- | ---------- |
| push on `main` | development | `dev`, `dev-<short-sha>` |
| tag `v1.2.3` | production | `1.2.3`, `1.2`, `latest` |

The environment is passed to the Dockerfiles with the `APP_ENV` build argument (`development` or `production`): it sets `NODE_ENV` in the API and `VITE_APP_ENV` in the frontend.

Pull an image:

```bash
docker pull ghcr.io/elodie-boweren/ci-cd-api:dev
docker pull ghcr.io/elodie-boweren/ci-cd-frontend:latest
```

---

## Notifications

At the end of every run, a message is posted to a Google Chat space through a webhook. It contains:

* the status: success, failure or cancelled;
* the branch or tag;
* the commit: short SHA, message and author;
* the result of each job;
* on failure, the last lines of the failing job's logs;
* a link to the full run in the **Actions** tab.

---

## Configuration

| Where | Name | Value |
| ----- | ---- | ----- |
| GitHub › Settings › Secrets and variables › Actions | `GOOGLE_CHAT_WEBHOOK` | Webhook URL of the Google Chat space (Space › Apps & integrations › Webhooks) |

* Pushing to GHCR uses the built-in `GITHUB_TOKEN` (`packages: write` permission is set in the workflow); no extra secret is needed.
* After the first push, set each package to **Public** (GitHub profile › Packages › package settings).

---

## Creating a Release

A production image is built from a Git tag following [Semantic Versioning](https://semver.org/):

```bash
git checkout main && git pull
git tag v1.0.0
git push origin v1.0.0
```

---

## The Application: SafeBase

SafeBase manages database backups and restores through a REST API and a web interface:

* manage connections to **MySQL** and **PostgreSQL** databases;
* run manual backups or schedule them with **cron** expressions;
* keep a history of backup versions and restore a selected one;
* get alerts when a backup or a restore fails.

```text
 Frontend (React + Vite, served by nginx)  :5173
                │ HTTP / REST
                ▼
 REST API (Node.js 22 + Express, JWT)      :3000
                │
        ┌───────┴────────┐
        ▼                ▼
     MySQL          PostgreSQL (also the app database)
     :3306          :5432
```

Backups use the native tools of each database: `mysqldump` (MariaDB client) and `pg_dump`.

---

## Installation

### Prerequisites

* **Git**
* **Docker Desktop** (includes Docker Compose): see [DOCKER.md](DOCKER.md)
* **Node.js 22** and npm, only to run the tests or the linter outside Docker

Ports `3000`, `3306`, `5432` and `5173` must be free.

### Start the application

```bash
git clone https://github.com/elodie-boweren/ci-cd.git
cd ci-cd
docker compose up --build -d
```

| Service | Container | Port |
| ------- | --------- | ---- |
| `frontend` | `ci_cd_frontend` | `5173` |
| `backend` | `ci_cd_backend` | `3000` |
| `postgres` | `ci_cd_postgres` | `5432` |
| `mysql` | `ci_cd_mysql` | `3306` |

* API health check: http://localhost:3000/api/health
* Interface: http://localhost:5173

No `.env` file is required: development values are set in [docker-compose.yml](docker-compose.yml). See [.env.example](.env.example) for the variables that can be overridden.

### Useful commands

| Command | Description |
| ------- | ----------- |
| `docker compose ps` | List the containers and their state |
| `docker compose logs -f backend` | Follow the API logs |
| `docker compose up --build -d` | Rebuild after a change in `api/` or `frontend/` |
| `docker compose down` | Stop the containers (data is kept) |
| `docker compose down -v` | Stop the containers **and delete the database data** |

If tables are missing, reset the volumes: `db/schema.sql` only runs when the PostgreSQL volume is first created.

---

## Testing

| Part | Tool | Location | Command |
| ---- | ---- | -------- | ------- |
| API | Jest | `api/tests/unit/` | `cd api && npm test` |
| Frontend | Vitest + React Testing Library | `frontend/tests/unit/` | `cd frontend && npx vitest run` |

Test names reference the cases of the test plan ([docs/testings/SAFEBASE - TestPlan.pdf](docs/testings/SAFEBASE%20-%20TestPlan.pdf)), e.g. `US02-07`. The API endpoints can also be tested with the Postman collection ([docs/testings/TESTS_API.md](docs/testings/TESTS_API.md)).

Run what the CI runs before pushing:

```bash
cd api && npm ci && npm run lint && npm test
cd ../frontend && npm ci && npm run lint && npx vitest run
```

---

## Linting

Both parts use [Oxlint](https://oxc.rs/docs/guide/usage/linter): `npm run lint` in `api/` or `frontend/`.

| Part | Configuration | Main rules |
| ---- | ------------- | ---------- |
| API | [api/.oxlintrc.json](api/.oxlintrc.json) | Node.js and Jest plugins, `correctness` errors, `eqeqeq` |
| Frontend | [frontend/.oxlintrc.json](frontend/.oxlintrc.json) | React plugin, `rules-of-hooks` |

Errors make the CI fail; warnings are only reported.

---

## Project Structure

```text
ci-cd/
├── .github/workflows/
│   ├── ci-cd.yml         # CI/CD pipeline (tests, Docker, GHCR, Google Chat)
│   └── lint.yml          # lint on feature branches
├── api/                  # REST API (Node.js / Express) + Dockerfile
│   └── tests/unit/       # Jest
├── frontend/             # React app (Vite) + Dockerfile (nginx)
│   └── tests/unit/       # Vitest
├── db/                   # schema and seed data
├── docs/                 # specs, data models, test plan, Postman collection
├── docker-compose.yml
├── DOCKER.md
└── .env.example
```

---

## Documentation

* [docs/SAFEBASE - Specs.pdf](docs/SAFEBASE%20-%20Specs.pdf): specifications
* [docs/SAFEBASE - UserStories.pdf](docs/SAFEBASE%20-%20UserStories.pdf): user stories
* [docs/database/](docs/database/): MCD / MLD / MPD
* [docs/testings/](docs/testings/): test plan and API tests
* [DOCKER.md](DOCKER.md): Docker installation and commands

---

Project developed as part of the **La Plateforme_** training program, for educational purposes.
