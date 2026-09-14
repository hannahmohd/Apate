# Apate / Mirage / Chronos

Apate is a research SSH honeypot with a focused Ubuntu command emulator, a FUSE filesystem backed by Redis, and local AI artifact generation. Mirage is the product name; Chronos is the implementation package.

This remains a research MVP. Removing subprocess execution closes one escape route; it does not establish a hardened container or a complete Ubuntu emulator.

## Actual execution path

An SSH connection gets a persistent worker process. The gateway registers its PID and session UUID in Redis. Emulated commands issue filesystem calls against the FUSE mount. FUSE resolves the caller PID and selects that session's Redis filesystem namespace, initialized from `config/ubuntu.yaml`.

A session cannot read or mutate another session's filesystem. FUSE data and attribute caching are disabled to preserve that boundary. Session filesystem namespaces are ephemeral and normally removed after disconnect; audit evidence is stored separately in PostgreSQL.

Known AI-backed files generate on their first read. The generator claims a Redis lease, applies policy, validates output, and atomically commits only if the inode has not changed and it still owns the lease. Writes and deletion supersede generation. Listings do not generate content; attacker-created files remain empty until written, even if they reuse a manifest filename.

The model receives trusted profile facts, not arbitrary session history or environment variables. Its output is file bytes only.

## Run locally

Requires Linux FUSE support through Docker and Docker Compose:

```sh
docker compose up --build -d
ssh -p 2222 ubuntu@localhost
```

Ports bind to localhost by default. PostgreSQL must be available at startup. Ollama must have the models named in `config/generation_policy.yaml` installed to exercise real inference; otherwise eligible files receive a bounded fallback. The repository does not automatically download models.

Examples:

```sh
cat /etc/passwd
echo example > /tmp/note
cat /tmp/note
cat /etc/nginx/nginx.conf
```

The shell supports a limited subset of common commands and flags. SSH exec channels, arbitrary binaries, interpreters and network tools are rejected.

## Verification

Install `requirements.txt`, pytest, pytest-asyncio, Redis server, and the native FUSE library, then:

```sh
PYTHONPATH=src python3 -m pytest tests -q
docker compose config --quiet
```

Regression tests start disposable Redis instances over Unix sockets. Seven legacy tests are skipped by default because they mutate localhost services or stop named Docker containers; opting in with `--legacy-infrastructure` requires a disposable environment.

`tests/integration/verify_linux_ssh.py` exercises real Linux SSH workers, FUSE, session isolation, truncation, and optionally PostgreSQL evidence inside a disposable container. See `docs/HARDENING_REVIEW.md` for findings, verification and remaining gaps.

## Scope and limits

HTTP gateway code has been removed. Rust Layer 0 still exists under `src/chronos/layer0` but is not integrated into deployment. The native dashboard and world simulation remain experimental.

Current limits include 32 accepted connections, 8 KiB command input, 1,000 commands per session, 1 MiB files, a conservative 16 MiB write allocation budget per session, bounded generation admission, and per-session/global inference quotas. These are safeguards, not load-test certification.

Outstanding work includes fuller shell/permission semantics, durable audit delivery across crashes, orphan namespace recovery after process death, reproducible image/model pinning, and a stronger runtime isolation boundary. Default credentials and mutable image tags are for local research only.

## License

[MIT](LICENSE).
