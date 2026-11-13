# vallus - Docker images

vallus runs Playwright test suites on your own infrastructure: a launcher, a
dashboard per project, scheduled and on-demand runs, reports and history. These
are its container images. The source is not public; this repository only
documents the images.

- **Website:** <https://vallus.eu>
- **Downloads for closed networks:** <https://downloads.vallus.eu> - the same
  images as zip packages, with `SHA256SUMS`, for sites that cannot pull from a
  registry.

## The images

| Image | Runner | Launcher port | Project dashboards |
|---|---|---|---|
| `ghcr.io/pgexpandit/vallus-rs` | Rust (recommended) | 3000 | 3001-3004 |
| `ghcr.io/pgexpandit/vallus-ts` | TypeScript / Node | 3500 | 3501-3504 |

Both serve the same dashboard and the same features. Every tag is a multi-arch image for `linux/amd64` and `linux/arm64`,
so `docker pull` picks the right one for your machine, Apple Silicon included.

## Tags

Each release comes in two flavours:

- **with browsers** - `playwright-<version>`: built on the official Playwright
  image, with Chromium, Firefox, WebKit and ffmpeg for that Playwright version.
  Pick the one that matches `@playwright/test` in your suites
  (`npx playwright --version`).
- **slim** - no browsers. Each test run gets a container of its own, built for the
  Playwright version the project uses, so one runner serves projects on different
  versions. Needs the Docker socket (see below).

| Tag | Meaning |
|---|---|
| `1.7.0-playwright-1.65.0` | That release, that Playwright. Never changes - pin this in production. |
| `1.7-playwright-1.65.0` | The newest 1.7.x release with Playwright 1.65.0. |
| `playwright-1.65.0` | The newest release with Playwright 1.65.0. |
| `1.7.0-slim`, `1.7-slim`, `slim` | The same, for the slim image. |
| `latest` | The newest release, with the newest Playwright it ships. |

Tags starting with `sha-` or a branch name are development builds; do not deploy them.

## Quick start

```bash
docker run -d --name vallus \
  --shm-size=2g \
  -p 3000-3004:3000-3004 \
  -v vallus-data:/data \
  -v /path/to/your/tests:/projects/my-tests \
  ghcr.io/pgexpandit/vallus-rs:latest
```

Open **https://localhost:3000/setup**, create the admin account, enter a licence
and add a project: its **Path** is the path inside the container
(`/projects/my-tests`) and its **Port** one of the published ones (3001).

For the TypeScript image use `-p 3500-3504:3500-3504` and
`https://localhost:3500/setup`; everything else is the same.

`--shm-size=2g` matters: Chromium crashes with Docker's default 64 MB `/dev/shm`.

## HTTPS

The image serves HTTPS on its own. On the first start it creates a local CA in
`/data/.vallus/tls` and issues a certificate from it, so the first visit warns
until you trust that CA:

```bash
docker cp vallus:/data/.vallus/tls/ca-cert.pem ./vallus-ca.pem
```

Name the hosts browsers will use with `-e VALLUS_TLS_HOSTS=runner.lan,192.168.1.20`.
Typing `localhost:3000` without `https://` gets no answer at all - the runner does
not listen on plain HTTP. Behind your own TLS proxy, set `-e VALLUS_PROTOCOL=http`
and `-e VALLUS_FORCE_SECURE_COOKIE=1`, and publish only the proxy's ports.

## Slim image: one container per run

```bash
docker run -d --name vallus \
  --shm-size=2g \
  -p 3000-3004:3000-3004 \
  -v vallus-data:/data \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e VALLUS_EXECUTOR=container \
  -e VALLUS_RUNNER_BUILD=1 \
  -e VALLUS_RUNNER_TRANSFER=copy \
  ghcr.io/pgexpandit/vallus-rs:slim
```

Mounting the Docker socket gives the container control of the host's Docker
daemon - in effect, root on that host. Run it this way on a machine dedicated to
test runs.

## Data

Everything the runner keeps lives in `/data`: settings, users, the encryption
keys for stored secrets, cloned projects, run history and reports. Keep it on a
named volume or a host directory, and back it up - a new container on the same
volume carries on where the old one stopped. Without the volume, the keys and
every stored secret go with the container.

## Licence

vallus is proprietary software, supplied under the
[End User Licence Agreement](https://vallus.eu/legal/eula/). Pulling the images is
free; running them needs a licence key, entered in the setup wizard.

- **Free licence:** 2 users, 1 project, 1 concurrent run, renewable -
  <https://vallus.eu/free/>
- **Commercial licences:** <https://vallus.eu/#pricing>

The open-source components inside vallus and their licences are listed in
`THIRD_PARTY_NOTICES.txt`, shipped in every package on
<https://downloads.vallus.eu>.
