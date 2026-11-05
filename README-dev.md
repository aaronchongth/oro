# ORO Development environment

## Requirements

 * `node` v24 and `nvm`
 * Meteor 3.5.3
  ```
  npx meteor
  ```
* Terraform: https://developer.hashicorp.com/terraform
* `docker` with the `compose` plugin (`docker compose` command)
* `tmux`

## Prepare config files

Local configuration files require some credentials and keys for services to interact.
Some of these are randomized; and settings files are not Git-controlled.

To configure OAuth mechanisms, create a file `terraform/local.tfvars` with some of these contents (uncommenting and entering your secrets):

```
# If using Google auth, create a Google app and enter these values
# oauth_google_client_id = ""
# oauth_google_secret    = ""

# If using GitHub auth, create a GitHub app and enter these values.
# Note: a GitHub *App* must be granted the "Email addresses" (read-only)
# account permission, otherwise the user's email cannot be retrieved.
# oauth_github_client_id = ""
# oauth_github_secret    = ""

# If authenticating via email and passcodes, enter an SMTP URL.
# For local testing without external OAuth/SMTP, run Mailpit:
#   docker run -d --name oro-mailpit -p 1025:1025 -p 8025:8025 axllent/mailpit
# and set smtp_url = "smtp://localhost:1025/?ignoreTLS=true" (read codes at http://localhost:8025/).
# smtp_url = "smtp://USER:PASS@SOME_SMTP_SERVER:PORT"
```

In `/scripts`, use `generate-settings.sh` to generate settings files (for ingest and appserver). Secrets are managed by Terraform `random` resources and persist in state across runs, so it is safe to re-run.

 * `generate-settings.sh` — apply the Terraform configuration, preserving existing secrets (default).
 * `generate-settings.sh --plan` — preview changes without writing files.
 * `generate-settings.sh --clean` — regenerate all secrets from scratch.

You can check created files `app/settings.json` and `ingest/settings.json`; as well as further customize them.

## Run services (manually)

First make sure `npm` packages are installed. `ingest/` fetches `@openrobops/iso21423` from GitHub Packages (`npm.pkg.github.com`), which requires authentication even for public packages — configure a classic GitHub PAT (`ghp_...`) with `read:packages` scope in your user `~/.npmrc` (not `ingest/.npmrc`):

```bash
npm config set //npm.pkg.github.com/:_authToken="<your-github-classic-pat>"
```

Then under `/app` and `/ingest`, run `npm i` (needed once unless packages are added).

Then to run all services, just run `start-local-env.sh`.

Alternatively, run components manually. Each of these on its own terminal window:

 * in `/mqtt`: `docker compose up`
 * in `/app`: `./run.sh`. Note: this runs both meteor and a mongodb instance (for now)
 * in `/ingest`: `./run.sh`

Finally point your browser to http://localhost:3000/

## Environment overview

The following components are run by the previous scripts:

| Name | Description | Port |
|------|-------------|------|
| Mosquitto (MQTT) | MQTT broker; auth via Meteor MongoDB | 1883 (MQTT), 9001 (WebSockets) |
| Web / App | Meteor app + MongoDB (`./run.sh` in `/app`); main UI | 3000 |
| MongoDB | Database used by Meteor (run via `/app` run.sh) | 3001 |
| Ingest | Ingest service (`./run.sh` in `/ingest`); connects to MQTT and MongoDB | — |
| ISO 21423 (optional) | ISO-native robots join the fleet via ingest, disabled by default — see `docs/iso21423-robots.md` | — |
| Floor plans (optional) | Draw robots on a pretty map: `SpatialAnnotation` + `SpatialTransformation`, fitted with `tools/align_maps.py` — see `docs/map-alignment.md` | — |
| Ghost robots (optional) | Static demo robots, online with datasources and a pose but no live data: the `Robot` config kind — see `docs/ghost-robots.md` | — |

## Instance initial setup

1. User/Role

If your sign-in email was listed in `admin_emails` in `terraform/local.tfvars` before creating your account, you are granted the `"admin"` role automatically. Otherwise, new users receive no role and see a "Please contact your team admin" dialog; to bootstrap manually, open MongoDB (`localhost:3001`), use the `meteor` database, find the `users` collection, and add `"admin"` to `userRoles` on your user document.
