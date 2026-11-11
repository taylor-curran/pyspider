# pyspider threat model

Repository: `taylor-curran/pyspider` (fork of binux/pyspider, a web crawler framework). No `SECURITY_MODEL.md` existed; this model is drafted from the code. Statements marked *(user)* come from Taylor's answers (2026-11-11); everything else is inferred from code.

## System overview

pyspider is a distributed crawler made of cooperating Python services (`pyspider/run.py` subcommands) that talk over a message queue and XML-RPC, with pluggable databases. Deployment reference: `Dockerfile` (python:3.6, PhantomJS 2.1.1, Node 8 + puppeteer) and `docker-compose.yaml` (all services on a `pyspider` Docker network, only WebUI published as `5050:5000`). No separate infrastructure repository was found.

## Components

| Component | Code | Notes |
|---|---|---|
| **WebUI** | `pyspider/webui/` (except `webdav.py`), `pyspider/webui/templates`, `static` | Flask app on Tornado; binds `0.0.0.0:5000` by default (`run.py --host`). Project editor, debugger (`/debug/<project>/run`, `/save`), task/result views, result dump (`/results/dump/<project>.<fmt>`), `/run`, `/update`. |
| **WebDAV** | `pyspider/webui/webdav.py` | Mounted at `/dav`; read/write project scripts. Own auth controller. |
| **Scheduler** | `pyspider/scheduler/` | XML-RPC on `0.0.0.0:23333` (`newtask`, `send_task`, `update_project`, `_quit`, `webui_update`, ...), no auth. |
| **Fetcher** | `pyspider/fetcher/` (`tornado_fetcher.py`, `phantomjs_fetcher.js`, `puppeteer_fetcher.js`, `splash_fetcher.lua`) | Fetches arbitrary URLs; XML-RPC on `0.0.0.0:24444` (`fetch`, `_quit`), no auth. JS renderers listen on 25555 (PhantomJS) and 22222 (puppeteer/express). |
| **Processor** | `pyspider/processor/`, `pyspider/libs/` (`base_handler.py`, `response.py`, `utils.py`, ...) | Loads project scripts from projectdb and executes them as Python modules (`project_module.py`); parses fetched pages. |
| **Result worker & storage** | `pyspider/result/`, `pyspider/database/` (sqlite, mysql, sqlalchemy, mongodb, redis, elasticsearch, couchdb, local), `pyspider/message_queue/` | taskdb / projectdb / resultdb and queues (RabbitMQ, Redis, Kombu, Beanstalk). |
| **Deployment** | `Dockerfile`, `docker-compose.yaml`, `config_example.json`, `run.py` CLI defaults, `requirements.txt` | Pinned, old dependencies (Flask 0.10, Jinja2 2.7, tornado 4.5.3, Flask-Login 0.2.11, requests 2.24, lxml 4.3.3, PhantomJS 2.1.1, Node 8). |

## Trust boundaries and entry points

1. **Browser → WebUI / WebDAV** (port 5000, published). Auth only when started with `--need-auth`; default is **off** (`run.py` `need_auth=False`). When on, HTTP Basic is compared in plaintext against `webui_username`/`webui_password` (`webui/login.py`, `webui/webdav.py`). `app.secret_key = os.urandom(24)` per process. No CSRF protection visible on POST routes.
2. **WebUI user → host code execution.** Any WebUI/WebDAV user can write a project script and run it (`/debug/<project>/run`, processor `build_module`). *Inferred: this is the product's purpose; WebUI access ≈ code execution as the pyspider process.*
3. **Internal network → XML-RPC / renderer ports** (23333, 24444, 25555, 22222). No auth; default bind `0.0.0.0`. In docker-compose these are only on the Docker network. Fetcher `fetch` RPC is an open fetch proxy; scheduler `newtask` injects tasks.
4. **Crawled websites → fetcher / processor / WebUI.** Page content, headers, redirects and cookies are untrusted. Rendered in the WebUI debugger and result pages (XSS risk), parsed by lxml/pyquery, rendered by PhantomJS/Chromium.
5. **Services → databases / message queue.** Credentials come from config/CLI (`config_example.json` shows placeholders; compose MySQL uses empty root password).

## Assets

- The host / containers running pyspider (code execution = full compromise of crawler infra and its network position).
- Project scripts in projectdb (may embed site credentials, cookies, API keys).
- Crawled results in resultdb.
- WebUI credentials (`webui_password`).
- Internal network reachable from the fetcher (SSRF target).

## Deployment facts *(user)*

- The WebUI is **internet-facing, behind `--need-auth` login**. *(user)*
- WebUI users are **not all trusted**: several users with different trust levels; **no user should be able to run code on the host or reach another user's projects**. *(user)* Note: pyspider has a single shared `webui_username`/`webui_password` and no per-user project ownership, and project scripts run as Python in the processor/WebUI debugger, so the stock design does not meet this requirement.
- Reachability of internal services (XML-RPC 23333/24444, renderers 25555/22222, DB, MQ) from untrusted networks is **unknown — assume they may be reachable**. *(user)*
- **No asset is prioritised over others** — weigh everything equally. *(user)*
- `docker-compose.yaml` is representative of the deployment; there is no separate infrastructure repository. *(user)*

## Attacker profiles

- **A1 Unauthenticated internet user** reaching the WebUI login boundary. In scope *(user: internet-facing)*.
- **A2 Malicious website** being crawled (stored XSS in WebUI, parser/renderer exploits, redirects to internal hosts).
- **A3 Network-adjacent attacker** reaching XML-RPC/renderer/DB/MQ ports. In scope *(user: assume reachable)*.
- **A4 Lower-trust authenticated WebUI user** — in scope *(user)*: must not get host code execution or access to other users' projects. Script execution is **not** accepted risk.
- **A5 Cross-site attacker** driving a logged-in operator's browser (CSRF against `/debug/*/save`, `/run`, `/update`).

## Existing controls

- Optional HTTP Basic auth (`--need-auth`), applied in `before_request` and the WebDAV controller.
- Project `lock` group makes WebDAV scripts read-only for unauthenticated users (`webdav.py` `readonly`).
- `verify_project_name` on project names; parameterized SQL in the SQL backends (placeholders), with some `%`-formatted identifiers (table/database names).
- docker-compose keeps internal services off published ports.

## Out of scope / accepted risk

- Nothing ruled out by the user. Script execution by WebUI users is explicitly **in scope**, not accepted risk *(user)*.

## Open questions

1. Which internal ports are actually exposed beyond the Docker network in production?
2. Is there a reverse proxy / TLS in front of the WebUI (Basic auth over plain HTTP otherwise)?
