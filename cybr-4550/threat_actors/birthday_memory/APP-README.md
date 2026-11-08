# 🎂 Birthday Memory

**Never miss a birthday again.** A modern, colorful web app to save, search, and celebrate the birthdays of everyone you care about. Add first name, last name, birthdate, phone and email once, and the app keeps them organized, searchable, and visible on a beautiful month-by-month calendar.

Every birthday is stored permanently in a Postgres database. The calendar highlights upcoming celebrations with countdown timers, and the search bar finds anyone by name, email or phone in milliseconds.

## What it does

- **Add birthdays** — a five-field form saves first name, last name, birthdate, telephone number, and email address to the `thebirthdates` Postgres database.
- **Search instantly** — find anyone by typing their name, email, or phone number. Filter by birth month with colored pill buttons. Sort by soonest birthday, alphabetical order, or age.
- **View a calendar** — each month displays as a 6-week grid. Days with birthdays show colorful avatar chips with the person's initials. Click a day to see full details. A sidebar counts down the next five celebrations.
- **Deterministic color gradients** — each person gets a unique, stable color derived from their name, so they look identical across the card view, search results, and the calendar.
- **Edit and delete** — update any field or remove entries from the search results with inline actions.
- **Responsive design** — works on desktop, tablet, and mobile. The calendar compacts on narrow screens without losing functionality.

## Stack

| Layer    | Technology                         |
| -------- | ---------------------------------- |
| Frontend | React 18 + Vite, hand-written CSS  |
| Backend  | Node.js + Express 4 + `pg`         |
| Database | PostgreSQL 16 (Docker or local)   |
| Package manager | npm                        |

## Prerequisites

You need **two** things installed before you can run this app:

1. **Node.js 18 or newer** — runs both the API and the build tooling. Ships with `npm`.
2. **PostgreSQL 14 or newer** — stores the data. You can either install it natively or run the bundled Docker container (easiest).

> **Every operating system installs these differently.** macOS, Windows and Linux each have their own package managers and conventions, so pick the section below that matches your machine. The commands differ, but once both tools are installed, the rest of the setup is identical everywhere.

Check whether you already have them:

```bash
node --version    # want v18.0.0 or higher
npm --version     # ships with Node
docker --version  # only if you plan to use the bundled database
```

If a command prints `command not found`, that tool is missing — install it below.

### Installing on macOS

