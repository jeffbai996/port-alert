# port-alert (archived)

> **Archive notice — this repository is not a working portfolio-alert application.** It preserves an early project brief for a deterministic IBKR account-monitoring tool; the described Python engine, configuration, checks, and alert backends were never implemented here.

## Status

Archived on 2026-08-24. Do **not** rely on this repository to monitor an account, send trading alerts, or manage portfolio risk.

The repository currently contains:

- `CLAUDE.md` — the historical design brief and proposed architecture.
- `test.sh` — a minimal shell smoke test only; it does not test an alerting application.
- `.gitignore` — initial ignore rules.

There is no Python package, service entry point, dependency manifest, configuration template, IBKR integration, persistence layer, or alert-delivery implementation in this repository.

## Original concept

The proposed tool would have run on a schedule during market hours, queried an IBKR gateway, evaluated deterministic thresholds, deduplicated alerts, and sent notifications through a file, webhook, or email backend. That concept was not developed into a usable implementation in this repository.

## Smoke test

The only runnable file is a shell sanity check:

```sh
sh test.sh
```

Its successful output confirms only that the checked-out script can run; it is **not** an application health check.

## Related projects

- [ibkr-terminal](https://github.com/jeffbai996/ibkr-terminal)
- [ibkr-terminal-core](https://github.com/jeffbai996/ibkr-terminal-core)

## License

No license file is included.