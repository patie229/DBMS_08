# DBMS_08 – From Container to Docker Compose

**Module:** Databases · THGA Bochum  
**Lecturer:** Stephan Bökelmann · <sboekelmann@ep1.rub.de>  
**Repository:** <https://github.com/MaxClerkwell/DBMS_08>  
**Prerequisites:** DBMS_01 – DBMS_07, Lecture 09  
**Duration:** 120 minutes

---

## Learning Objectives

After completing this exercise you will be able to:

- Explain the difference between a **container** and a **virtual machine**
- Start, inspect, and remove Docker **containers** and **images**
- Write a minimal **Dockerfile** and build a custom image from it
- Demonstrate the **ephemeral** nature of container storage
- Attach a **named volume** to a PostgreSQL container and verify data persists
  across container restarts
- Explain why running two services in one container is an anti-pattern
- Connect two containers via a **custom bridge network** using container names
  as hostnames
- Describe all key sections of a **docker-compose.yml** file
- Start a multi-service application with `docker compose up` and take it down
  cleanly
- Use a **`.env` file** to keep credentials out of `docker-compose.yml`
- Explain what a **Multi-Stage Build** is and why it reduces image size
- Apply the **Principle of Least Privilege** by running containers as a
  non-root user
- Use **init scripts** to initialise a PostgreSQL database on first start

**After completing this exercise you should be able to answer the following questions independently:**

- What is the difference between a Docker image and a running container?
- Why does deleting a container without a volume lose all data written to it?
- Why can `host="localhost"` inside one container not reach a second container?
- What does `docker compose down -v` do that `docker compose down` does not?
- Why should credentials never appear directly in `docker-compose.yml`?

---

## Prerequisites Check

You need Docker and Docker Compose installed.

```bash
docker --version
docker compose version
```

> Both commands should succeed and show version numbers.

> **Screenshot 1:** Take a screenshot showing both version outputs.
>
> `[insert screenshot]`

---

## 1 – Hello Container

### Step 1 – Run the hello-world Image

```bash
docker run hello-world
```

> Docker pulls the image from Docker Hub (first run only), starts a container,
> prints a message, and exits.

List all containers, including stopped ones:

```bash
docker ps -a
docker images
```

> **Screenshot 2:** Take a screenshot showing `docker ps -a` and
> `docker images` output.
>
> `[insert screenshot]`

### Step 2 – Run an nginx Webserver

```bash
docker run -d -p 8080:80 --name webserver nginx
curl http://localhost:8080
docker logs webserver
docker exec -it webserver bash
```

Inside the container shell, inspect the running process and exit:

```bash
ps aux
exit
```

Stop and remove the container:

```bash
docker stop webserver && docker rm webserver
docker system df
```

### Questions for Section 1

**Question 1.1:** The flag `-d` starts the container in detached mode.
What happens without `-d`, and why is detached mode useful for a web server?

> **Answer:** Without `-d`, the container runs in the **foreground**
> (attached mode): your terminal is bound to the container's stdout/stderr,
> you see its log output live, and the terminal is blocked until the
> container stops (e.g. with `Ctrl+C`). With `-d` (detached), the container
> runs in the **background** and Docker immediately returns control of the
> terminal, printing only the container ID.
>
> Detached mode is useful for a web server because a server is a long-running
> process meant to keep serving requests indefinitely. You want it running in
> the background while you continue to use the terminal for other commands
> (`docker logs`, `curl`, etc.), rather than having one terminal permanently
> tied up.

**Question 1.2:** `-p 8080:80` maps host port 8080 to container port 80.
Which port is the application actually listening on inside the container?
What would `-p 9000:80` change?

> **Answer:** The application (nginx) listens on port **80 inside the
> container** — that is the right-hand side of the mapping. The left-hand
> side (8080) is the port exposed on the **host**. Traffic arriving at
> `host:8080` is forwarded to `container:80`.
>
> `-p 9000:80` would change only the **host** port: you would reach the same
> nginx (still listening on 80 internally) via `http://localhost:9000`
> instead of `:8080`. The container-side port stays 80.

---

## 2 – Writing a Dockerfile

### Step 1 – Create the Project Directory

```bash
mkdir ~/dbms09_dockerfile
cd ~/dbms09_dockerfile
git init
git remote add origin git@github.com:<your-username>/dbms09_dockerfile.git
```

### Step 2 – Write the Dockerfile

```bash
vim Dockerfile
```

```dockerfile
FROM debian:13
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
CMD ["bash"]
```

### Step 3 – Build and Run

