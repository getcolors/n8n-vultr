# CLAUDE.md

Guidance for agents working in this deployment. Read
`~/code/getcolors/CLAUDE.md` first.

## What this is

Desired state only. No source code. `colors.yml` is the single file to edit;
everything else is either generated (`.colors/`), secret (`.envrc.private`), or
an installed copy of the Package Skill launcher.

## Things specific to this deployment

- **`colors.yml` carries `neon-*` keys deliberately.** The package renders the
  `getcolors/neon` templates from a SHA pin rather than copying them, so it must
  speak that package's vocabulary. Renaming them would fork the templates.
- **`.envrc` maps the R2 credential** onto `COLORS_PAR_NEON_R2_*`, which is what
  the imported storage-tier play reads via `lookup('env')`. Without that mapping
  the converge fails at *Refuse to converge with empty credentials* with
  `empty R2 key id`.
- **`vultr-http-sources: cloudflare`** is a symbolic source the package
  resolves at converge time, not a pinned list. The resolved set and its
  checksum are recorded under `.colors/`. It requires `cloudflare-proxied: true`
  — unproxied, ACME HTTP-01 arrives from Let's Encrypt directly and is
  firewalled, and the converge still succeeds while no certificate is ever
  issued. The validator refuses the combination.
- **Gate R2 reports `skip`,** not pass, while one R2 credential spans state,
  data and backups. That is a real weakness, named rather than hidden.

## Before any converge

```sh
./green build                                    # renders offline
ansible-playbook --syntax-check -i inventory.json site.yml   # in .colors/*/n8n-ansible
```

The syntax check is one second and offline. Several live converges were spent
on playbooks that failed at *load* time before this became routine.

## Never

- Edit `.colors/` — it is generated.
- Edit `compute-prevent-destroy` in committed state.
- Export `COLORS_PAR_PROFILE`.
- `golden:accept` upstream without reading the diff.
