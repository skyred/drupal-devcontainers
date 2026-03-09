# Drupal Devcontainers — IDE as Code

> **Tired of Lando/DDev?** This devcontainer starter-kit gives you a faster, more composable, and truly portable Drupal development environment — with your IDE configured automatically.

### ⚡ 30 Seconds to Start

For most Drupal projects, you need exactly **4 commands**:

```bash
# 1. Clone this repo (one-time — put it alongside your projects)
git clone https://github.com/aspect-developer/drupal-devcontainers.git
#    ~/projects/
#    ├── drupal-devcontainers/    ← this repo
#    ├── my-drupal-site/
#    └── another-project/

# 2. Start a shared database (one-time — reuse across all projects)
cd drupal-devcontainers/shared-dev/database && docker compose up -d

# 3. Copy the devcontainer to your project
cp -r ../../../drupal-devcontainers/.devcontainer /path/to/my-drupal-site/

# 4. Open in VS Code — click "Reopen in Container" when prompted
code /path/to/my-drupal-site/
```

**That's it.** Xdebug, PHPCS, Drupal coding standards, IntelliSense — all configured. No `lando start`. No `ddev config`. Just open and code.

> *Everything below is details, customization, and optional services. The above is all you need to get started.*

---

## Why Devcontainers over Lando / DDev

| | **Devcontainers** | **Lando** | **DDev** |
|---|---|---|---|
| **Performance** | Native Docker — no wrapper overhead | Extra abstraction layer | Extra abstraction layer |
| **IDE Integration** | VS Code runs *inside* the container — full IntelliSense, debugging, linting | IDE runs on host, connects externally | IDE runs on host, connects externally |
| **Composability** | Pick exactly the services you need with standard Docker Compose | Opinionated `.lando.yml` recipes | Opinionated `config.yaml` |
| **Shared Services** | One database/Redis/Solr instance across *all* projects | Each project spins up its own DB | Each project spins up its own DB |
| **Docker Images** | Use any community/official Docker image | Locked to Lando-specific images | Locked to DDev-specific images |
| **Team IDE Consistency** | `devcontainer.json` enforces project-wide IDE settings (tab size, formatting, extensions) — every developer gets the *same* editor config automatically | IDE settings left to each developer; coding style drift is common | IDE settings left to each developer; coding style drift is common |
| **Reproducibility** | `devcontainer.json` is committed — every dev gets the same IDE *and* runtime environment | `.lando.yml` configures services only, not the IDE | `config.yaml` configures services only, not the IDE |
| **Resource Usage** | Shared services = fewer containers running | N projects = N×databases, N×webservers | N projects = N×databases, N×webservers |

### Performance Notes

#### FrankenPHP — Built-in Speed Advantage

