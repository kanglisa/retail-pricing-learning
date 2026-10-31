# Setup lesson A — Docker and PostgreSQL

## Goal

Allow 60–90 minutes plus download time. Install or verify Docker, start the
project's database, and explain ports and persistent storage. If Docker is already installed, begin with verification rather than reinstalling.

## 1. Understand the pieces

| Term | Meaning here |
|---|---|
| Docker Desktop | Mac application that manages the Docker environment |
| Image | Packaged software template: `postgres:17` |
| Container | An instance created from an image |
| Compose | Tool that reads `compose.yaml` and manages services |
| Service | Named component: our database service is `db` |
| Volume | Persistent storage separate from a container's lifecycle |
| Port | Number identifying a network connection endpoint |

An image is like a recipe; a container is an instance made from it. Database
records belong in persistent storage. Docker Desktop manages Linux containers
on the Mac; you do not need a separate native PostgreSQL installation.

## 2. Install Docker Desktop on a new Mac

1. Open Apple menu → **About This Mac**. Identify Apple silicon or Intel.
2. Visit [Docker's official Mac installation page](https://docs.docker.com/desktop/setup/install/mac-install/).
   Check current system requirements and download the matching installer.
3. Open the downloaded disk image and move Docker to Applications.
4. Start Docker from Applications and complete its setup prompts.
5. Wait for the engine to be running, then open a new VS Code terminal.

Follow the official instructions if installation screens differ. This project
does not require publishing an image or connecting to cloud hosting.

## 3. Verify installation and readiness

```sh
docker --version
docker compose version
docker info
```

The first two verify command availability; the third checks that the engine is
reachable. A version number alone does not prove the engine is running.

Optional first-container exercise:

```sh
docker run --rm hello-world
```

Expect a success message. The first run needs internet access to download its
image. `--rm` removes this example container after it exits, not the project's
database or the downloaded image.

## 4. Find the project and configuration

```sh
cd ~/projects/retail-pricing-learning
pwd
ls
```

Open `compose.yaml` and `.env.example` in VS Code. On a fresh checkout, **only if `.env` does not exist**, run:

```sh
cp .env.example .env
```

Replace the example password locally before the first database initialization.
Do not commit `.env`. Changing its password later does not automatically change
the password of an already initialized PostgreSQL user.

## 5. Read the Compose file yourself

Find this excerpt in the existing file; do not overwrite the complete file:

```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_DB: retail
      POSTGRES_USER: retail
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
    ports:
      - "127.0.0.1:${POSTGRES_PORT:-5434}:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

Explain each entry:

- `postgres:17` selects the image's major version.
- `retail` is the initial database and username; `db` is the service name.
- `${...}` reads configuration. The password is required; host port defaults to 5434.
- Localhost port 5434 forwards to PostgreSQL's port 5432 inside the container.
- `postgres_data` holds database files at the mounted data directory.
- The rest of the actual file defines the named volume and a readiness health check.

YAML indentation represents structure. Preserve spaces and nesting.

Draw and explain:

```text
Python on Mac → 127.0.0.1:5434 → PostgreSQL inside container:5432 → named volume
```

## 6. Start and inspect the database

```sh
docker compose up -d --wait
docker compose ps
docker compose logs --tail=30 db
```

`up` creates/starts services, `-d` runs them in the background, and `--wait`
waits for readiness. Expect `db` to be running and healthy. Initial image
downloads take time. Logs explain startup behavior; not every log line is an error.

Connect and run a query:

```sh
docker compose exec -T db psql -U retail -d retail -c "SELECT current_database(), current_user;"
```

Expected database and user: `retail`. `exec` runs a command in the existing
container. `psql` is the SQL client. `-c` executes SQL and exits. `-T` avoids
an interactive terminal.

For an interactive session:

```sh
docker compose exec db psql -U retail -d retail
```

At the SQL prompt, type `SELECT 2 + 3;`, then `\q` to leave. A semicolon finishes
a SQL statement. Lesson 1 creates your separate `retail_learning` database.

## 7. Understand the lifecycle

This pauses the database, so practice between dashboard sessions:

```sh
docker compose stop
docker compose ps -a
docker compose start --wait
```

Predict before running: data remains, but applications cannot query while the
database is stopped. Repeat the connection query after restarting.

`docker compose down` removes service containers/network while retaining named
volumes by default. **Do not add `-v` here**: that removes the named data volume.
Do not use volume pruning or a factory reset to troubleshoot an ordinary connection
problem. A Git commit does not back up database storage.

See [Docker's Compose walkthrough](https://docs.docker.com/compose/gettingstarted/)
for another example of services and persistent volumes.

## Troubleshooting

| Symptom | Check |
|---|---|
| Command not found | Complete installation and reopen the terminal |
| Cannot connect to engine | Open Docker Desktop and wait for readiness |
| No configuration file found | Run from the directory containing `compose.yaml` |
| Password variable missing | Check the local `.env` and `POSTGRES_PASSWORD` |
| Port already allocated | Check for another service on 5434; align any port change with Python configuration |
| Password rejected | Check existing credentials; editing `.env` does not reset an initialized user |
| Data appears missing | Verify database, Compose project, and volume before changing anything |

## Checkpoint

Explain image versus container versus volume, host versus container port, and
installed client versus running engine. Why does stopping preserve data? Why
keep an existing `.env` on an already working project?

Record versions and observations in your learning notes—never passwords. After
lesson 1 creates the folders, save them as `learning/reports/docker_setup.md`.

Next: [setup lesson B — Git](00_git.md), then [lesson 1](01_foundations.md).