```bash
docker build -t mein-debian .
docker run -it mein-debian
```

Inside the container:

```bash
curl --version
whoami
exit
```

> **Screenshot 3:** Take a screenshot showing the `docker build` output and
> the commands run inside the container.
>
> `[insert screenshot]`

### Step 4 – Commit

```bash
git add Dockerfile
git commit -m "feat: minimal Debian image with curl"
git push -u origin main
```

### Questions for Section 2

**Question 2.1:** Why does the `RUN` instruction combine `apt-get update`,
`apt-get install`, and `rm -rf /var/lib/apt/lists/*` in a single line?
What would happen to the image size if these were three separate `RUN` lines?

> **Answer:** Each `RUN` instruction creates a new **image layer**, and a
> layer's contents are frozen once written — a later layer cannot shrink an
> earlier one. Combining the three commands in a single `RUN` means the
> downloaded package lists are created *and* deleted within the same layer,
> so they never end up stored in the image.
>
> If these were three separate `RUN` lines, the `apt-get update` layer would
> permanently contain the downloaded package index (tens of MB), and the
> later `rm -rf` would only *hide* it in a subsequent layer without removing
> it from the earlier one. The image would be larger, because the deleted
> files still occupy space in the layer where they were created. Combining
> also avoids the "stale cache" problem where a cached `update` layer serves
> outdated package indexes to a later `install`.

**Question 2.2:** `EXPOSE 80` in a Dockerfile does **not** actually open port
80. What does it do, and what is required at `docker run` time to actually
forward a port?

> **Answer:** `EXPOSE 80` is **documentation / metadata**. It declares that
> the containerised application is intended to listen on port 80, which tools
> and `docker inspect` can read, and which `-P` (uppercase) can use to
> auto-publish. By itself it does not publish or open anything on the host.
>
> To actually forward a port you must publish it at run time with the `-p`
> (lowercase) flag, e.g. `docker run -p 8080:80 ...`, which creates the
> host→container port mapping. (Or use `-P` to publish all `EXPOSE`d ports to
> random host ports.)

---

## 3 – The Persistence Problem

### Step 1 – Start a PostgreSQL Container Without a Volume

```bash
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -d postgres:16
```

Connect and create a table:

```bash
docker exec -it pg psql -U postgres
```

Inside `psql`:

```sql
CREATE TABLE test (id SERIAL PRIMARY KEY, wert TEXT);
INSERT INTO test (wert) VALUES ('Hallo Docker');
SELECT * FROM test;
\q
```

### Step 2 – Destroy and Recreate the Container

```bash
docker stop pg && docker rm pg
docker run --name pg -e POSTGRES_PASSWORD=geheim -d postgres:16
docker exec -it pg psql -U postgres -c "SELECT * FROM test;"
```

> Expected output: `ERROR: relation "test" does not exist`

> **Screenshot 4:** Take a screenshot showing the error message.
>
> `[insert screenshot]`

### Questions for Section 3

**Question 3.1:** You stopped and removed the container but the image
`postgres:16` still exists on your machine. Why does recreating a container
from the same image not restore the data?

> **Answer:** An **image** is a read-only template; a **container** adds a
> thin writable layer on top of it where all runtime changes (the tables and
> rows you created) are stored. When you `docker rm` the container, that
> writable layer is deleted along with everything written to it. The image
> `postgres:16` is unchanged — it never contained your data in the first
> place.
>
> Recreating a container from the same image gives you a fresh, empty
> writable layer. It starts from the identical initial state the image
> defines, so your `test` table (which lived only in the removed container's
> layer) is gone. Data survives container removal only if it was written to a
> **volume**, which lives outside the container's lifecycle.

**Question 3.2:** `docker stop` sends SIGTERM and waits for the process to
exit cleanly. `docker kill` sends SIGKILL immediately. Why is `docker stop`
preferred for a database container?

> **Answer:** `docker stop` sends `SIGTERM`, giving PostgreSQL a grace period
> to perform a **clean shutdown**: flush buffers to disk, complete or roll
> back in-flight transactions, write a checkpoint, and release locks. This
> leaves the data directory in a consistent state.
>
> `docker kill` sends `SIGKILL`, which terminates the process instantly with
> no chance to clean up. For a database this risks an unclean shutdown:
> uncommitted data in memory is lost and the server must perform crash
> recovery (WAL replay) on next start, which is slower and, in the worst
> case, risks corruption. `docker stop` is therefore the safe choice for a
> stateful service like a database.

---

## 4 – Named Volumes

### Step 1 – Create a Volume and Attach It

