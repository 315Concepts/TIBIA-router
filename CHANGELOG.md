# Changelog

All notable changes to this project are documented here. The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/).

## [1.0.0] - 26-10-6

First stable release.

### Added

- _Traefik_ 3.7 stack in _Docker Compose_ with three entrypoints: public HTTP, public HTTPS and private HTTPS.
- Permanent HTTP to HTTPS redirect (`301`) on the public HTTP entrypoint.
- **Let's Encrypt** staging (`stg`) and production (`prd`) **ACME** resolvers, with bringup mode on staging by default.
- _Traefik_ dashboard on the private HTTPS port.
- Two `whoami` canary containers (subdomain-routed and path-routed) for validating DNS, certificates and routing.
- Dynamic _Traefik_ configuration in `config/dynamic.yml`.
- Environment-based configuration through `.env` and a git-ignored `.env.local`, created by `./init`.
- Helper scripts: `./init`, `./up [production]`, `./down`, `./clean` and `./verify [production]`.
- `./verify` checks containers, the HTTP redirect, canary routes, certificate issuer and dashboard, and exits non-zero on failure.
- `scripts/install-debian.sh` and `scripts/install-ubuntu.sh` to install _Docker Engine_ and the _Compose_ plugin.
- `scripts/configure-ufw.sh` to back up and apply a strict `ufw` ruleset.
- `scripts/create-network.sh` to create the shared `traefik_backend` network.
- `scripts/configure-swap.sh` to add a swap file sized at 100% of RAM (override with `SWAP_PERCENT`), persisted in `/etc/fstab`, with `vm.swappiness=10`.
- README covering configuration, entrypoints, firewall, certificates, verification, troubleshooting and security notes.
