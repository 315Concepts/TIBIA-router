<p align="center">
  <img src="docs/tibia-logo.png" alt="A cartoon gopher in a hi-vis vest and ear defenders, waving traffic wands while springing out of a Docker-whale jack-in-the-box" height="480">
</p>

# Traefik In A Box In Advance (TIBIA)

A production-ready _Traefik_ routing stack in _Docker Compose_ - with TLS certificates, HTTP-to-HTTPS redirects, and canary services already wired up.

## Overview

One job: put the latest version of _Traefik_ in front of your containers, with Let's Encrypt certificates and canary deployments included - so you aren't left hand-assembling the same compose file for the fifth time. TIBIA is a rock-solid but un-opinionated building block, it pairs natively with [SOCKS](https://github.com/315Concepts/SOCKS-server) or any other container that can sit on a _Docker_ network. What comes in the Box: _Traefik_, three entrypoints (public HTTP, public HTTPS, private HTTPS), staging and production **ACME** resolvers, a service dashboard, and two `whoami` canary containers all live in _less than 5 minutes_. Add in all the required tools to setup and validate the stack and suddenly a task that shouldn't be a big deal... isn't.

## Pre-Requisites

- _Docker Engine_ and _Compose_ plugin (scripts provided for **Debian** & **Ubuntu**)
- Both _curl_ and _openssl_, if not installed by default on your host
- A domain you control, with two DNS **A** records pointing at the host's public IPv4 address:
  - Base domain: _domain.example.com_
  - Wildcard subdomain: _*.domain.example.com_
- Ports reachable from where **ACME** needs them (public **80**/**443** access)

## Quick Start

1. Init the local environment by running `./gateway init` and then editing `.env.local` with `SERVICE_DOMAIN` and `ACME_CERTIFICATE_EMAIL` values
2. Optional, fresh host:
    - Install _curl_, _openssl_, and _Docker_ with `./scripts/install-debian.sh` or `./scripts/install-ubuntu.sh`
    - Review then run `./scripts/configure-ufw.sh`
    - Add swap equal to your RAM with `./scripts/configure-swap.sh`
3. Create the shared network: `./scripts/create-network.sh`
4. Start _Traefik_ in **BRINGUP MODE**: `./gateway up` which will bring _Traefik_ up with **ACME** Staging certificates and canary deployments active
5. Confirm routing works: `./gateway verify` - see [Verifying your setup](#verifying-your-setup)
6. Stop _Traefik_: `./gateway down && ./gateway clean` which will bring all containers down and delete **_ALL_** **ACME** certificate caches 
7. Start _Traefik_ in **PRODUCTION MODE**: `./gateway up production` which will bring _Traefik_ up by itself with live **ACME** Production certificates
8. Run `./gateway verify production` to validate the full production stack and you're ready to attach your own domain services!

## Static Configuration

All domain-specific static configuration for _Traefik_ is through environment variables. `.env` holds the defaults and `.env.local` holds your machine-specific values; `.env.local` is git-ignored and wins over `.env`. Running the `./gateway init` script creates `.env.local` from `.env.base`, if it doesn't exist yet. The helper scripts provided will load both files for you, the alternative is to `source` the environment files manually and then running additional _Docker Compose_ commands.

| Variable                 | Default                | Description |
| ------------------------ | ---------------------- | ----------- |
| `SERVICE_DOMAIN`         | *(required)*           | Base domain you control DNS for - scripts stop if it is unset  |
| `ACME_CERTIFICATE_EMAIL` | *(required)*           | Contact address registered with **Let's Encrypt** - scripts stop if it is unset  |
| `TRAEFIK_VERSION`        | `3.7`                  | _Traefik_ image tag  |
| `PRIVATE_HTTPS_PORT`     | `8443`                 | Port for the private HTTPS entrypoint (inc. the dashboard)  |
| `ACME_DEFAULT_RESOLVER`  | `stg`                  | **ACME** certificate resolver for the dashboard and canaries: `stg` or `prd`  |
| `TRAEFIK_SUBDOMAIN`      | `traefik`              | Dashboard host: `<TRAEFIK_SUBDOMAIN>.<SERVICE_DOMAIN>`  |
| `WHOAMI_SUBDOMAIN`       | `whosub`               | Subdomain-routed canary host. Bring-up aid; remove once your domain validates  |
| `WHOAMI_PUBLIC_PATH`     | `/whopath`             | Path-routed canary prefix. Bring-up aid; remove once your domain validates  |

- **Precedence:** a variable already set in your shell beats `.env.local`, which beats the defaults in `.env`.
- **Required Values:** the only two you must set are `SERVICE_DOMAIN` and `ACME_CERTIFICATE_EMAIL`. Everything else has a sensible, secure default.
- **Running Compose by Hand:** `source` both files first - plain `docker compose` doesn't read `.env.local`.

## Dynamic Configuration
_Traefik_ can load its own dynamic configuration from two places - TIBIA ships with the first and has the second ready to switch on.

| Approach                          | Where the config lives                              | Changing it |
| ------------------------------    | --------------------------------------------------- | ----------- |
| **_Docker_ labels** (default)       | Labels on the _Traefik_ container itself              | Recreate the container (`./gateway down && ./gateway up [production]`)|
| **File provider** (`dynamic.yml`) | One file, mounted into the _Traefik_ container        | Edit the file; _Traefik_ reloads it |

- **Switching:** Comment out the dynamic configuration labels in `docker-compose-traefik.yaml` and uncomment the `providers.file.filename` and `/etc/traefik/config` directory mounting lines, plus whatever [workload](#workload-configuration) changes are needed for their middlewares.
- **Labels are the more compact option for an Operator.** There is one central _Traefik_ configuration in `docker-compose-traefik.yaml` instead of carrying around an additional file and mount on the container.
- **The catch:** _Docker_ labels can't be modified after a container is created. Any change in what is nominally a dynamic configuration actually means dropping and recreating the container (`./gateway down && ./gateway up [production]`). Since we're talking about the _Traefik_ container itself, that change **_briefly takes your domain ingress offline_** - this shouldn't be a frequent occurrence but with certain traffic levels or shapes that may be an unacceptable operations pattern.
- **The file provider avoids that.** _Traefik_ watches the configuration and applies edits without a restart, and the definitions live in an external file mounted to the container. We mount the file's directory instead of the single file because some editors replace a file rather than modify it, and a single-file bind mount can keep pointing at the old copy.

Neither one is wrong. Labels suit compact stacks that change rarely or are always redeployed together, and the file method suits stacks where routing configurations change more often or that have an ingress where even momentary service interruption is unacceptable.

## Workload Configuration

Any container that joins the `traefik_backend` network and carries `traefik.*` labels gets routed - nothing else (`exposedbydefault=false`). The two `whoami` canaries in `docker-compose-debug.yaml` are working examples; copy/paste whatever fits your own workload needs.

- **Router names are global.** Router names must be unique across every container _Traefik_ sees - reusing one (e.g. the dashboard router is called `dashboard`) merges or breaks the routes.
- **Dynamic middleware binding.** When using file-based dynamic configuration for _Traefik_ there is an additional `@file` provider tag that must be added - it's included but commented out in the `whoami` path routing definition.
- **Changing routing labels requires a full removal.** Changing workload labels to adjust _Traefik_ routing means the container has to be completely removed - just like _Traefik_'s own dynamic configuration label issue. This can result in Operators trying to reconfigure deployed workloads and not seeing their changes reflected in _Traefik_.
- **Subdomain or path?** Prefer a subdomain per service (``Host(`app.example.com`)``) when the app needs to see its normal URLs and has no special path handling. Use a path (``Host(`example.com`) && PathPrefix(`/app`)``) when you prefer clamping DNS entries and only if the service copes with living under a prefix.
- **Public or private.** `web-secure` (443) is open to the internet. Use `private-secure` instead to put a service on the restricted port. See [Entrypoints](#entrypoints) and [Firewall](#firewall) for more information.
- **Health and ports.** _Traefik_ skips containers that are unhealthy. If an image exposes more than one port, set the `loadbalancer.server.port` label or _Traefik_ can't pick one.
- **Remove the canaries' config** (`WHOAMI_*` variables and the `prefix-strip` entry for `WHOAMI_PUBLIC_PATH`) once your own services are routing.

## Entrypoints

The following entrypoints are enabled by default in TIBIA:

| Entrypoint       | Port                 | Purpose |
| ---------------- | -------------------- | ------- |
| `web-insecure`   | `80`                 | Public HTTP. Permanently redirects everything to `web-secure`. |
| `web-secure`     | `443`                | Public HTTPS. Terminates TLS for your services. |
| `private-secure` | `PRIVATE_HTTPS_PORT` (default `8443`) | Private HTTPS. Serves private workloads over TLS, including the dashboard. |

- **What "private" means here:** the `private-secure` entrypoint is a **standard _Traefik_ entrypoint** on a custom port with TLS enabled and a router attached. It is **not authenticated by default**, and **_Docker_ publishes the port on all host interfaces**. Restricting who can reach it is the operator's job, at the host firewall or the cloud security group - see [Firewall](#firewall) for more information. We ship the _Traefik_ dashboard on this port by default to allow Operator access while preventing public use, as a practical demonstration of a restricted service.
- **It's yours to change.** Remove the dashboard router, move it behind your own auth middleware, or change the port. None of that affects the rest of the stack, and other services can make use of the entrypoint to provide secure network-layer traffic segmentation for private workloads.
- **Ports 80 and 443 must stay open to the internet,** whatever else you change. The **ACME** TLS-ALPN challenge connects to 443, and 80 carries the HTTPS redirect.

### Firewall

| Port                 | Open to                              |
| -------------------- | ------------------------------------ |
| `22` (SSH)           | Cloud-provider delimited access - **NEVER CLOSE THIS PORT**.  |
| `80`, `443`          | Everyone - public HTTPS workloads  |
| `PRIVATE_HTTPS_PORT` | Operators only - private HTTPS workloads  |

We supply a script at `scripts/configure-ufw.sh` that backs up the current `ufw` ruleset and then applies the above inbound rules with a `default: deny` policy for any other incoming traffic. **_Check the rules before running it._** Please note that _Traefik_ opens the host's private ports to **_all addresses_** and **_Docker_ can bypass basic `ufw` firewalls** - so this script is only guaranteed to result in locking down non-stack ports. In order to protect the private HTTPS port itself **_you must run a cloud or service firewall_** to be secure. Ensuring appropriate network security is in place is the Operator's responsibility.

## Certificates

TIBIA runs two **Let's Encrypt** resolvers side-by-side, and which one a service uses is set in its router labels.

| Resolver | Issues                              | State file      | Use it for |
| -------- | ----------------------------------- | --------------- | ---------- |
| `stg`    | Untrusted staging certificates      | `acme/stg.json` | Bring-up and debugging. Generous rate limits; browsers will warn. |
| `prd`    | Real, publicly trusted certificates | `acme/prd.json` | Production, once routing is proven on `stg`. |

- **Challenge type:** TLS-ALPN, so public port 443 must reach _Traefik_ from the WAN/internet. Both resolvers need it.
- **Storage:** certificate cache lives in `./acme/` on the host and is mounted into the container at `/etc/traefik/acme`. `./gateway up` creates any missing state file as an empty `{}` with mode `600`. _Traefik_ can silently choke and refuse to use **ACME** storage with incorrect permissions.
- **Never commit these files.** They hold your account key and certificate private keys. `acme/` is already in `.gitignore`.
- **Resetting:** `./gateway clean` deletes both state files; the next `./gateway up` recreates them empty. Stop _Traefik_ first (`./gateway down`) so it doesn't rewrite them.
- **Switching from `stg` to `prd`:** Always run `./gateway down && ./gateway clean`, then `./gateway up production`. **Let's Encrypt** rate-limits duplicate certificates, so performing challenges repeatedly for **_any reason_** when performing domain bringup can lock that **_entire domain_** out of the production issuer - this is compounded by the fact that _Traefik_ can aggressively retry certificate challenges when it thinks they fail, even if **Let's Encrypt** has actually issued the certificate. This challenge/issuance conflict can happen for many reasons, so regardless of the specific method we always recommend bringing a domain online with **ACME** Staging resolvers first. Additionally, if _Traefik_ has cached certificates from another resolvers it may not request new certificates at all when the configuration is updated. The provided scripts manage all of this for the Operator, but manually editing the JSON files is also possible for more granular needs.

## Verifying your setup

Running `./gateway up` starts two [`whoami`](https://github.com/traefik/whoami) canaries next to _Traefik_ - tiny containers that echo back the request they received. The `./gateway verify` script uses them to prove DNS, certificates and routing work before you trust the production resolver or attach a real service.

```sh
./gateway verify               # bringup mode: staging certs, canaries running
./gateway verify production    # after ./gateway up production: live certs, no canaries
```

Run it on the _Traefik_ host. It prints `PASS`/`FAIL` for each check and exits non-zero if any fail:
- _Traefik_ (and in bringup mode, both canaries) are running
- `http://` redirects to HTTPS with a `301`
- The subdomain and path canaries answer, and the path prefix is stripped
- The certificate issuer is staging in bringup mode, and not staging in production
- The dashboard answers on the private port

If the check is made from the host itself then the dashboard passing doesn't prove your firewall is configured correctly - test that from another machine. Some hosts also can't reach their own public address; if every check fails from the host but works from elsewhere, that's why.

### Troubleshooting

| Symptom | Likely cause |
| ------- | ------------ |
| Certificate never issues          | Port 443 isn't reachable from the internet, or DNS doesn't point at this host yet. Fix DNS **before** retrying; repeated failures on `prd` hit **Let's Encrypt** rate limits. _Traefik_'s logs should show the **ACME** errors.  |
| `404 page not found`              | No router matched. Check the `Host` rule, that the container has `traefik.enable=true`, and that it is on `traefik_backend`. The dashboard lists what _Traefik_ actually loaded. _Traefik_'s logs may help diagnose missing routers.  |
| Staging certificate on `prd`      | _Traefik_ reused a cached certificate. Run `./gateway down && ./gateway clean`, then `./gateway up production`, or manually remove the certificate from `acme/stg.json`  |
| Redirect loop                     | A CDN or load balancer in front of _Traefik_ is terminating TLS and forwarding plain HTTP. Pass HTTPS through, or configure _Traefik_'s forwarded-headers trust for it.  |
| Error about the **ACME** file         | The certificate cache's `acme/*.json` files must be mode `600`. `./gateway down && ./gateway clean` then `./gateway up [production]` recreates them correctly. Sometimes certificate cache issues are logged by _Traefik_, but many fail silently.  |

To read _Traefik_'s logs from the host, run: `docker compose -f ./docker-compose-traefik.yaml logs -f`

## Scripts

Run these from the repository root. The Compose files are `docker-compose-traefik.yaml` (the router) and `docker-compose-debug.yaml` (the `whoami` canaries).

| Script                         | What it does |
| ------------------------------ | ------------ |
| `./gateway init`                       | Creates `.env.local` from `.env.base` if it doesn't exist, and creates the `acme/` directory. |
| `./gateway up [production]`            | Loads the environment, creates any missing **ACME** state files, then starts the canaries and _Traefik_ (detached). With `prd`, `prod` or `production` it selects the `prd` resolver and skips the canaries. |
| `./gateway down`                       | Stops the canaries, then _Traefik_. |
| `./gateway clean`                      | Deletes `acme/stg.json` and `acme/prd.json` if present. Run `./gateway down` first. |
| `./gateway verify [production]`        | Runs the routing, certificate and dashboard checks described in [Verifying your setup](#verifying-your-setup). |
| `scripts/create-network.sh`    | Creates the shared `traefik_backend` _Docker_ network. Run once per host. |
| `scripts/install-debian.sh`, `scripts/install-ubuntu.sh` | Installs _Docker Engine_ and the _Compose_ plugin from _Docker_'s apt repository. |
| `scripts/configure-ufw.sh`     | Locks down ports that aren't in use by _Traefik_ - the rules are strict, review them before enabling. See [Firewall](#firewall).|
| `scripts/configure-swap.sh`    | Adds a swap file sized at 100% of RAM (override with `SWAP_PERCENT`), persists it in `/etc/fstab` and sets `vm.swappiness=10`. Skips if swap is already active. |

## Security notes

TIBIA keeps _Traefik_'s defaults except where noted. Here is what is and isn't protected.

- **_Docker_ socket.** _Traefik_ mounts `/var/run/docker.sock` to discover labelled containers. Access to it is effectively root on the host, so **_compromising the Traefik container compromises the machine_**. If that tradeoff doesn't suit you, put a read-only socket proxy in front of it, or switch to the file provider and drop the _Docker_ provider.
- **Only labelled containers are routed** (`exposedbydefault=false`). That limits what gets published, **_not what Traefik can see_**.
- **Dashboard.** Served only on the private port, with `--api.insecure` left off (keep it that way outside short-lived testing). It has no authentication by default; restrict the port at your firewall and consider an auth middleware. See [Entrypoints](#entrypoints) and [Firewall](#firewall).
- **Shared network.** Every container on `traefik_backend` can reach every other one. Give sensitive services their own network as well.
- **Canaries.** `whoami` echoes request headers. Use it for bring-up only; `./gateway up production` skips it.
- **Secrets.** `.env` and `.env.local` hold a domain, an email and port numbers - nothing secret. The sensitive material is `acme/` (account and certificate private keys); keep it out of git and back it up like any other key store.
- **Updates.** `TRAEFIK_VERSION` tracks a minor line (`3.7`), so `docker compose -f ./docker-compose-traefik.yaml pull` then `./gateway down && ./gateway up` picks up patch releases. Read the release notes before changing minor versions.
- **_Docker_ and `ufw`.** _Docker_ publishes ports through its own firewall rules, **_which can bypass `ufw`_**. Verify from an outside machine with `nmap` rather than trusting `ufw status` if you need to restrict published ports from breaching the firewall. Do not rely on a single approach to network security - understanding your full networking model and layering security in depth is strongly recommended.