```bash
docker volume create pg_data
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    -d postgres:16
```

Insert data:

```bash
docker exec -it pg psql -U postgres
```

```sql
CREATE TABLE test (id SERIAL PRIMARY KEY, wert TEXT);
INSERT INTO test (wert) VALUES ('Daten überleben');
SELECT * FROM test;
\q
```

### Step 2 – Destroy and Recreate With the Same Volume

```bash
docker stop pg && docker rm pg
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    -d postgres:16
docker exec -it pg psql -U postgres -c "SELECT * FROM test;"
```

> The data should still be there.

```bash
docker volume ls
docker volume inspect pg_data
```

> **Screenshot 5:** Take a screenshot showing the `SELECT` result after
> container recreation, and the `docker volume inspect` output.
>
> `[insert screenshot]`

### Step 3 – Clean Up

```bash
docker stop pg && docker rm pg
docker volume rm pg_data
```

### Questions for Section 4

**Question 4.1:** `docker volume inspect pg_data` shows a `Mountpoint` on
the host filesystem. Why is it still recommended to use named volumes instead
of bind-mounting that path directly with `-v /var/lib/docker/volumes/...`?

> **Answer:** Named volumes are **managed by Docker**, which brings several
> advantages over reaching into the internal path directly. Docker handles
> their creation, lifecycle, permissions, and driver; they are portable
> across hosts and storage backends (a volume driver could put them on NFS,
> cloud storage, etc.); and they are referenced by a stable logical name
> rather than an implementation-specific path.
>
> The `/var/lib/docker/volumes/...` path is an **internal implementation
> detail** that can change between Docker versions and platforms (on
> Docker Desktop it lives inside a VM and is not directly accessible at all).
> Bind-mounting it directly couples your setup to that fragile path, bypasses
> Docker's management, and breaks portability. Use the name `pg_data`; let
> Docker manage where it physically lives.

**Question 4.2:** You want to back up the database. Which `docker` command
lets you copy files out of a running container, and how would you copy the
volume contents to a `.tar.gz` archive on the host?

> **Answer:** `docker cp` copies files between a container and the host, e.g.
> `docker cp pg:/var/lib/postgresql/data ./backup`. However, for a live
> database the recommended backup is a logical dump rather than copying data
> files:
>
> ```bash
> docker exec pg pg_dump -U postgres postgres | gzip > backup.sql.gz
> ```
>
> To archive the raw volume contents to a `.tar.gz` on the host, run a
> throwaway container that mounts the volume and tars it to a bind-mounted
> host directory:
>
> ```bash
> docker run --rm \
>     -v pg_data:/data:ro \
>     -v "$(pwd)":/backup \
>     debian:13 \
>     tar czf /backup/pg_data.tar.gz -C /data .
> ```
>
> This mounts `pg_data` read-only, mounts the current host directory as
> `/backup`, and writes the compressed archive there. (For consistency, stop
> the database or use `pg_dump` rather than archiving live data files.)

---

## 5 – Two Containers and the Network Problem

### Step 1 – Reproduce the Connectivity Failure

Start PostgreSQL:

```bash
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    -d postgres:16
```

Start a second container and try to reach the first via `localhost`:

```bash
docker run --rm -it postgres:16 \
    psql -h localhost -U postgres -c "SELECT 1;"
```

> This should fail with a connection refused error.

> **Screenshot 6:** Take a screenshot showing the connection error.
>
> `[insert screenshot]`

### Step 2 – Fix It With a Custom Bridge Network

```bash
docker network create mein-netz

docker stop pg && docker rm pg
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    --network mein-netz \
    -d postgres:16

docker run --rm -it \
    --network mein-netz \
    postgres:16 \
    psql -h pg -U postgres -c "SELECT 1;"
```

> Notice that `-h pg` uses the **container name** as the hostname — Docker's
> internal DNS resolves it automatically.

```bash
docker network inspect mein-netz
```

### Step 3 – Clean Up

```bash
docker stop pg && docker rm pg
docker network rm mein-netz
docker volume rm pg_data
```

### Questions for Section 5

**Question 5.1:** Without a custom bridge network, containers are placed on
the default bridge. Why can containers on the default bridge **not** resolve
each other by name, while containers on a user-defined bridge can?

> **Answer:** The reason is Docker's **embedded DNS server**. On a
> **user-defined bridge network**, Docker runs an internal DNS resolver that
> automatically maps each container's name (and network aliases) to its
> current IP, so containers can reach each other by name (`-h pg`).
>
> The **default bridge** network does not provide this automatic name
> resolution — for backward-compatibility reasons, DNS-based service discovery
> is not enabled there. Containers on the default bridge can only reach each
> other by IP address (or via the legacy, deprecated `--link` flag). Creating
> a user-defined bridge is the modern, recommended way and enables name-based
> discovery out of the box.

