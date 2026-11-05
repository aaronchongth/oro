---
sidebar_position: 2
---

# Quick Start

Get OpenRobOps running locally in under 10 minutes.

:::warning[Work in Progress]
These manual steps work in under 10 minutes. The `start-local-env.sh` helper (step 4) launches all services in one command; broader setup automation is still evolving.
:::


## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| **Node.js** | v24 | Via [nvm](https://github.com/nvm-sh/nvm) |
| **Meteor** | 3.5.3 | `npx meteor` |
| **Docker** | With Compose plugin | `docker compose` command |
| **Terraform** | Latest | [hashicorp.com](https://developer.hashicorp.com/terraform) |
| **tmux** | Any | Via your package manager |
| **pipx** | Any | [pipx.pypa.io](https://pipx.pypa.io/) (installs the ORO CLI, step 6) |

## 1. Clone the Repository

```bash
git clone https://github.com/OpenRobOps/oro.git
cd oro
```

## 2. Configure Sign-In and Generate Settings

OpenRobOps uses Terraform to generate `app/settings.json` and `ingest/settings.json` with randomized credentials for inter-service communication.

### Configure Authentication (`terraform/local.tfvars`)

At least **one authentication method** (`smtp_url`, Google OAuth, or GitHub OAuth) **must** be configured before you can sign in — without any configured, `public.oauthProviders` is empty and `http://localhost:3000/` displays **"Sign in to continue"** with no login controls.

Create `terraform/local.tfvars` with at least one sign-in method and list your address in `admin_emails` so your first sign-in is automatically granted the `admin` role (otherwise new accounts have no role and land on an access-request screen):

```hcl
# Administrators — users whose sign-in email matches one of these are
# granted the "admin" role automatically when their account is created.
admin_emails = ["you@example.com"]

# Email passwordless login (6-digit one-time codes)
# smtp_url = "smtp://USER:PASS@SOME_SMTP_SERVER:PORT"

# Google OAuth
# oauth_google_client_id = "your-client-id"
# oauth_google_secret    = "your-secret"

# GitHub OAuth
# oauth_github_client_id = "your-client-id"
# oauth_github_secret    = "your-secret"
# Note: a GitHub App must be granted the "Email addresses" (read-only) account
# permission, or the user's email cannot be retrieved on sign-in.
```

:::tip[Local sign-in without external OAuth or SMTP]
For local development, you can run a local catch-all SMTP inbox such as [Mailpit](https://github.com/axllent/mailpit) instead of registering an OAuth app:

```bash
docker run -d --name oro-mailpit -p 1025:1025 -p 8025:8025 axllent/mailpit
```

Then set in `terraform/local.tfvars`:

```hcl
smtp_url     = "smtp://localhost:1025/?ignoreTLS=true"
admin_emails = ["admin@local.dev"]
```

Sign in with `admin@local.dev` and read the 6-digit login code at **http://localhost:8025/**.
:::

### Generate Settings Files

```bash
# Preview what will be generated (optional)
./scripts/generate-settings.sh --plan

# Generate and write settings files
./scripts/generate-settings.sh
```

This creates `app/settings.json` and `ingest/settings.json` with MQTT credentials, encryption keys, and service URLs. (If `app` is already running when you re-run `generate-settings.sh`, restart `app` so Meteor reloads `app/settings.json`.)

Secrets are Terraform `random_*` resources persisted in state, so the script is
safe to re-run: it preserves the existing MQTT master password, encryption keys,
peer key, and robot API key. To force-rotate all secrets run
`./scripts/generate-settings.sh --clean` — note this invalidates credentials
already handed out to connected robots.

## 3. Install Dependencies

`ingest/` installs `@openrobops/iso21423` from GitHub Packages (`npm.pkg.github.com`, configured in `ingest/.npmrc`), which **requires authentication even for public packages**. Configure a GitHub **Personal Access Token (classic)** (`ghp_...`) with the `read:packages` scope in your user `~/.npmrc` (fine-grained `github_pat_...` tokens are not supported by `npm.pkg.github.com`; do not add tokens to the git-tracked `ingest/.npmrc`):

```bash
npm config set //npm.pkg.github.com/:_authToken="<your-github-classic-pat>"
# Or, if using the GitHub CLI:
# gh auth refresh -h github.com -s read:packages && npm config set //npm.pkg.github.com/:_authToken="$(gh auth token)"
```

Then install packages in both services:

```bash
cd app && npm install
cd ../ingest && npm install
cd ..
```

Note: on every start, `ingest/run.sh` syncs shared files (`oro.proto` and
shared JavaScript) from `app/` into `ingest/src/shared/` via `ingest/import.sh`,
so the `app/` directory must be present alongside `ingest/`.

## 4. Start All Services

The quickest way is the helper script, which launches all three services (each
via its `run.sh`) in separate tabs of a tmux session named `oro`:

```bash
./scripts/start-local-env.sh
```

This requires **tmux** to be installed. Stop everything with
`./scripts/stop-local-env.sh`.

Prefer to run them manually? Start each component in its own terminal:

```bash
# Terminal 1: MQTT broker
cd mqtt && ./run.sh

# Terminal 2: Web app (also starts MongoDB)
cd app && ./run.sh

# Terminal 3: Ingest service
cd ingest && ./run.sh
```

## 5. Open the Dashboard

Navigate to **http://localhost:3000/** in your browser and sign in using any
configured authentication method (OAuth or email).

If you're greeted with an access request screen, it means your email didn't
match any of the addresses listed in `admin_emails` — add it there (step 2) and
sign in again, or have an existing admin approve your account.

You now have a running OpenRobOps instance ready to accept robot connections.

## 6. Install and Configure the ORO CLI

Configuration (dashboards, data sources, actions, robot admissions...) is applied
as code with the [`oro` CLI](https://github.com/OpenRobOps/oro-cli). It talks to
the Config API on your behalf, so you need it for the next step and for most
guides in these docs.

Install it with pipx so it gets its own isolated Python environment:

```bash
pipx install oro-cli
```

Then create an API key in the web app under **Settings → API keys** (copy it
when shown, it is displayed only once) and register your local instance as a
context:

```bash
oro context add local --url http://localhost:3000/api --api-key <your-api-key>
oro whoami        # prints the context and "Authenticated as user ..."
```

The first context you add becomes the current one. You can register other ORO
instances the same way and switch between them with `oro use <name>`. See the
[CLI README](https://github.com/OpenRobOps/oro-cli#readme) for all commands.

## 7. (Optional) Connect a Simulated Robot with Flatland

Want to see data flowing without setting up real hardware? The
[Flatland simulator](https://github.com/OpenRobOps/sim-flatland) is a lightweight
2D simulator with a full Nav2 robot. It connects to ORO either through the InOrbit
ROS2 agent — publishing the same MQTT/protobuf telemetry as a real ORO agent (pose,
laser, costmap, system stats) — or as an [ISO 21423 robot](../iso21423/flatland-simulator.md),
so the dashboard lights up exactly as it would with a physical robot.

```bash
git clone https://github.com/OpenRobOps/sim-flatland.git
cd sim-flatland
# Follow the README to point the simulator at your local MQTT broker
```

In addition to starting the simulation, a suitable configuration for OpenRobOps should be
created. Configuration files (see Config API) are included in that repository; apply them
with the CLI from step 6 (`oro apply -f <file>`) following the instructions in its README.

Once the simulator is running, you should see a robot appear in the fleet view
at `http://localhost:3000/` with live localization, vitals, and laser data.

## Next Steps

- [Core Concepts](./core-concepts.md) — understand robots, telemetry, and dashboards
- [Connecting Your First Robot](./connecting-first-robot.md) — register a robot and see data flow
- [Flatland Simulator](https://github.com/OpenRobOps/sim-flatland) — drive the dashboard with a simulated robot
- [Deployment](../guides/deployment.md) — production deployment with Docker Compose
