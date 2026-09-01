# n8n-vultr

Desired state for one **n8n** instance on Vultr, backed by a colocated
self-hosted **Neon** storage tier with layers and WAL in Cloudflare R2.

Live at **https://n8n.bigconfig.online**.

This repository holds no source code. `colors.yml` is the deployment; the
behaviour comes from the SHA-pinned [`getcolors/n8n`](https://github.com/getcolors/n8n)
Package Skill, whose launcher is installed here as `./green`.

## Use

```sh
direnv allow          # once, after populating .envrc.private
./green build         # render .colors/ — offline, no credentials
./green create        # converge
./green delete        # guarded; see below
```

## Credentials

In the gitignored `.envrc.private`:

| Variable | For |
|---|---|
| `COLORS_PAR_VULTR_API_KEY` | instance, firewall, SSH key |
| `COLORS_PAR_CLOUDFLARE_API_TOKEN` | the DNS record |
| `COLORS_PAR_R2_ACCESS_KEY_ID` / `_SECRET_ACCESS_KEY` | OpenTofu state |
| `COLORS_PAR_N8N_ENCRYPTION_KEY` | **≥32 chars, keep a copy off this machine** |

`.envrc` maps the R2 pair onto `COLORS_PAR_NEON_R2_*`, which is what the
imported storage-tier play reads. Supplying a dedicated `COLORS_PAR_NEON_R2_*`
pair takes precedence automatically and is the better posture.

Never export `COLORS_PAR_PROFILE`.

### Currently one credential does too much

The same R2 pair reaches OpenTofu state, live Neon data, and backups. Bucket
separation is done (`n8n-storage`, `n8n-backup`); credential separation is not,
so a compromised host can reach the backups that would survive it. Acceptance
gate **R2 reports `skip` with that reason** rather than passing quietly. Adding
`COLORS_PAR_N8N_BACKUP_R2_ACCESS_KEY_ID` and `_SECRET_ACCESS_KEY`, scoped to
`n8n-backup` only, turns the gate on with no code change.

## Server-generated secrets

The database role password, the n8n owner password and the task-runner token are
created once on the server and are not operator credentials:

```sh
ssh n8n-vultr "cat /etc/n8n/secrets/owner-password"       # the n8n login
ssh n8n-vultr "cat /etc/neon/secrets/neon_role_password"  # the database role
```

Because `cloudflare-proxied: true`, the public name resolves to Cloudflare's
edge. SSH needs the origin address, which the `~/.ssh/config` alias holds.

## Deleting

`compute-prevent-destroy: true` guards `delete`. Lift it for exactly one run:

```sh
COLORS_PAR_COMPUTE_PREVENT_DESTROY=false ./green delete
```

Never edit the committed flag.

## Recovery

Backups run every six hours to `n8n-backup`, as a set: a logical dump, a tar of
the n8n data directory, and a manifest with checksums and one shared timestamp.
The unit gracefully stops n8n and drains its database sessions across both
captures, so the pair is consistent rather than merely simultaneous.

The interval is the RPO, because a rebuilt safekeeper does not recover its
offloaded WAL. Verify and rehearse:

```sh
ssh n8n-vultr "/opt/neon/n8n-restore.sh <stamp> --verify-only"   # checksums
ssh n8n-vultr "/opt/neon/n8n-rehearsal.sh"                       # full restore drill
```

The rehearsal restores into an isolated scratch database and data directory,
boots the pinned image, logs in, and executes a workflow whose node carries a
stored credential — the only way to prove the encryption key survived.

## Operational drills

```sh
ssh n8n-vultr /opt/neon/n8n-smoke.sh                    # the acceptance gates
ssh n8n-vultr /opt/neon/n8n-soak.sh                     # load, declared thresholds
ssh n8n-vultr /opt/neon/n8n-prune-drill.sh              # retention, isolated
ssh n8n-vultr /opt/neon/n8n-restart-drill.sh recreate   # full stack recreate
```

A monitor timer writes a dead-man heartbeat object to `n8n-backup` every 15
minutes. Its *staleness* is the alert; turning that into a page needs an
external poller, which a single host cannot provide for itself.