**Question 5.2:** You could find the IP address of the `pg` container with
`docker inspect` and hard-code it. Why is using the container name as a
hostname strongly preferable?

> **Answer:** Container IP addresses are **dynamic and ephemeral**: Docker
> assigns them at start time from the network's address pool, and a container
> can receive a different IP every time it is recreated or restarted.
> Hard-coding an IP means your configuration breaks as soon as the container
> is recreated.
>
> The container **name** is stable and Docker's embedded DNS resolves it to
> whatever the current IP is, automatically. Using the name (`host: postgres`
> in the API config) makes the setup resilient to restarts, self-documenting,
> and portable — exactly the abstraction that service discovery is meant to
> provide.

---

## 6 – Docker Compose

Managing multiple `docker run` commands by hand is error-prone. Docker Compose
describes an entire multi-service application in a single YAML file.

### Step 1 – Create the Project

```bash
mkdir ~/dbms09_compose
cd ~/dbms09_compose
git init
git remote add origin git@github.com:<your-username>/dbms09_compose.git
```

### Step 2 – Write docker-compose.yml

```bash
vim docker-compose.yml
```

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: vorlesung
      POSTGRES_PASSWORD: geheim
      POSTGRES_DB: vorlesung
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - backend

  api:
    image: python:3.12-slim
    working_dir: /app
    volumes:
      - ./api:/app
    command: >
      bash -c "pip install fastapi uvicorn psycopg2-binary --quiet
               && uvicorn main:app --host 0.0.0.0 --port 8000"
    ports:
      - "8000:8000"
    depends_on:
      - postgres
    networks:
      - backend

volumes:
  pg_data:

networks:
  backend:
```

### Step 3 – Create the API Code

```bash
mkdir api
vim api/main.py
```

```python
import psycopg2
import psycopg2.extras
from fastapi import FastAPI

app = FastAPI(title="Studenten-API")

DB_CONFIG = {
    "dbname": "vorlesung",
    "user": "vorlesung",
    "password": "geheim",
    "host": "postgres",   # container name as hostname
    "port": 5432,
}

@app.get("/")
def root():
    return {"status": "API läuft"}

@app.get("/studenten")
def alle_studenten():
    conn = psycopg2.connect(**DB_CONFIG)
    cur = conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor)
    cur.execute("SELECT id, matrikel, nachname, vorname FROM student ORDER BY nachname")
    rows = cur.fetchall()
    cur.close()
    conn.close()
    return rows
```

### Step 4 – Start and Test

```bash
docker compose up -d
docker compose ps
docker compose logs api
```

Wait for the API to start, then test:

```bash
curl http://localhost:8000/
curl http://localhost:8000/studenten
```

> The `/studenten` endpoint will return an empty list for now — that is
> expected. You will add the schema in Section 7.

> **Screenshot 7:** Take a screenshot showing `docker compose ps` and the
> `curl /` response.
>
> `[insert screenshot]`

### Step 5 – Observe Compose Networking

```bash
docker network ls
docker network inspect dbms09_compose_backend
```

> Compose automatically prefixes network names with the project directory name.

### Step 6 – Take It Down

```bash
docker compose down
docker compose down -v   # also removes the named volume
```

> **Question:** What is the difference between `down` and `down -v`?
> When would you use each?
>
> **Answer:** `docker compose down` stops and removes the containers and the
> default network, but **keeps** named volumes — so your database data
> survives. `docker compose down -v` additionally **removes the named
> volumes**, deleting all persisted data. Use plain `down` for a normal
> stop/restart where you want to keep your data; use `down -v` when you
> deliberately want a clean slate (for example to force the init script to
> run again, as in Section 7).

### Step 7 – Commit

```bash
git add docker-compose.yml api/main.py
git commit -m "feat: initial docker-compose setup with postgres and api"
git push -u origin main
```

### Questions for Section 6

**Question 6.1:** `depends_on: postgres` ensures the `postgres` service
starts before `api`. Does it guarantee that PostgreSQL is **ready to accept
connections** when the API starts? What is the correct way to handle this?

> **Answer:** **No.** Plain `depends_on` only controls **start order** — it
> waits until the postgres *container* has been started, not until PostgreSQL
> inside it has finished initialising and is ready to accept connections.
> PostgreSQL typically needs a few seconds to initialise, so the API may try
> to connect before the database is listening and fail.
>
> The correct way is to wait for **readiness**, not just startup. Options:
> - Add a **healthcheck** to the postgres service (e.g. using `pg_isready`)
>   and use the long form `depends_on:  postgres:  condition:
>   service_healthy` so Compose waits until the database reports healthy.
> - Make the application **retry** the connection with backoff on startup
>   (robust even outside Compose).
> - Use a wait-for script (e.g. `wait-for-it.sh` / `pg_isready` loop) in the
>   API's entrypoint before launching uvicorn.

**Question 6.2:** The `api` service uses `volumes: - ./api:/app` (a bind
mount). What is the advantage of this during development compared to
`COPY`-ing the code into an image at build time?

> **Answer:** A bind mount maps your **host source directory** directly into
> the container, so edits you make on the host are immediately visible inside
> the container **without rebuilding the image**. Combined with a
> live-reloading server (uvicorn `--reload`), you can change `main.py` and see
> the effect instantly — a fast edit/test loop ideal for development.
>
> With `COPY` at build time, the code is baked into the image, so every change
> requires rebuilding the image and recreating the container. That is the
> correct approach for **production** (immutable, reproducible, self-contained
> image), but slow and cumbersome during active development.

---

## 7 – Init Script for PostgreSQL

The official `postgres` image runs all scripts placed in
`/docker-entrypoint-initdb.d/` on first start. This lets you initialise the
schema automatically.

### Step 1 – Write init.sql

```bash
vim init.sql
```

```sql
CREATE TABLE IF NOT EXISTS student (
    id        INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    matrikel  CHAR(8)      NOT NULL UNIQUE,
    nachname  VARCHAR(100) NOT NULL,
    vorname   VARCHAR(100) NOT NULL,
    email     VARCHAR(200)
);

