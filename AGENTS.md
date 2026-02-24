# AGENTS.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

This is **Goat Bot V2** (shizuka-mongodb variant), a Facebook Messenger chatbot built on Node.js using an unofficial Facebook Chat API (`fb-chat-api/`). It supports commands, events, a web dashboard, multiple database backends (MongoDB, SQLite), and i18n (English/Vietnamese).

## Build and Run Commands

- **Install dependencies:** `npm install`
- **Start (production):** `npm start` (runs `node index.js`, which spawns `Goat.js` as a child process)
- **Start (development):** `npm run dev` (uses `config.dev.json` and `*.dev.js` variants)
- **No test suite or linter scripts are configured.** ESLint is installed as a devDependency but no `lint` script exists in `package.json`.
- **Node.js version:** 18.x (per `engines` in `package.json`)

## Architecture

### Boot Sequence

1. `index.js` — Entry point. Spawns `Goat.js` as a child process; restarts it on exit code 2. Also starts a simple Express uptime server on port 3000.
2. `Goat.js` — Main orchestrator. Validates JSON configs, initializes global state (`global.GoatBot`, `global.db`, `global.client`, `global.utils`, `global.temp`), sets up config file watchers, auto-restart scheduling, then delegates to `bot/login/login.js`.
3. `bot/login/login.js` — (Obfuscated) Handles Facebook authentication (email/password, cookie, appstate, 2FA). After login, calls `bot/login/loadData.js` and `bot/login/loadScripts.js`.
4. `bot/login/loadData.js` — Connects to the database via `database/controller/index.js`, loads all thread/user/dashboard/global data into `global.db`.
5. `bot/login/loadScripts.js` — Scans `scripts/cmds/` and `scripts/events/`, auto-installs missing npm packages, registers commands/events/aliases into `global.GoatBot.commands` / `global.GoatBot.eventCommands`.

### Global State

All major runtime state lives on `global`:
- `global.GoatBot` — Commands map, event commands map, aliases, config, FCA API handle, bot ID, listening handle.
- `global.db` — Database models and data controllers (`threadsData`, `usersData`, `dashBoardData`, `globalData`), plus cached `allThreadData` / `allUserData` arrays.
- `global.utils` — Utility functions exported from `utils.js` (Google Drive ops, download helpers, translation, image uploads, etc.).
- `global.client` — Config file paths, cooldown tracking, database creation queues.

### Command System (`scripts/cmds/`)

Each command is a `.js` file exporting:
- `config` — `{ name, version, author, countDown, role, shortDescription, description, category, guide, aliases, envGlobal, envConfig }`
- `onStart(params)` — Required. Called when the command prefix + name is matched. Receives `{ api, args, message, event, threadsData, usersData, dashBoardData, globalData, threadModel, userModel, dashBoardModel, globalModel, role, commandName, getLang }`.
- Optional hooks: `onLoad`, `onChat` (runs on every message), `onFirstChat`, `onEvent`, `onAnyEvent`, `onReply`, `onReaction`.
- `langs` — Object with `vi` and `en` keys for i18n strings. Use `%1`, `%2` etc. as placeholders.
- Files ending in `.eg.js` are ignored (example/template files).
- Role values: 0 = normal user, 1 = group admin, 2 = bot owner.
- See `scripts/cmds/newcommand.eg.js` for the canonical template.

### Event System (`scripts/events/`)

Same module structure as commands but `config.category` must be `"events"`. The `onStart` function fires on group events (member join/leave, name change, etc.). See `scripts/events/newcommandevent.eg.js`.

### Event Dispatch (`bot/handler/`)

- `handlerAction.js` — Routes incoming FCA events by `event.type` to the appropriate handler (message → onStart/onChat/onReply; event → onEvent/handlerEvent; message_reaction → onReaction; etc.).
- `handlerEvents.js` — Resolves command name from message body, checks permissions/cooldowns/bans, then invokes the matched command's handler.
- `handlerCheckData.js` — Ensures thread/user records exist in the database before processing.

### Database Layer (`database/`)

- `database/controller/index.js` — Factory that connects to the configured DB type and returns data controllers.
- DB type is set in `config.json` → `database.type` (`"mongodb"`, `"sqlite"`, or `"json"`). This repo is configured for MongoDB.
- `database/controller/threadsData.js`, `usersData.js`, `dashBoardData.js`, `globalData.js` — CRUD operations with `get`, `set`, `create`, `getAll`, `remove`, `refreshInfo` methods.
- `database/models/mongodb/` and `database/models/sqlite/` contain Mongoose/Sequelize schemas.
- `database/connectDB/connectMongoDB.js` and `connectSqlite.js` handle connection setup.

### Dashboard (`dashboard/`)

Express web app (Eta templates) providing:
- User registration/login (Passport.js + bcrypt + Gmail OAuth2 for email verification)
- Thread management, custom commands, welcome/leave messages, rank-up settings
- Routes in `dashboard/routes/`, views in `dashboard/views/` (`.eta` templates)
- Runs on the port specified in `config.json` → `dashBoard.port` (default 3001)

### Facebook Chat API (`fb-chat-api/`)

Bundled fork of the unofficial Facebook Chat API. Key entry point is `fb-chat-api/index.js`. Individual API methods live in `fb-chat-api/src/` (e.g., `sendMessage.js`, `listenMqtt.js`, `getThreadInfo.js`). The bot listens via MQTT (`listenMqtt`).

### Configuration Files

- `config.json` — Main bot config: Facebook account credentials, database URI, prefix, admin IDs, language, dashboard settings, auto-restart, FCA options. Watched for live reloading.
- `configCommands.json` — Per-command environment variables (`envCommands`, `envEvents`), global env (`envGlobal`), and banned commands. Also watched for live reloading.
- `account.txt` — Facebook appstate/cookie for authentication (supports JSON array, cookie string, Netscape format, or email/password lines).

### Internationalization (`languages/`)

- `languages/en.lang` / `languages/vi.lang` — Key-value `.lang` files for system messages (format: `section.key = value`).
- `languages/cmds/en.js` / `languages/events/en.js` — Additional command/event-specific translations.
- `languages/makeFuncGetLangs.js` — Parses `.lang` files into `global.language` and exports the `getText(section, key, ...args)` function.
- Commands define their own `langs` object inline for per-command i18n.

### Utilities (`utils.js`)

Exports a large utility object including: Google Drive upload/download/delete, image upload (imgbb), stream helpers, translation API, time formatting, random string/number generation, custom error class, task queue, and the `message()` wrapper that provides `send`, `reply`, `unsend`, `reaction`, `err` methods for responding to chat events.

## Important Conventions

- The `bot/login/login.js` file is obfuscated — do not attempt to modify it directly.
- Process exit code `2` triggers an automatic restart (used by auto-restart and manual restart commands).
- Commands auto-install their npm dependencies at load time if missing from `node_modules/`.
- Config files (`config.json`, `configCommands.json`) are live-reloaded on file change — no restart needed for config updates.
- The `autoLoadScripts` feature (when enabled in config) watches `scripts/cmds/` and `scripts/events/` for file changes and hot-reloads commands without restarting the bot.
