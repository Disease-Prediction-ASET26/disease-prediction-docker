# Disease Prediction Docker setup

This repository runs the AI, backend, and frontend repositories together. Keep
all four repositories as sibling folders:

```text
disease-prediction/
├── disease-prediction-docker/
├── disease-prediction-ai/
├── disease-prediction-be/
└── disease-prediction-fe/
```

The Compose build contexts point to those sibling folders. Clone the app
repositories beside this one before building.

## Configure local values

From this directory:

```sh
cp .env.example .env
openssl rand -base64 32
```

Paste the generated value into `DATA_ENCRYPTION_KEY`. Set unique local values
for `POSTGRES_PASSWORD` and `SONAR_DB_PASSWORD`. Keep `.env` out of Git.
`LOCAL_UID` and `LOCAL_GID` should match `id -u` and `id -g` on Linux so bind
mounted files stay owned by your user. Create the bind mount folders once:

```sh
mkdir -p ../disease-prediction-ai/models ../disease-prediction-ai/data
mkdir -p ../disease-prediction-be/.cache/maven
```

The API backend requires the encryption key before it starts. The local, dev,
and production Compose profiles connect the backend to the AI service over the
Compose network. The backend sends free text and checklist symptoms to
`POST /predict`, then returns the ranked result through `POST /api/assessments`.
The AI readiness check loads the classifier and language models before Compose
starts the backend. Add the trained `best.joblib` file under
`disease-prediction-ai/models/` for these profiles. Oracle dev uses a separate
reviewed AI runtime image that contains the classifier and pinned language
model weights.

## Local development and testing

Start the live source mounts and PostgreSQL:

```sh
docker compose --profile local up --build
docker compose --profile local logs -f
```

Open the Vite frontend at `http://localhost:5173`, the backend at
`http://localhost:8080`, and the AI API at `http://localhost:8000`. Requests
flow from Vite through the backend to AI. PostgreSQL is available on the host
at `localhost:5433`; its port is bound to loopback.

The frontend and AI test commands can run in their local containers:

```sh
docker compose exec frontend-local npm test
docker compose exec frontend-local npm run test:integration
docker compose exec ai-local python -m pytest tests
```

Backend tests use Testcontainers. Run them from the backend repo on the host so
Testcontainers can use the host Docker daemon:

```sh
cd ../disease-prediction-be
./mvnw -B -ntp test
```

Stop the stack and keep data with `docker compose --profile local down`. Add
`-v` to remove PostgreSQL, downloaded model cache, and frontend dependency
volumes. The generated `models/` and `data/` folders remain in the AI repo.

## Dev and production image checks

The dev profile builds the same immutable, non-root service images used for the
`develop` branch. It keeps logs on the container output and publishes local
ports `18000` (AI), `18080` (backend), and `18081` (frontend):

```sh
docker compose --profile dev up --build
docker compose --profile dev logs -f
```

The production profile builds the production Dockerfiles and runs them against
a local PostgreSQL container on loopback ports `28000`–`28081`:

```sh
docker compose --profile prod up --build
```

This profile is for a local production-image check. It is not a production
deployment. Oracle dev uses the reviewed AI runtime image, which includes the
classifier and pinned inference weights. Production on k3s should use reviewed
GHCR image digests, a managed database, and secrets from the cluster secret
store. Set `DATA_ENCRYPTION_KEY` as a Kubernetes Secret. The frontend proxies
`/api` to the `backend` service by default; set `API_UPSTREAM` if the backend
service has another name.

CI builds `Dockerfile.dev` from `develop` and pull requests into `develop`. It
builds `Dockerfile.prod` from `main` and pull requests into `main`. Every image
is scanned before publishing. Dev images use the `-dev` package and `develop`
tag; production images use the repository package and `main` tag. Use the
reported digest for deployment.

## Optional SonarQube

SonarQube is isolated behind the `sonar` profile. It uses a local PostgreSQL
database and keeps its data in named volumes:

```sh
docker compose --profile sonar up -d sonar-db sonarqube
```

Wait for `http://localhost:9000` to finish starting. Log in with `admin` /
`admin`, change the password, and create a user token under **My Account →
Security**. Put the token in `SONAR_TOKEN` in `.env`. The scanner image runs as
an amd64 container; Docker can emulate it on an ARM host, with slower scans.

Select the repository with `SONAR_REPO`. For example, scan the AI source:

```sh
SONAR_REPO=disease-prediction-ai docker compose --profile sonar run --rm sonar-scanner \
  -Dsonar.projectKey=disease-prediction-ai \
  -Dsonar.sources=prediction_service
```

For coverage, create the report before scanning. AI:

```sh
docker compose exec ai-local python -m pytest tests --cov=prediction_service --cov-report=xml
SONAR_REPO=disease-prediction-ai docker compose --profile sonar run --rm sonar-scanner \
  -Dsonar.projectKey=disease-prediction-ai \
  -Dsonar.sources=prediction_service \
  -Dsonar.python.coverage.reportPaths=coverage.xml
```

Backend (run `./mvnw verify` from the backend repo first):

```sh
SONAR_REPO=disease-prediction-be docker compose --profile sonar run --rm sonar-scanner \
  -Dsonar.projectKey=disease-prediction-be \
  -Dsonar.sources=src/main/java \
  -Dsonar.java.binaries=target/classes \
  -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
```

Frontend (run `npm ci && npm run coverage` from the frontend repo first):

```sh
SONAR_REPO=disease-prediction-fe docker compose --profile sonar run --rm sonar-scanner \
  -Dsonar.projectKey=disease-prediction-fe \
  -Dsonar.sources=src \
  -Dsonar.tests=src \
  '-Dsonar.test.inclusions=**/*.test.ts,**/*.test.tsx' \
  -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
```

On Linux, SonarQube needs host `vm.max_map_count` of at least `524288` and
`fs.file-max` of at least `131072`. Check with `sysctl vm.max_map_count
fs.file-max`; if needed, set them using your host's normal system configuration.
The Compose service also sets the open file and process limits. SonarQube is
limited to 2 CPUs and 4 GiB of memory, so start it only when needed.

Stop SonarQube with `docker compose --profile sonar down`. Add `-v` to remove
its database, projects, plugins, and scanner cache.

## Image design

- Builder tools stay in multi-stage build layers; production images contain
  only runtime dependencies and app output.
- Base images use version tags and manifest digests. Review digest updates as
  part of routine dependency updates.
- Build contexts exclude Git metadata, secrets, local dependencies, datasets,
  model artifacts, and generated files with each repo's `.dockerignore`.
- App processes use non-root UIDs, drop Linux capabilities, and write logs to
  stdout/stderr. Compose keeps published ports on loopback and rotates local
  container logs at 10 MiB across three files.
- Dependencies are installed from lock or exact version files. No app secret is
  passed as a build argument or copied into an image.

These follow Docker's [build best practices](https://docs.docker.com/build/building/best-practices/)
and [multi-stage build guidance](https://docs.docker.com/build/building/multi-stage/).
The Compose profiles keep local tools and SonarQube opt-in; see the
[Compose profiles guide](https://docs.docker.com/compose/how-tos/profiles/).