INSERT INTO student (matrikel, nachname, vorname, email) VALUES
    ('12345678', 'Meier',   'Anna',  'a.meier@stud.thga.de'),
    ('23456789', 'Schmidt', 'Ben',   'b.schmidt@stud.thga.de'),
    ('34567890', 'Yilmaz',  'Ceren', 'c.yilmaz@stud.thga.de'),
    ('45678901', 'Nguyen',  'David', 'd.nguyen@stud.thga.de');
```

### Step 2 – Mount the Script in docker-compose.yml

Add a bind mount to the `postgres` service so the script lands in the init
directory:

```yaml
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: vorlesung
      POSTGRES_PASSWORD: geheim
      POSTGRES_DB: vorlesung
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend
```

### Step 3 – Reinitialise and Test

The init script only runs when the data directory is empty. Remove the
existing volume first:

```bash
docker compose down -v
docker compose up -d
```

Wait a moment, then query the data:

```bash
curl http://localhost:8000/studenten
```

> You should now see all four students in the JSON response.

> **Screenshot 8:** Take a screenshot showing the `curl /studenten` response
> with all four rows.
>
> `[insert screenshot]`

### Step 4 – Commit

```bash
git add init.sql docker-compose.yml
git commit -m "feat: add init.sql for automatic schema and seed data"
git push
```

### Questions for Section 7

**Question 7.1:** You run `docker compose down` (without `-v`), change
`init.sql`, and run `docker compose up -d` again. The schema change does
**not** appear in the database. Why not, and how do you force re-initialisation?

> **Answer:** Scripts in `/docker-entrypoint-initdb.d/` are executed by the
> postgres image **only when the data directory is empty** — that is, on a
> *first* initialisation. Since `docker compose down` (without `-v`) keeps the
> `pg_data` volume, the data directory is already populated on the next
> `up`, so the entrypoint skips the init scripts entirely and your changed
> `init.sql` is ignored.
>
> To force re-initialisation you must remove the volume so the data directory
> is empty again:
>
> ```bash
> docker compose down -v
> docker compose up -d
> ```
>
> (Alternatively, apply the schema change with a migration/`psql` command
> against the running database rather than relying on the init script.)

**Question 7.2:** `GENERATED ALWAYS AS IDENTITY` is used instead of
`SERIAL`. What is the practical difference? Which one is the modern
SQL-standard approach?

> **Answer:** `SERIAL` is a PostgreSQL-specific shortcut that creates an
> integer column backed by a separate `SEQUENCE` and sets its default to
> `nextval(...)`. Because it is only a *default*, a client can still insert an
> explicit value into the column, which can desynchronise the sequence and
> later cause duplicate-key errors.
>
> `GENERATED ALWAYS AS IDENTITY` is the **SQL-standard** approach (portable
> across compliant databases). With `ALWAYS`, PostgreSQL manages the value and
> **rejects** attempts to insert an explicit one (unless you use
> `OVERRIDING SYSTEM VALUE`), which prevents accidental sequence
> desynchronisation. It is the modern, recommended choice; `SERIAL` is
> considered legacy.

---

## 8 – Secrets via .env

Passwords must not appear in `docker-compose.yml` because that file is
committed to version control.

### Step 1 – Create .env

```bash
vim .env
```

```
POSTGRES_USER=vorlesung
POSTGRES_PASSWORD=geheim
POSTGRES_DB=vorlesung
```

### Step 2 – Reference Variables in docker-compose.yml

Replace the hard-coded values in the `postgres` service:

```yaml
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
```

Also update `api/main.py` to read the password from the environment:

```python
import os