The default Docker image used by this devcontainer is based on [FrankenPHP](https://frankenphp.dev/) — a modern PHP application server built on top of Caddy. Unlike traditional Apache + mod_php or Nginx + PHP-FPM setups, FrankenPHP:

- Serves PHP directly with **no reverse-proxy overhead**
- Supports **worker mode** — keeping your Drupal application in memory between requests (dramatically reducing bootstrap time)
- Handles **HTTP/2 and HTTP/3** out of the box
- Provides **automatic HTTPS** via Caddy

This means your local Drupal development environment is not only simpler but measurably faster than traditional LAMP stacks used by Lando and DDev.

#### What About macOS? The Mutagen Question

Let's be honest: DDev's [Mutagen](https://mutagen.io/) integration is the **one genuine advantage** it offers for macOS users. Docker on macOS suffers from slow file I/O due to the Linux VM layer, and Mutagen solves this by syncing files bidirectionally rather than using bind mounts.

This devcontainer setup does not yet include Mutagen support. However:

- **On Linux**, this is irrelevant — bind mounts are native-speed, and this devcontainer setup is already significantly faster than both Lando and DDev.
- **On macOS**, the FrankenPHP worker mode partially compensates by reducing the number of file reads per request (the application stays in memory).
- Docker Desktop's [VirtioFS file sharing](https://docs.docker.com/desktop/settings-and-maintenance/settings/#general) (default since Docker Desktop 4.15) has also dramatically narrowed the macOS performance gap.

> If you're on macOS and file I/O performance is critical, DDev + Mutagen may still edge out on raw filesystem operations. For everything else — IDE integration, composability, shared services, team consistency — devcontainers win.

### The "IDE as Code" Concept

Just as **Infrastructure as Code** manages servers through definition files, **IDE as Code** manages your development environment the same way. Your `devcontainer.json` declares:

- The Docker image (PHP version, extensions, tools)
- VS Code settings (Drupal coding standards, formatting rules)
- VS Code extensions (Xdebug, Intelephense, PHPCS, Twig support)
- Port mappings, environment variables, network configuration

New developer onboarding becomes: *clone → open in VS Code → start coding*.

---

## Architecture Overview

This setup uses a **two-layer networking model** that separates your project containers from shared infrastructure services.

**Layer 1 — Project containers**: Each Drupal project runs in its own devcontainer with its own isolated Docker network (named after the project folder). This is your PHP + Apache runtime where you write code, run Drush, debug with Xdebug, etc.

**Layer 2 — Shared services**: Common infrastructure (database, search, caching, email) runs as long-lived containers on a single `shared-dev` network. These are started once and reused across all your projects — no need to spin up a new MariaDB container every time you switch projects.

```
┌──────────────────────────────────────────────────────────────────┐
│  YOUR HOST MACHINE                                               │
│                                                                  │
│  ┌─────────────────────┐   ┌─────────────────────┐               │
│  │  Project A          │   │  Project B          │               │
│  │  ┌───────────────┐  │   │  ┌───────────────┐  │               │
│  │  │ .devcontainer │  │   │  │ .devcontainer │  │               │
│  │  │ (PHP + Apache)│  │   │  │ (PHP + Apache)│  │               │
│  │  └───────┬───────┘  │   │  └───────┬───────┘  │               │
│  │          │ network: │   │          │ network: │               │
│  │          │ project-a│   │          │ project-b│               │
│  └──────────┼──────────┘   └──────────┼──────────┘               │
│             │                         │                          │
│             │  docker network connect │                          │
│             ▼                         ▼                          │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  shared-dev network                                         │ │
│  │                                                             │ │
│  │  ┌────────┐ ┌──────┐ ┌─────────┐ ┌───────┐ ┌──────┐         │ │
│  │  │ devdb  │ │ solr │ │ mailhog │ │ redis │ │ollama│  ...    │ │
│  │  │MariaDB │ │ 9    │ │         │ │  7    │ │      │         │ │
│  │  └────────┘ └──────┘ └─────────┘ └───────┘ └──────┘         │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### How it works

1. **Each project creates its own network** automatically when you open it in VS Code (via `initializeCommand` in `devcontainer.json`).
2. **Shared services are started independently** — you pick only the ones you need. A simple Drupal site might only need `devdb`. A project with full-text search adds `solr`. A headless Drupal + decoupled frontend might need `devdb` + `redis` + `mailhog`.
3. **You bridge the networks** with `docker network connect YOUR_PROJECT devdb` to give your project access to a shared service. Each project connects only to the services it actually uses.
4. **Multiple projects share the same service instance**. Project A and Project B both talk to the same `devdb` container — they just use different database names. This is significantly more resource-efficient than Lando or DDev, where each project spins up its own database, webserver, and supporting services.

### Why this matters for multi-project workflows

As developers, we frequently juggle multiple projects. With Lando or DDev, switching between 3 projects means running 3 separate database containers, 3 webservers, and potentially 3 Redis/Solr instances — consuming memory and CPU even when idle. With this devcontainer approach, you run **one** database and **one** of each service, and every project connects to them as needed. Start a service once, use it everywhere.

---

## Quick Start

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed
- [VS Code](https://code.visualstudio.com/) with the [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension

### 1. Start Shared Services

Clone this repo (or copy the `shared-dev/` folder to a permanent location on your machine):

```bash
# Start the database (required for most projects)
cd shared-dev/database && docker compose up -d

# Start any other services you need
cd ../mailhog && docker compose up -d
cd ../solr && docker compose up -d
cd ../redis && docker compose up -d

# Optional: database management UI (pick one)
cd ../phpmyadmin && docker compose up -d   # → http://localhost:8080
cd ../adminer && docker compose up -d      # → http://localhost:8081

# Optional: local AI
cd ../ollama && docker compose up -d
```

> **Tip**: These services persist across reboots (they use `restart: unless-stopped`). You only need to start them once.

### 2. Set Up a Drupal Project

```bash
# Copy the devcontainer to your project root
cp -r .devcontainer /path/to/your-drupal-project/

# Open the project in VS Code
code /path/to/your-drupal-project/
```

VS Code will detect the `.devcontainer` folder and prompt you to **Reopen in Container**. Click yes.

### 3. Connect Shared Services to Your Project

Once your devcontainer is running, connect the shared services to your project's network:

```bash
# Connect the database (use your project folder name as the network name)
docker network connect YOUR_PROJECT_FOLDER devdb

# Connect other services as needed
docker network connect YOUR_PROJECT_FOLDER mailhog
docker network connect YOUR_PROJECT_FOLDER solr
docker network connect YOUR_PROJECT_FOLDER redis
```

### 4. Create a Database for Your Project

```bash
# From your host (or any terminal with docker access)
docker exec devdb mariadb -uroot -ppassword -e "CREATE DATABASE my_project;"
```

Then configure your Drupal `settings.php`:

```php
$databases['default']['default'] = [
  'driver' => 'mysql',
  'host' => 'devdb',        // ← container name, not localhost
  'database' => 'my_project',
  'username' => 'root',
  'password' => 'password',
  'port' => 3306,
  'prefix' => '',
];
```

---

## Devcontainer Variants

### `.devcontainer` — Drupal 8+ / 9 / 10 / 11 (Backend)

The primary devcontainer for PHP/Drupal backend development.

- **Image**: `insready/drupal-commerce:8-dev`
- **Mount**: Project root → `/opt/drupal`
- **Port**: `80` (Apache)
- **User**: `www-data`
- **Preconfigured**: Xdebug 3, PHPCS (Drupal + DrupalPractice standards), Intelephense, Twig support, PHP DocBlocker, REST Client

Copy to your project root:
```bash
cp -r .devcontainer /path/to/your-project/
```

### `D7.devcontainer` — Drupal 7 (Backend)

Same tooling as above, targeting Drupal 7.

- **Image**: `insready/drupal-dev:7`
- **Mount**: Project root → `/var/www`

Copy and rename to `.devcontainer`:
```bash
cp -r D7.devcontainer /path/to/your-d7-project/.devcontainer
```

### `.themingcontainer` — Drupal Theming (Frontend)

A Node.js devcontainer for Drupal theme development (Gulp, SASS, etc.).

- **Image**: `node:lts`
- **Auto-runs**: `npm install`, `gulp-cli` install, and `gulp` on attach

Copy to your custom theme folder and rename:
```bash
cp -r .themingcontainer /path/to/your-project/web/themes/custom/YOUR_THEME/.devcontainer
```

### Project Folder Structure

```
PROJECT_ROOT/
├── .devcontainer/                          # Backend Drupal devcontainer
│   ├── devcontainer.json
│   └── devcontainer.env
├── web/
│   └── themes/
│       └── custom/
│           └── YOUR_THEME/
│               └── .devcontainer/          # Theming devcontainer (optional)
│                   └── devcontainer.json
├── vendor/
└── composer.json
```

---

## Shared Services Reference

All services live in `shared-dev/` and share the `shared-dev` Docker network. Each service has its own `docker-compose.yml` so you only run what you need.

| Service | Container Name | Ports | Start Command |
|---|---|---|---|
| **MariaDB 11** | `devdb` | `3306` | `cd shared-dev/database && docker compose up -d` |
| **Apache Solr 9** | `solr` | `8983` ([UI](http://localhost:8983)) | `cd shared-dev/solr && docker compose up -d` |
| **Mailhog** | `mailhog` | `1025` (SMTP), `8025` ([UI](http://localhost:8025)) | `cd shared-dev/mailhog && docker compose up -d` |
| **Redis 7** | `redis` | `6379` | `cd shared-dev/redis && docker compose up -d` |
| **phpMyAdmin** | `phpmyadmin` | `8080` ([UI](http://localhost:8080)) | `cd shared-dev/phpmyadmin && docker compose up -d` |
| **Adminer** | `adminer` | `8081` ([UI](http://localhost:8081)) | `cd shared-dev/adminer && docker compose up -d` |
| **Ollama** | `ollama` | `11434` | `cd shared-dev/ollama && docker compose up -d` |

### Service Configuration in Drupal

#### Database (MariaDB)

```php
// settings.php or settings.local.php
$databases['default']['default'] = [
  'driver' => 'mysql',
  'host' => 'devdb',
  'database' => 'YOUR_DB_NAME',
  'username' => 'root',
  'password' => 'password',
  'port' => 3306,
  'prefix' => '',
];
```

#### Redis

Install the [Redis module](https://www.drupal.org/project/redis) and add to `settings.php`:

```php
$settings['redis.connection']['host'] = 'redis';
$settings['redis.connection']['port'] = '6379';
$settings['cache']['default'] = 'cache.backend.redis';
```

#### Solr

Install the [Search API Solr module](https://www.drupal.org/project/search_api_solr) and configure:
- **Solr host**: `solr`
- **Solr port**: `8983`
- **Solr core**: `drupal`

#### Mailhog

Install the [SMTP module](https://www.drupal.org/project/smtp) or [Mailsystem](https://www.drupal.org/project/mailsystem) and configure:
- **SMTP host**: `mailhog`
- **SMTP port**: `1025`
- **No authentication required**

View captured emails at [http://localhost:8025](http://localhost:8025).

---

## Customization

### Environment Variables

Edit `.devcontainer/devcontainer.env` to add project-specific environment variables:

```env
DRUSH_ALLOW_XDEBUG=1
MY_CUSTOM_VAR=value
```

### Adding More Services

Follow the pattern in `shared-dev/`:

1. Create a new folder: `shared-dev/YOUR_SERVICE/`
2. Add a `docker-compose.yml` with `name: shared-dev` and the `shared-dev` named network
3. Start it: `docker compose up -d`
4. Connect it: `docker network connect YOUR_PROJECT_FOLDER YOUR_SERVICE_CONTAINER`

### Using Docker Compose for Project-Specific Services

If you need services scoped to a single project (not shared), uncomment the `dockerComposeFile` line in your `devcontainer.json` and create a `docker-compose.extend.yml`:

```jsonc
// devcontainer.json
"dockerComposeFile": ["../docker-compose.yml", "docker-compose.extend.yml"],
```

### Adding a Local HTTPS Reverse Proxy

```bash
docker run --name caddy -p 443:443 -d caddy \
  caddy reverse-proxy --from localhost --to YOUR_PROJECT_NAME:80
```

---

## Migration Guide: Lando / DDev → Devcontainers

### Concept Mapping

| Lando / DDev | Devcontainers Equivalent |
|---|---|
| `.lando.yml` / `.ddev/config.yaml` | `.devcontainer/devcontainer.json` |
| `lando start` / `ddev start` | Open folder in VS Code → "Reopen in Container" |
| `lando stop` / `ddev stop` | Close VS Code window (container stops) |
| `lando drush` / `ddev drush` | Run `drush` directly in VS Code terminal (you're already inside the container) |
| `lando composer` / `ddev composer` | Run `composer` directly in VS Code terminal |
| `lando db-import` | `docker exec -i devdb mariadb -uroot -ppassword DB_NAME < dump.sql` |
| `lando db-export` | `docker exec devdb mariadb-dump -uroot -ppassword DB_NAME > dump.sql` |
| Per-project database containers | One shared `devdb` container (create separate databases per project) |
| `lando info` | `docker ps` / `docker network ls` |
| Custom services in `.lando.yml` | `shared-dev/` compose files or `dockerComposeFile` in `devcontainer.json` |
| Lando tooling commands | Shell aliases or VS Code tasks |

### Step-by-Step Migration

1. **Remove Lando/DDev config** from your project (`.lando.yml`, `.ddev/`)

2. **Copy the devcontainer** to your project root:
   ```bash
   cp -r .devcontainer /path/to/your-project/
   ```

3. **Start shared services** (if not already running):
   ```bash
   cd shared-dev/database && docker compose up -d
   ```

4. **Import your database**:
   ```bash
   # Create the database
   docker exec devdb mariadb -uroot -ppassword -e "CREATE DATABASE my_project;"

   # Import your dump
   docker exec -i devdb mariadb -uroot -ppassword my_project < my_dump.sql
   ```

5. **Update `settings.php`** to use `devdb` as the database host (see [Database configuration](#database-mariadb) above)

6. **Open in VS Code** and reopen in container when prompted

7. **Connect shared services**:
   ```bash
   docker network connect YOUR_PROJECT_FOLDER devdb
   docker network connect YOUR_PROJECT_FOLDER mailhog   # if needed
   ```

8. **Code!** — Xdebug, PHPCS, IntelliSense are all ready.

---

## Preconfigured IDE Settings

The devcontainer includes [Drupal.org recommended VS Code settings](https://www.drupal.org/docs/develop/development-tools/configuring-visual-studio-code):

- **Tab size**: 2 spaces
- **Ruler**: 80 characters
- **File associations**: `*.module`, `*.install`, `*.theme`, `*.inc` → PHP
- **PHPCS**: Drupal + DrupalPractice standards
- **PHPCBF**: Auto-format to Drupal standard
- **Intelephense**: K&R braces, PSR annotations
- **Xdebug**: Ready for step debugging (web and CLI)
- **Trailing whitespace**: Auto-trimmed on save
- **Final newline**: Auto-inserted on save

### Included Extensions

| Extension | Purpose |
|---|---|
| PHP Debug (Xdebug) | Step debugging |
| Intelephense | PHP IntelliSense |
| Apache Conf | Apache config highlighting |
| Twig | Twig template support |
| Composer | Composer integration |
| Empty Indent | Remove empty indentation |
| OpenAPI | OpenAPI/Swagger support |
| PHP DocBlocker | PHPDoc generation |
| PHPCBF | PHP Code Beautifier |
| PHPCS | PHP CodeSniffer |
| REST Client | HTTP request testing |
| DotENV | `.env` file support |
| Better PHPUnit | PHPUnit test runner |

---

## License

GPL-3.0 — See [LICENSE](LICENSE) for details.
