# AGENTS.md

## Dependencies

Keep `elixir/mix.lock`, `elixir/assets/package-lock.json`, `website/package-lock.json`, and `rust/Cargo.lock` identical to the upstream release the fork is built on. Do not update dependencies in the fork; they arrive with the next upstream release, which the `teslamate-fork-upgrade` skill brings in. A diverging lockfile conflicts on every upgrade and changes the pinned Nix hash. For the same reason, the fork's Dependabot covers only Docker images and GitHub Actions.

Do not adopt a release until it has been public for the minimum age below, and do not bypass the cooldown (for example with `HEX_COOLDOWN=0d` or `--min-release-age=0`) unless the user explicitly asks for it.

| Ecosystem                          | Setting                                     |
| ---------------------------------- | ------------------------------------------- |
| Hex                                | `hex: [cooldown: "4d"]` in `elixir/mix.exs` |
| npm (`elixir/assets/`, `website/`) | `min-release-age=4` in `.npmrc`             |

When the user explicitly asks for a fork-only fix ahead of upstream, run Mix from `elixir/` inside the devenv shell (`direnv exec .`), check that every changed version meets the cooldown, then run `nix run .#update-nix-hashes` and commit the hash in `nix/flake-modules/package.nix` together with the lockfile.

## Production Data Access

Read the production TeslaMate data only through the scripts of the global `teslamate` skill: `bash ~/.agents/skills/teslamate/scripts/query.sh '<SQL>'` for SQL and `bash ~/.agents/skills/teslamate/scripts/api.sh '<path>'` for any Grafana GET. They call the Grafana HTTP API as the `teslamate-agent` service account, whose token chezmoi exports from 1Password. Never print the token or read the files that define it; to diagnose access, run a script and read its error. There is no direct database connection.

The access is read-only by design: the service account has the Viewer role, the `TeslaMate` datasource logs in as the SELECT-only `grafana_ro` role, and `query.sh` wraps each query in a `READ ONLY` transaction. Change production dashboards or data through this repository, a new image, and the Ansible deployment, never through the API.

Use it to check a dashboard query against real data, compare result shapes before and after a change, and measure plans with `EXPLAIN (ANALYZE, BUFFERS)`. `api.sh /api/dashboards/uid/<uid>` returns the deployed dashboard JSON, so compare it with `grafana/dashboards/` to confirm what is running.

Do not put production data in commits, pull requests, issues, or reports: no addresses, coordinates, geofence names, identifiers, or actual timestamps. Report aggregates, row counts, and plan shapes, and use anonymized parameters in any query you share.
