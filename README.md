# n8n on Miabi: queue-mode stack

Five apps in one stack, created from `docker-compose.yml` with **Stacks → Import Compose**.

| App | Image | Role | Volume | Deploy strategy |
|---|---|---|---|---|
| `n8n-postgres` | `postgres:17.12-alpine` | Workflows, credentials, executions, binary data | `n8n-pg-data` | **recreate** |
| `n8n-redis` | `redis:8.10.3-alpine` | Execution queue (Bull) | `n8n-redis-data` | **recreate** |
| `n8n` | `n8nio/n8n:2.48.4` | Editor, API, triggers, webhooks, the only public app | `n8n-n8n-data` | **recreate** |
| `n8n-worker` | `n8nio/n8n:2.48.4` (`worker`) | Runs every execution | none | rolling |
| `n8n-runners` | `n8nio/runners:2.48.4` | Code-node sandbox for the worker | none | rolling |

Traffic flow: a webhook or trigger reaches `n8n`, which queues the execution ID in Redis. `n8n-worker`
picks it up, runs it against Postgres, and sends Code-node work to `n8n-runners`. In queue mode n8n
keeps binary data in Postgres, so the worker needs no shared volume.

> Just want n8n with no tuning? The Marketplace has a one-click n8n template. This stack is for
> queue mode, a sandboxed Code node and full control over versions.

---

## 1. Create three vault secrets

Generate three values and add them to the workspace secret vault with these exact names:

```bash
openssl rand -hex 32   # N8N_ENCRYPTION_KEY
openssl rand -hex 24   # N8N_DB_PASSWORD
openssl rand -hex 32   # N8N_RUNNERS_AUTH_TOKEN
```

**Back up `N8N_ENCRYPTION_KEY` outside Miabi.** It encrypts every credential stored in n8n. If you
lose or change it, every saved credential becomes unreadable.

## 2. Edit, then import

1. In `docker-compose.yml`, replace `UTC` (`GENERIC_TIMEZONE` and `TZ`, on `n8n` and
   `n8n-worker`). There's no hostname in the file: Miabi generates it in step 5.
2. **Stacks → Import Compose**: name `n8n`, pick the location, paste the file.
3. The import report should list 5 apps and 3 volumes, with nothing skipped.

Nothing is deployed yet.

## 3. Check what the importer created

- **Environment tab on `n8n`, `n8n-worker`, `n8n-runners` and `n8n-postgres`:** each secret
  variable must read `${{ secrets.… }}` exactly. The importer is documented as not expanding
  `${VAR}`, so the reference should come through intact. If any value is blank or changed, set it
  again by hand to the same `${{ secrets.NAME }}` reference.
- Mark nothing else as secret. The other values are plain configuration.

## 4. Per-app settings (Settings tab)

**Deployment strategy:** use `recreate` for `n8n-postgres`, `n8n-redis` and `n8n`. The default
rolling switch briefly runs old and new side by side, which is unsafe for these three:

- Two Postgres processes must never open one data directory at the same time.
- Two Redis processes would both write the same AOF file.
- Two n8n mains would both fire scheduled triggers.

Leave the worker and runners on rolling. **Never use canary for any of these apps.**

**Healthcheck.** "Image default" doesn't make a deploy wait, so set these:

| App | Type | Check | Start period |
|---|---|---|---|
| `n8n-postgres` | Command | `pg_isready -U n8n -d n8n` | 30 s |
| `n8n-redis` | Command | `redis-cli ping` | 10 s |
| `n8n` | HTTP | path `/healthz`, port `5678` | 120 s (first boot runs DB migrations) |
| `n8n-worker` | HTTP | path `/healthz`, port `5678` | 60 s |
| `n8n-runners` | Image default | none | none |

**Resources:** starting points to adjust after a week in Monitoring. Rolling apps need headroom
for the brief overlap during a deploy.

| App | CPU | Memory |
|---|---|---|
| `n8n-postgres` | 1 | 1 GiB |
| `n8n-redis` | 0.5 | 256 MiB |
| `n8n` | 1 | 1 GiB |
| `n8n-worker` | 2 | 2 GiB |
| `n8n-runners` | 1 | 1 GiB |

## 5. Expose `n8n` (only this app)

**Default: the generated URL from `MIABI_EXTERNAL_BASE_DOMAIN`.**

1. **Check the base domain is live.** On the default cluster's page, the External access section
   should show your domain, read-only because the environment variable pins it. A wildcard
   record `*.<base-domain>` must point at the gateway, and port 80 must be reachable for HTTP-01
   certificates (unless `MIABI_EXTERNAL_BASE_PROVIDER` names a DNS provider).
