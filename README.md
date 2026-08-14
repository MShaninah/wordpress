# WordPress with Docker Compose

## Table of Contents

- [Description](#description)
- [Repository Contents](#repository-contents)
- [Quickstart](#quickstart)
- [Usage](#usage)
  - [Environment Variables](#environment-variables)
  - [Changing the Port](#changing-the-port)
  - [Changing Database Settings](#changing-database-settings)
  - [Data Persistence](#data-persistence)
  - [Restart Behavior](#restart-behavior)
  - [Managing the Stack](#managing-the-stack)
- [Security Notes](#security-notes)

## Description

This repository contains a ready-to-run [WordPress](https://wordpress.org/) setup based on Docker Compose. Its purpose is to spin up a complete WordPress installation — consisting of the WordPress application itself and a MySQL database — with a single command, without installing PHP, a web server, or a database on the host machine.

The stack consists of two services:

| Service     | Image                 | Purpose                                  |
|-------------|-----------------------|------------------------------------------|
| `wordpress` | `wordpress:6.8-apache`| The WordPress application (PHP + Apache) |
| `db`        | `mysql:8.0`           | The MySQL database backing WordPress     |

Both services run in a dedicated bridge network (`wordpress_net`), the database content is persisted in a named Docker volume (`db_data`), and all credentials are injected via a `.env` file that is excluded from version control.

## Repository Contents

| File                  | Description                                                          |
|-----------------------|----------------------------------------------------------------------|
| `docker-compose.yaml` | Defines the `wordpress` and `db` services, network, and volume       |
| `.env.template`       | Template for the required `.env` file (copy it and fill in values)   |
| `.gitignore`          | Excludes secrets (`.env`), keys, and local data from the repository  |
| `README.md`           | This documentation                                                   |

## Quickstart

**Prerequisites:**

- [Docker Engine](https://docs.docker.com/engine/install/) (20.10+)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2, i.e. the `docker compose` command)

**Steps:**

```bash
# 1. Clone the repository and enter it
git clone <repository-url>
cd wordpress

# 2. Create your .env file from the template and set your own passwords
cp .env.template .env
nano .env

# 3. Start the stack
docker compose up -d
```

Then open `http://<host-ip>:8080` (e.g. `http://localhost:8080` locally, or the public IP of your cloud VM) in a browser and complete the WordPress installation wizard, where you choose your admin username and password.

## Usage

### Environment Variables

All configuration lives in the `.env` file (created from `.env.template`). The template ships with working values for all non-critical variables; only the credentials are placeholders you must replace.

| Variable                 | Template value       | Description                                        |
|--------------------------|----------------------|----------------------------------------------------|
| `WORDPRESS_DB_USER`      | placeholder, replace | MySQL user WordPress connects with                 |
| `WORDPRESS_DB_PASSWORD`  | placeholder, replace | Password for that MySQL user                       |
| `MYSQL_ROOT_PASSWORD`    | placeholder, replace | Root password of the MySQL server                  |
| `WORDPRESS_PORT`         | `8080`               | Host port WordPress is published on                |
| `WORDPRESS_DB_HOST`      | `db:3306`            | Database host as seen from the WordPress container |
| `WORDPRESS_DB_NAME`      | `wordpress`          | Name of the database (used by both services)       |
| `WORDPRESS_TABLE_PREFIX` | `wp_`                | Table prefix for the WordPress tables              |
| `WORDPRESS_DEBUG`        | `0`                  | Set to `1` to enable WordPress debug mode          |

To change any value, edit your `.env` file and restart the stack:

```bash
docker compose up -d
```

### Changing the Port

By default WordPress is reachable on port `8080`. To publish it on a different port (e.g. `9090`), change this line in your `.env` file:

```bash
WORDPRESS_PORT=9090
```

The `ports` mapping in `docker-compose.yaml` reads this variable for the host side; the container side (`80`) must stay unchanged.

### Changing Database Settings

The database name, user, and passwords are shared between both services through the same variables, so changing them in `.env` keeps WordPress and MySQL consistent. Note that MySQL only creates the database and user on the **first** start with an empty volume — if you change credentials later, either update them inside MySQL manually or reset the volume (see below, this deletes all data).

### Data Persistence

The database files are stored in the named volume `db_data` (mounted at `/var/lib/mysql`). This means all WordPress content — posts, users, settings — survives container restarts and re-creations:

```bash
docker compose down     # stops and removes the containers, data is KEPT
docker compose up -d    # starts again, all content is still there
```

To wipe everything and start from scratch:

```bash
docker compose down -v  # WARNING: -v deletes the db_data volume and all content
```

### Restart Behavior

Both services use `restart: unless-stopped`. If a container crashes or the Docker daemon restarts (e.g. after a VM reboot), the containers are started again automatically. They stay down only if you stopped them explicitly.

### Managing the Stack

```bash
docker compose ps          # show service status
docker compose logs -f     # follow the logs of both services
docker compose logs -f db  # follow only the database logs
docker compose restart     # restart both services
docker compose down        # stop and remove the containers (data is kept)
```

## Security Notes

- **Never commit the `.env` file.** It contains all credentials and is excluded via `.gitignore`.
- No passwords, tokens, usernames, or IP addresses are stored anywhere in this repository — only in your local `.env` file.
- On a public cloud VM, consider restricting access to port `8080` via your provider's firewall / security group while setting up, and choose strong passwords in `.env`.