The usual route is [Homebrew](https://brew.sh). If you don't have it:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Then:

```bash
brew install node                      # Node.js + npm
brew install --cask docker             # Docker Desktop (for the bundled database)
```

Launch Docker Desktop once from Applications so the background daemon starts — the `docker` CLI fails with a socket error until you do.

On a fresh Mac, Homebrew may refuse to install anything until you accept the Xcode license:

```bash
sudo xcodebuild -license accept
```

Prefer a native database over Docker? `brew install postgresql@16 && brew services start postgresql@16`.

### Installing on Windows

Use [winget](https://learn.microsoft.com/windows/package-manager/winget/) (built into Windows 11 and recent Windows 10):

```powershell
winget install OpenJS.NodeJS.LTS
winget install Docker.DockerDesktop
```

Or download the installers directly from [nodejs.org](https://nodejs.org) and [docker.com](https://www.docker.com/products/docker-desktop/).

Things specific to Windows:

- **Restart your terminal** after installing Node, or `node` won't be on your `PATH`.
- Docker Desktop requires **WSL 2**. The installer normally sets this up; if it complains, run `wsl --install` in an admin PowerShell and reboot.
- Use **PowerShell** or **Windows Terminal**, not the legacy `cmd.exe`. All the `npm` commands below work unchanged.
- Running inside **WSL 2 (Ubuntu)** instead? Follow the Linux instructions below — that environment is Linux, not Windows.
- Prefer a native database? Download the [PostgreSQL Windows installer](https://www.postgresql.org/download/windows/) and note the port and password you choose during setup.

### Installing on Linux

**Debian / Ubuntu:**

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
sudo apt-get install -y docker.io docker-compose-plugin
```

**Fedora / RHEL:**

```bash
sudo dnf install -y nodejs npm
sudo dnf install -y docker docker-compose-plugin
```

**Arch:**

```bash
sudo pacman -S nodejs npm docker docker-compose
```

Things specific to Linux:

- The version of Node in default `apt` repositories is often **too old**. Use NodeSource (shown above) or [nvm](https://github.com/nvm-sh/nvm) to get 18+.
- Docker needs its daemon started and your user added to the `docker` group, otherwise every command needs `sudo`:
  ```bash
  sudo systemctl enable --now docker
  sudo usermod -aG docker $USER   # then log out and back in
  ```
- Prefer a native database? `sudo apt-get install postgresql` (or your distro's equivalent), then create the database manually as shown in step 3 below.

## Project dependencies

These install automatically via `npm run install:all` — listed here so you know what the app pulls in.

**Backend** (`server/package.json`):

| Package   | Purpose                                              |
| --------- | ---------------------------------------------------- |
| `express` | HTTP server and routing                              |
| `pg`      | PostgreSQL driver and connection pool                |
| `cors`    | Allows the browser client to call the API            |
| `dotenv`  | Loads database credentials from `server/.env`        |

**Frontend** (`client/package.json`):

| Package               | Purpose                                  |
| --------------------- | ---------------------------------------- |
| `react`, `react-dom`  | UI library                               |
| `vite`                | Dev server and production bundler        |
| `@vitejs/plugin-react`| React fast-refresh support for Vite      |

**Root** (`package.json`):

| Package        | Purpose                                         |
| -------------- | ----------------------------------------------- |
| `concurrently` | Runs the API and client together via `npm run dev` |

There is no CSS framework — all styling is hand-written in `client/src/styles/index.css`.

## Running the app

These steps are the **same on macOS, Windows and Linux** once Node and Postgres are installed.

### Quick start

Run these commands from the `birthday_memory` directory:

```bash
# Install all root, server, and client dependencies
npm run install:all

# Create the server environment file
cp server/.env.example server/.env

# Start Docker Desktop before running this command
npm run db:up

# Optional: insert sample birthdays
npm --prefix server run seed

# Start the API and frontend together
npm run dev
```

Open **http://localhost:5173** in your browser. The API runs at **http://localhost:4000**.

On Windows PowerShell, use this instead of the `cp` command:

```powershell
Copy-Item server\.env.example server\.env
```

Keep the terminal running while you use the application. Press `Ctrl+C` to stop the API and frontend.

### 1. Get the code and install dependencies

```bash
git clone <repository-url>
cd birthday_memory
npm run install:all
```

`install:all` installs the root, server, and client packages in one pass.

### 2. Configure the database connection

macOS / Linux:

```bash
cp server/.env.example server/.env
```

Windows PowerShell:

```powershell
Copy-Item server\.env.example server\.env
```

The defaults in that file already match the bundled Docker database, so you can leave it untouched if you use Docker. See [Configuration](#configuration) to point it elsewhere.

### 3. Start PostgreSQL

If you are using Docker on macOS or Windows, open **Docker Desktop** and wait until it reports that Docker is running. Verify the Docker engine before starting the database:

```bash
docker info
```

If you see an error such as:

```text
failed to connect to the docker API
```

Docker Desktop is not running yet. Start it, wait a few seconds, and run `docker info` again.

**Option A — bundled Docker database (recommended).** No Postgres install needed, and the `thebirthdates` database is created for you:

```bash
npm run db:up
```

Confirm that the database container is running:

```bash
docker compose ps
```

The service should be listed as `postgres`.

**Option B — your own PostgreSQL.** Create the database, then update `server/.env` with your host, port, user and password:

```bash
createdb thebirthdates
# or:  psql -U postgres -c "CREATE DATABASE thebirthdates;"
```

The `birthdays` table and its indexes are created automatically when the API starts — you only need the empty database to exist.

### 4. Add sample data (optional)

```bash
npm --prefix server run seed
```

Inserts eight well-known people, several with birthdays in the next few days so the countdown and calendar have something to show.

### 5. Start the app

```bash
npm run dev
```

This runs the API on **http://localhost:4000** and the web client on **http://localhost:5173** at the same time.

Open **http://localhost:5173** in your browser.

You can verify that the API can reach PostgreSQL with:

```bash
curl http://localhost:4000/api/health
```

Expected response:

```json
{"status":"ok","database":"thebirthdates"}
```

Press `Ctrl+C` in the terminal running `npm run dev` to stop the API and frontend.

When you are finished, stop the database container with:

```bash
npm run db:down
```

## Troubleshooting

**`command not found: node` / `'node' is not recognized`**
Node isn't installed or isn't on your `PATH`. Reinstall it for your OS above and open a new terminal — `PATH` changes don't apply to already-open terminals.

**`Cannot reach the server. Is the API running?`**
The client loaded but the API didn't. Check the terminal running `npm run dev` for errors, and confirm http://localhost:4000/api/health responds.

**`[db] could not initialize schema` / `ECONNREFUSED`**
Postgres isn't reachable. If using Docker, run `docker compose ps` — the container should say `healthy`. If using your own instance, confirm the host, port, user and password in `server/.env` and that the `thebirthdates` database exists.

**`failed to connect to the docker API`**
The Docker daemon isn't running. Start Docker Desktop (macOS/Windows) or `sudo systemctl start docker` (Linux).

**`Port 5173 is already in use` / `EADDRINUSE`**
Something else has the port. Stop it, or change the client port in `client/vite.config.js` and the API port via `PORT` in `server/.env`.

**`permission denied while trying to connect to the Docker daemon` (Linux)**
Your user isn't in the `docker` group. Run `sudo usermod -aG docker $USER`, then log out and back in.

## Configuration

`server/.env` accepts either a single connection string or individual settings:

```bash
# Option A — connection string wins if present
DATABASE_URL=postgresql://user:password@host:5432/thebirthdates

# Option B — individual settings (defaults match docker-compose.yml)
PGHOST=localhost
PGPORT=5544
PGUSER=birthday
PGPASSWORD=birthday
PGDATABASE=thebirthdates

PGSSL=false   # set to "true" for managed Postgres that requires SSL
PORT=4000
```

The database itself must exist; the `birthdays` table and its indexes are created automatically on server start.

## Database schema

Table `birthdays` in database `thebirthdates`:

| Column       | Type          | Notes                        |
| ------------ | ------------- | ---------------------------- |
| `id`         | `uuid`        | primary key, auto-generated  |
| `first_name` | `text`        | required                     |
| `last_name`  | `text`        | required                     |
| `birthdate`  | `date`        | required                     |
| `phone`      | `text`        | optional                     |
| `email`      | `text`        | optional                     |
| `created_at` | `timestamptz` | defaults to `now()`          |
| `updated_at` | `timestamptz` | refreshed on update          |

## API

| Method   | Path                            | Description                                |
| -------- | ------------------------------- | ------------------------------------------ |
| `GET`    | `/api/health`                   | Connectivity check                         |
| `GET`    | `/api/birthdays?q=&month=`      | List, optionally filtered by text or month |
| `GET`    | `/api/birthdays/upcoming?days=` | Soonest celebrations (default 30 days)     |
| `POST`   | `/api/birthdays`                | Create                                     |
| `PUT`    | `/api/birthdays/:id`            | Update                                     |
| `DELETE` | `/api/birthdays/:id`            | Delete                                     |

Responses use camelCase and include computed `age`, `turningAge`, `daysUntil` and `nextBirthday` fields. Validation failures return `422` with a per-field `errors` object.

The API accepts requests from any HTTP client, so you can explore it with `curl`, Postman, Insomnia, or your browser:

```bash
curl http://localhost:4000/api/birthdays

curl -X POST http://localhost:4000/api/birthdays \
  -H 'Content-Type: application/json' \
  -d '{"firstName":"Jane","lastName":"Doe","birthdate":"1995-03-22","phone":"+1 555 123 4567","email":"jane@example.com"}'
```

## Inspecting the database

To browse the data with a GUI such as **DBeaver**, **TablePlus**, **pgAdmin** or **Postico**, use these settings (they match the bundled Docker database):

| Setting  | Value            |
| -------- | ---------------- |
| Host     | `localhost`      |
| Port     | `5544`           |
| Database | `thebirthdates`  |
| Username | `birthday`       |
| Password | `birthday`       |

Note the port is **5544**, not the default 5432 — chosen so it never collides with a PostgreSQL instance you already run. Most GUI tools pre-fill 5432, so remember to change it.

To use the command line instead, the container already includes `psql`:

```bash
docker exec -it thebirthdates-db psql -U birthday -d thebirthdates
```

Once connected: `\dt` lists tables, `\d birthdays` shows the schema, `\q` quits.

## Project structure

```
birthday_memory/
├── client/                     # React front end
│   ├── index.html
│   ├── vite.config.js          # Dev server + /api proxy to the backend
│   └── src/
│       ├── App.jsx             # Layout, nav, hero stats
│       ├── main.jsx            # React entry point
│       ├── components/
│       │   ├── AddSection.jsx      # Section 1 — add a birthday
│       │   ├── SearchSection.jsx   # Section 2 — search, filter, edit, delete
│       │   ├── CalendarSection.jsx # Section 3 — month grid + countdown
│       │   ├── BirthdayForm.jsx    # Shared add/edit form
│       │   ├── PersonCard.jsx      # Reusable person card
│       │   └── Toast.jsx           # Notifications
│       ├── hooks/useBirthdays.js   # Data fetching and cache
│       ├── lib/api.js              # API client
│       ├── lib/utils.js            # Dates, colors, calendar grid
│       └── styles/index.css        # All styling
├── server/                     # Express API
│   ├── .env.example            # Copy to .env and edit
│   └── src/
│       ├── index.js            # Server entry point
│       ├── db.js               # Connection pool + schema bootstrap
│       ├── routes.js           # REST endpoints
│       ├── validate.js         # Server-side validation
│       ├── dates.js            # Age / next-birthday math
│       └── seed.js             # Sample data
├── docker-compose.yml          # Bundled PostgreSQL 16
└── package.json                # Root scripts
```

## Scripts

| Command                       | What it does                          |
| ----------------------------- | ------------------------------------- |
| `npm run dev`                 | Run API and client together           |
| `npm run dev:server`          | API only, with watch mode             |
| `npm run dev:client`          | Vite dev server only                  |
| `npm run build`               | Production build of the client        |
| `npm start`                   | Run the API without watch mode        |
| `npm run db:up` / `db:down`   | Start / stop the bundled Postgres     |
| `npm --prefix server run seed`| Insert sample birthdays               |

## Deploying

Build the client with `npm run build` and serve `client/dist` from any static host, pointing it at the API with `VITE_API_URL`. Run the API with `npm start` and a `DATABASE_URL` for your managed Postgres.