2. On `n8n`, go to **Network → External access**, tick `5678`, then **Save external access**.
   Miabi generates `<app-handle>-<token>.<base-domain>`, creates the route and requests the
   certificate.
3. Give n8n that hostname. Miabi has no variable that passes an app its own generated hostname
   (even Marketplace templates ask the installer for it), so paste it once. Build the three lines:

   ```bash
   H=n8n-a1b2c3.apps.example.com   # the generated hostname, bare
   printf 'N8N_HOST=%s\nN8N_EDITOR_BASE_URL=https://%s/\nN8N_WEBHOOK_URL=https://%s/\n' "$H" "$H" "$H"
   ```

   Then go to **Stacks → n8n → Shared environment → Import .env** and paste them as plain
   variables, not secrets. They reach `n8n` and `n8n-worker`, and the other three apps ignore them.
4. If the card refuses because the app has never been deployed, carry on with step 6. Then come
   back, enable external access and paste the variables. The stack page then offers **Redeploy**
   for the members running older configuration. Redeploy `n8n` and `n8n-worker`.

If you skip step 3, n8n still serves the editor and receives webhooks at the generated URL, because
those use the request path. But every URL n8n builds itself points at `localhost`:

- the webhook URLs shown in trigger nodes
- Wait and "send and wait" resume links
- OAuth2 credential callbacks
- invite and password-reset emails

The hostname stays with the app through renames and redeploys, and through turning external access
off and on again. It changes if the cluster's base domain changes or the app moves to another
location. In either case, update the three shared variables, redeploy `n8n` and `n8n-worker`, and
re-register webhook URLs with any third parties.

Limits of the generated route: it can't carry middlewares such as auth, rate limits or IP rules
(only maintenance mode). It also needs Goma Gateway; Miabi's Traefik setup doesn't support it.

**Alternative: your own domain.** Use this if you need middlewares or a name you choose.

1. Under **Networking → Domains**, add and verify the domain (TXT record).
2. Create an A/AAAA record for your hostname pointing at the gateway node.
3. On `n8n`, go to **Routes → Add route**: your hostname, path blank, target port `5678`, TLS
   `acme`.
4. Set the same three shared variables to that hostname.

## 6. First deploy, in order

Ideally the three shared hostname variables from step 5 are in place before `n8n` and
`n8n-worker` deploy. If they're added later, they take effect on the next redeploy of those two.

The importer drops `depends_on`, and **Deploy all** starts every app at once. For the first
deploy, go one app at a time and wait for each to turn healthy:

1. `n8n-postgres` and `n8n-redis`
2. `n8n`: creates the schema. Open the URL and create the owner account.
3. `n8n-worker`
4. `n8n-runners`

Smoke test: run a workflow containing a Code node from the editor. It should succeed, and
`n8n-worker`'s Logs tab should show it picking up the execution.

After the first boot, Deploy all and Redeploy are fine for config changes.

---

## Running it

- **Upgrades:** change the tag on `n8n`, `n8n-worker` **and** `n8n-runners` together, since
  n8n requires the runners image to match. Deploy `n8n` first (it runs migrations), then the worker,
  then the runners. Rollback (Releases → Activate) restores the image only; n8n migrations aren't
  reversible, so take a Postgres backup before every upgrade.
- **Scaling:** don't raise `n8n-worker` replicas. `n8n-runners` connects to one worker by name, so
  extra replicas would have no runner of their own, and their Code-node tasks would stall. To add capacity, add another
  `n8n-worker-2` + `n8n-runners-2` pair (the runner's broker URI points at `n8n-worker-2:5679`), or
  raise the worker's concurrency (default 10).
- **Rotating the DB password:** run `ALTER USER n8n PASSWORD '…'` in Postgres first, then update
  `N8N_DB_PASSWORD` in the vault, then redeploy `n8n` and `n8n-worker`. `POSTGRES_PASSWORD` is
  only read when the data directory is first created.
- **Never rotate `N8N_ENCRYPTION_KEY`.**
- **Editor shows "Connection lost":** if the gateway isn't passing websockets, add
  `N8N_PUSH_BACKEND=sse` to `n8n` and redeploy.
- **Backups:** `n8n-pg-data` holds everything that matters, plus the encryption key you saved in
  step 1.
- **Redis has no password.** It isn't routed or published, but any app in the workspace can reach
  `n8n-redis:6379` by name. If the workspace holds untrusted apps, put n8n in its own workspace.
- **Into Git:** once the stack runs, use **Settings → GitOps manifest → Generate manifest** on each app
  to capture it as `miabi.io/v1` manifests. These can also carry `deployment.strategy`,
  health checks and resources, which the compose import can't.