DB_CONFIG = {
    "dbname": os.environ.get("POSTGRES_DB", "vorlesung"),
    "user": os.environ.get("POSTGRES_USER", "vorlesung"),
    "password": os.environ.get("POSTGRES_PASSWORD", ""),
    "host": "postgres",
    "port": 5432,
}
```

Add `env_file` to the `api` service in `docker-compose.yml`:

```yaml
  api:
    ...
    env_file: .env
```

### Step 3 – Gitignore .env

```bash
echo ".env" >> .gitignore
```

### Step 4 – Restart and Verify

```bash
docker compose down -v
docker compose up -d
curl http://localhost:8000/studenten
```

> The response should be identical to before.

### Step 5 – Commit

```bash
git add docker-compose.yml api/main.py .gitignore
git commit -m "feat: move credentials to .env file"
git push
```

> **Do not add `.env` to the commit.** Confirm with `git status` that it
> is untracked.

> **Screenshot 9:** Take a screenshot showing `git status` confirming
> `.env` is not staged, and the working `curl` response.
>
> `[insert screenshot]`

### Questions for Section 8

**Question 8.1:** A teammate clones your repository and runs
`docker compose up -d`. The application fails because `.env` is missing.
What is the standard practice to document which variables are required
without committing the actual secrets?

> **Answer:** The standard practice is to commit a **template file**,
> conventionally named `.env.example` (or `.env.sample`), that lists **all
> required variable names** with placeholder or empty values but **no real
> secrets**:
>
> ```
> POSTGRES_USER=
> POSTGRES_PASSWORD=
> POSTGRES_DB=
> ```
>
> This file *is* committed to version control and documents the required
> configuration. A teammate copies it (`cp .env.example .env`) and fills in
> the real values locally. The actual `.env` stays in `.gitignore`. A note in
> the README explaining this step is also good practice.

**Question 8.2:** Even with `.env` excluded from git, the password is still
stored in plain text on disk. Name one mechanism Docker provides for
production-grade secret management that avoids plain-text env files entirely.

> **Answer:** **Docker Secrets** (available with Docker Swarm and referenced
> in Compose via the top-level `secrets:` section). Instead of an environment
> variable, a secret is stored encrypted in the Swarm's Raft store and mounted
> into the container at runtime as an in-memory file under `/run/secrets/`,
> readable only by the services granted access. This keeps the value out of
> plain-text env files, out of the image, and out of `docker inspect` output.
>
> (Other valid production-grade options: an external secrets manager such as
> HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault, from which the
> application fetches credentials at runtime.)

---

## 9 – Multi-Stage Build

A Python image that includes `pip`, build tools, and cache is larger than
necessary. A Multi-Stage Build separates dependency installation from the
final runtime image.

### Step 1 – Create an api/Dockerfile

```bash
vim api/Dockerfile
```

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml .
RUN pip install uv && uv sync --no-dev

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv .venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Step 2 – Add pyproject.toml

```bash
vim api/pyproject.toml
```

```toml
[project]
name = "studenten-api"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "fastapi",
    "uvicorn[standard]",
    "psycopg2-binary",
]
```

### Step 3 – Switch the api Service to Use the Build

Replace the `image:` key with `build:` in the `api` service:

```yaml
  api:
    build: ./api
    ports:
      - "8000:8000"
    depends_on:
      - postgres
    env_file: .env
    networks:
      - backend
```

Remove the `volumes: - ./api:/app` bind mount and the `command:` key — they
were only needed for the quick-start approach.

### Step 4 – Build and Test

```bash
docker compose down -v
docker compose build
docker images    # compare sizes
docker compose up -d
curl http://localhost:8000/studenten
```

> **Screenshot 10:** Take a screenshot showing `docker images` with the
> final image size and the working `curl` response.
>
> `[insert screenshot]`

### Step 5 – Commit

```bash
git add api/Dockerfile api/pyproject.toml docker-compose.yml
git commit -m "feat: multi-stage Dockerfile for slim production image"
git push
```

### Questions for Section 9

**Question 9.1:** `COPY --from=builder /app/.venv .venv` copies the virtual
environment from the builder stage. The final image does not contain `pip` or
`uv`. What security advantage does this provide?

> **Answer:** It **reduces the attack surface**. The final runtime image
> contains only the resolved virtual environment and the application — not the
> build tools (`pip`, `uv`), compilers, or package caches used to produce it.
> Every tool present in an image is something an attacker who gains access to
> the container could potentially abuse (to install malicious packages,
> compile exploits, or fetch code). By excluding the package managers and
> build toolchain, there are fewer binaries to exploit and fewer components
> that could carry vulnerabilities.
>
> As a bonus, the image is also smaller and faster to pull/deploy, and has
> fewer packages for vulnerability scanners to flag.

**Question 9.2:** The builder stage installs dependencies from `pyproject.toml`
before copying the application code. Why does this ordering improve build
cache efficiency when you frequently change only `main.py`?

> **Answer:** Docker caches image layers and reuses a layer only if its
> instruction **and all preceding layers** are unchanged. By copying
> `pyproject.toml` and running the (slow) dependency install **before**
> copying the application code, the expensive install layer depends only on
> `pyproject.toml`.
>
> When you change only `main.py`, `pyproject.toml` is unchanged, so Docker
> reuses the cached dependency-install layer and only re-runs the cheap final
> `COPY . .`. If the code were copied *before* installing dependencies, any
> edit to `main.py` would invalidate the cache for the install step and force
> a full (slow) reinstall on every build. Ordering "least-frequently-changed
> first" maximises cache hits.

---

## 10 – Non-Root User

Containers run as `root` by default. If an attacker escapes the container,
they have root on the host.

### Step 1 – Add a Non-Root User to the Dockerfile

Open `api/Dockerfile` and add the user before the `CMD`:

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml .
RUN pip install uv && uv sync --no-dev

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv .venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
RUN adduser --disabled-password --gecos "" appuser
USER appuser
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Step 2 – Rebuild and Verify

```bash
docker compose build
docker compose up -d
docker compose exec api whoami
```

> Expected output: `appuser`

> **Screenshot 11:** Take a screenshot showing `docker compose exec api whoami`
> returning `appuser`.
>
> `[insert screenshot]`

### Step 3 – Commit

```bash
git add api/Dockerfile
git commit -m "feat: run api container as non-root appuser"
git push
```

### Questions for Section 10

**Question 10.1:** The `USER appuser` instruction is placed after
`COPY . .`. Why would placing it *before* `COPY` cause a permission problem?

> **Answer:** Each `COPY` writes files owned by the user that is active at
> that point in the build. With `USER appuser` placed **after** `COPY . .`,
> the copy runs as `root`, and the application files are then owned by root
> but readable by everyone (so `appuser` can still read and run them).
>
> If `USER appuser` were placed **before** `COPY`, the copy would run as the
> unprivileged `appuser`, which may lack permission to write into the target
> directory (e.g. `/app` created by an earlier root step) — causing the build
> to fail or produce files the process cannot manage. More generally, install
> and copy steps often need root privileges; the convention is to do all
> privileged setup as root first and switch to the non-root user with `USER`
> as the **last** step, just before `CMD`, so the *running process* is
> unprivileged while the *build* still had the rights it needed.

**Question 10.2:** State the **Principle of Least Privilege** in one
sentence, and name one other place in a typical web application stack
(outside of containers) where this principle is applied.

> **Answer:** The **Principle of Least Privilege** states that every user,
> process, or component should be granted only the minimum permissions
> necessary to perform its function — and no more.
>
> Another place it is applied in a typical web stack: the **database role**
> used by the application should have only the privileges it needs (e.g.
> `SELECT/INSERT/UPDATE` on specific tables) rather than being a superuser —
> exactly the dedicated non-superuser role created in DBMS_06. (Other valid
> examples: filesystem permissions restricting which files a service can read;
> a reverse proxy exposing only ports 80/443 while backend services stay
> private; scoped API tokens/OAuth scopes granting access to only certain
> endpoints.)

---

## 11 – Reflection

**Question A – The Monolith Anti-Pattern:**  
Section 6 of the lecture shows a Dockerfile that runs both PostgreSQL and
FastAPI in a single container. Describe two concrete operational problems
this causes in a production environment.

> **Answer:**
> 1. **No independent scaling or lifecycle:** The database and the API have
>    very different scaling and restart needs. If you want to run three API
>    instances behind a load balancer, you cannot — you would also spin up
>    three PostgreSQL instances fighting over the same data. Restarting or
>    redeploying the API to ship a bug fix would also restart the database,
>    causing unnecessary downtime for a stateful service.
> 2. **Broken process model and observability:** A container is designed
>    around a single main process (PID 1). Running two services means adding a
>    process supervisor and losing Docker's clean per-service logs, health
>    checks, and exit-code semantics. If PostgreSQL crashes, Docker cannot
>    detect and restart *just* the database; `docker logs` mixes both
>    services' output; and a crash of one can take down the other.
>
> (Other valid problems: you cannot patch/upgrade one service without
> rebuilding the whole image; you cannot place them on different hosts;
> security blast radius is larger since both share one container.)

**Question B – Volume vs. Bind Mount:**  
Compare named volumes and bind mounts. When is each type appropriate?

> **Answer:** A **named volume** is storage fully managed by Docker, living in
> Docker's own area and referenced by a logical name (e.g. `pg_data`). A
> **bind mount** maps a specific **host path** directly into the container
> (e.g. `./api:/app` or `./init.sql:/docker-entrypoint-initdb.d/init.sql`).
>
> - **Named volumes** are appropriate for **persistent application data**
>   whose host location you do not care about and want Docker to manage —
>   above all database data directories. They are portable, backup-friendly,
>   and decoupled from the host's directory layout.
> - **Bind mounts** are appropriate when the container must see a **specific
>   host file or directory**: mounting source code during development for live
>   reload, injecting a config file or init script, or sharing a known host
>   directory. They tie the container to the host's filesystem layout, which
>   is fine for dev and config but usually undesirable for production data.

**Question C – Compose and Reproducibility:**  
A colleague says: "I can just write the `docker run` commands in a shell
script — why do I need `docker-compose.yml`?" Give two specific advantages
of Compose over a shell script of `docker run` commands.

> **Answer:**
> 1. **Declarative desired-state management:** `docker-compose.yml` describes
>    the *desired end state* of the whole application (services, networks,
>    volumes, dependencies). `docker compose up` reconciles reality to that
>    state — creating only what is missing — and `docker compose down` cleanly
>    tears down exactly what it created, including the network and (with `-v`)
>    volumes. A shell script only runs imperative commands; cleanup and
>    idempotency you must code and maintain by hand.
> 2. **Built-in orchestration of multi-service concerns:** Compose handles
>    inter-service wiring for you — it creates a shared user-defined network
>    with automatic DNS (so `host: postgres` just works), manages start order
>    via `depends_on`, namespaces resources by project, and centralises
>    environment/`.env` handling. Reproducing all of that correctly and
>    consistently in a shell script is verbose and error-prone.
>
> (Other valid advantages: one readable, version-controlled file documents the
> entire stack; easy to share and run identically on any machine; scaling with
> `--scale`; consistent logs via `docker compose logs`.)

**Question D – The Complete Chain:**  
You have now built and containerised the full stack: PostgreSQL in a
container with a named volume and init script → FastAPI in a slim
non-root image → both orchestrated by Docker Compose with credentials
in `.env`. Describe in two sentences what each layer contributes to
**portability** and **security**.

> **Answer:** For **portability**: the PostgreSQL container with a named
> volume and init script gives a self-contained, reproducible database that
> initialises its own schema anywhere; the slim FastAPI image bundles the app
> and its exact dependencies so it runs identically on any Docker host; and
> Docker Compose ties them together in one declarative file so the whole stack
> starts identically with a single `docker compose up` on any machine.
>
> For **security**: the multi-stage slim image and non-root `appuser` minimise
> the attack surface and limit damage from a container escape; the `.env` file
> (git-ignored) keeps credentials out of the image and out of version control;
> and the user-defined Compose network isolates the database on an internal
> network so it is reachable by the API by name but not needlessly exposed to
> the host or the internet.

---

## Further Reading

- [Docker – Get started](https://docs.docker.com/get-started/)
- [Docker – Volumes](https://docs.docker.com/storage/volumes/)
- [Docker – Networking overview](https://docs.docker.com/network/)
- [Docker Compose – Reference](https://docs.docker.com/compose/compose-file/)
- [Docker – Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [postgres Docker Hub – Environment variables](https://hub.docker.com/_/postgres)
- Lecture 09 handout
