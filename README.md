# Running SIXTA from a release

This is where SIXTA is distributed. The source repositories are private;
this page, the releases beside it and the two release image packages are
public, and nothing on this page needs a GitHub account. SIXTA is two
container images deployed as one unit, or two RPM packages for a RHEL 9
host, versioned together: one release `vX.Y.Z` here carries both
packages, the public signing key, an SBOM of each binary, a checksum file
per package, the deployment bundle and the digest of each image
(`sixta-backend.image-digest`, `sixta-mcp-tools.image-digest`), which the
two image tags `vX.Y.Z` resolve to.

This is proprietary software, and downloading it grants no licence: use is
under a written agreement with Sixta Systems Inc. The terms are in
`LICENSE.txt` beside this page and in every release, inside each RPM under
`/usr/share/licenses/`, at the root of each image, and printed by either
binary with `--license`. To report a security issue, see `SECURITY.md`.

| Image | What | Port |
|---|---|---|
| `ghcr.io/sixta-systems/sixta-releases/sixta-mcp-tools:vX.Y.Z` | the MCP tools server (database, AWS, Datadog, PMM, ServiceNow reads; governed writes) | 8080, never published |
| `ghcr.io/sixta-systems/sixta-releases/sixta-backend:vX.Y.Z` | investigations, the web UI, the standing analyses | 8100 web, 8091 probes, 8090 notify (never published) |

The two packages hold release tags and nothing else: an image appears there
only once its release was published, and there is no `latest`, so pin a
version, or the digest behind it. Both are `linux/amd64`, run as user
`sixta` (uid 1000), and carry a SLSA provenance attestation and an SPDX SBOM:

    docker buildx imagetools inspect ghcr.io/sixta-systems/sixta-releases/sixta-backend:vX.Y.Z --format '{{.Manifest.Digest}}'   # the release's sixta-backend.image-digest
    docker buildx imagetools inspect ghcr.io/sixta-systems/sixta-releases/sixta-backend:vX.Y.Z --format '{{json .SBOM}}'

Every release is cut in both source repositories on the same day, and
everything in a release here was built from that pair. A rollback is the
previous `vX.Y.Z` on both. Start mcp-tools first, then the backend (the
backend's tool schemas are strict about the server's version). The compose
file and the systemd units order their startup; in a pod both containers
start together, and the backend retries each MCP URL for two minutes and
re-establishes its session when the server restarts, so the rule matters
only when you start the two by hand.

## Pulling the images

The images are public, so no login is needed:

    docker pull ghcr.io/sixta-systems/sixta-releases/sixta-backend:vX.Y.Z
    docker pull ghcr.io/sixta-systems/sixta-releases/sixta-mcp-tools:vX.Y.Z

If runtime pulls from an external registry are blocked on your network,
import once into your own registry and run from there:

    az acr import --name <yourregistry> \
      --source ghcr.io/sixta-systems/sixta-releases/sixta-backend:vX.Y.Z \
      --image sixta-backend:vX.Y.Z

and the same for `sixta-mcp-tools`. The attestation manifests travel with the
image (they appear as `unknown/unknown` platform entries in an index listing;
that is expected).

## The deployment bundle

Each release carries `sixta-deploy-vX.Y.Z.tar.gz`: this page with
`LICENSE.txt` and `SECURITY.md` beside it, the compose file, the pod
spec, the two environment templates and the smoke script, which the next
two sections use. The two files that name a version,
`.env.example`'s `SIXTA_VERSION` and the pod spec's `image:` lines, are
stamped with the release's tag when the bundle is built, so the bundle
from `vX.Y.Z` runs `vX.Y.Z` as it stands. Download it from the release
page, or:

    gh release download vX.Y.Z -R sixta-systems/sixta-releases -p 'sixta-deploy-*'
    tar xzf sixta-deploy-vX.Y.Z.tar.gz

(`curl -LO https://github.com/sixta-systems/sixta-releases/releases/download/vX.Y.Z/sixta-deploy-vX.Y.Z.tar.gz`
does the same without `gh`.)

## Docker Compose

    cp .env.example .env                       # SIXTA_VERSION
    cp example.env sixta.env && chmod 600 sixta.env && $EDITOR sixta.env
    cp /path/to/targets.yaml targets.yaml
    mkdir -p knowledge/client && cp /path/to/<customer>.md knowledge/client/
    docker compose up -d && ./smoke.sh

`example.env` names every variable and is pinned to the code by a test;
`sixta.env` is your copy. The MCP server binds the private compose network
and requires `SIXTA_MCP_AUTH_TOKEN`; the backend sends the same value. That
bearer travels in cleartext on the compose network, a bridge on this host
that nothing off it can read, and the backend refuses to send a token over
plain http to any host but loopback unless the link is stated private:
the compose file sets `SIXTA_MCP_ALLOW_PLAINTEXT_TOKEN=1` for exactly that
network. Split the two containers across hosts and that statement stops
being true: put TLS in front of the MCP server (an `https` URL) instead.
The web UI is on `127.0.0.1:8100`; to expose it, set `SIXTA_WEB_TLS_CERT`
and `SIXTA_WEB_TLS_KEY` (or a reverse proxy that passes SSE through
unbuffered) and change the bind on purpose.

## Kubernetes

`k8s/sixta.yaml` is a Deployment of one pod with both containers, a Service
for the web port and a ReadWriteOnce claim. The header of that file says how
to split `example.env` into the Secret and the ConfigMap and how to bind a
managed identity on AKS. It also names a `ghcr-pull` registry secret: the
images are public and need none, so create it only if you mirror them into
a private registry of your own (after the import above, rewrite the
`image:` lines to it), or remove the `imagePullSecrets` lines. The
constraints are not optional: one replica, `Recreate`, and a disk-backed
claim (never Azure Files or NFS), because the web store is SQLite. Azure
Container Apps offers only Azure Files for persistent storage and is
therefore not a supported shape for the web store.

## RPM packages (RHEL 9 family hosts)

Each release also carries `sixta-mcp-tools-X.Y.Z-1.x86_64.rpm` and
`sixta-backend-X.Y.Z-1.x86_64.rpm`, signed with the Sixta packaging key
(fingerprint `80FA 1B7E 96B5 09B0 2869  3256 58E2 C5EB B870 397D`, public key `RPM-GPG-KEY-sixta`
beside them), with an SPDX SBOM of each binary and a checksum file per
package, `SHA256SUMS.sixta-mcp-tools` and `SHA256SUMS.sixta-backend`,
each listing the files its package's build uploaded (the key and the
deployment bundle among them). Download the whole release into one
directory:

    gh release download vX.Y.Z -R sixta-systems/sixta-releases -p '*' -D sixta-vX.Y.Z
    cd sixta-vX.Y.Z

(or each asset with `curl -LO https://github.com/sixta-systems/sixta-releases/releases/download/vX.Y.Z/<asset>`).
Then, as root on the host:

    gpg --show-keys --with-fingerprint RPM-GPG-KEY-sixta   # must match the fingerprint you were given
    rpm --import RPM-GPG-KEY-sixta
    sha256sum -c SHA256SUMS.sixta-mcp-tools SHA256SUMS.sixta-backend
    rpm -K *.rpm                                           # "digests signatures OK"
    dnf install ./*.rpm

Two things about that sequence. The key file and the fingerprint above come
from the same place, so one confirms only that the other is intact, not
that either is Sixta's: compare the fingerprint `gpg` prints with the one
Sixta gave you through another channel before you import. And `rpm -K` is
the check that matters: `dnf install` of a local file does not verify the
signature unless `localpkg_gpgcheck=1` is set in `/etc/dnf/dnf.conf`, so
run `rpm -K` as shown, or set that option and let `dnf` refuse an unsigned
or mis-signed package itself.

1. `/etc/sixta/targets.yaml` (owner `root:sixta`, mode 0640): the targets file.
2. `/etc/sixta/mcp-tools.env`: the credentials the targets file names by `*_env`.
3. `/etc/sixta/backend.env`: a copy of `example.env`; every CHANGEME.
4. `/var/lib/sixta/knowledge/client/<SIXTA_CUSTOMER_ID>.md` (optional).
5. `systemctl enable --now sixta-mcp-tools sixta-backend`, then `journalctl -u sixta-backend -f`.

| Path | What |
|---|---|
| `/usr/bin/sixta-mcp-tools`, `/usr/bin/sixta-backend` | the binaries |
| `/usr/lib/systemd/system/sixta-*.service` | the units (`User=sixta`, `ProtectSystem=strict`, `MemoryMax=512M`) |
| `/etc/sixta/*.env` | configuration; an upgrade keeps your edits and writes the new template as `.rpmnew` |
| `/var/lib/sixta` | backend state: `memory/`, `logs/`, `web/`, the shipped `knowledge/skills/`, your `knowledge/client/` |
| `/var/lib/sixta-mcp` | MCP server state: `manifests/` (empty; see the mcp-tools image doc for the binaries policy), `tasks/` |

The MCP server binds `127.0.0.1:8080` and the backend dials it there; on one
host no token is needed. Upgrade with `dnf upgrade` on both packages (the
units restart on upgrade; mcp-tools first when done by hand). No SELinux
policy module is shipped; the services run in the default unconfined service
domain. The web UI listens on `SIXTA_WEB_PORT`; set `SIXTA_WEB_TLS_CERT` and
`SIXTA_WEB_TLS_KEY` or keep the port behind a firewall.

## What is baked in, what you mount

| | baked in | you supply |
|---|---|---|
| mcp-tools | the binary, CA certificates | `targets.yaml` (read-only); manifests and their binaries, if any (see the mcp-tools image doc) |
| backend | the binary, the web UI, `knowledge/skills/` | `knowledge/client/<customer>.md`; volumes for `memory/`, `logs/`, `web/`; optional schedules, prices and certification files |
| the RPMs | the same, with a systemd unit and an environment template each | the same files under `/etc/sixta` and `/var/lib/sixta` (see above) |

Volumes attach to `/opt/sixta/data/memory`, `/opt/sixta/data/logs` and
`/opt/sixta/data/web`, never to `/opt/sixta/data` itself: the skills live
there and a volume would hide them (the backend would log one WARN and
answer with no domain knowledge). Both compose and the pod spec follow this;
a test in the source repository keeps them to it.

## Probes and shutdown

- mcp-tools: `GET /livez`, `GET /readyz` on 8080 (ready once the targets
  file is parsed and the first cluster-discovery pass has finished; exempt
  from the token).
- backend: `GET /livez`, `GET /readyz` on `SIXTA_HEALTH_PORT` (ready once the
  tool registry has a tool, every listener is bound and, where a relay is
  configured, registration succeeded; back to 503 when an MCP server stops
  answering or shutdown begins).
- Shutdown: the backend prints at startup the grace period it needs (160
  seconds on default budgets: a 155-second drain plus 5 seconds to deliver
  the signal and exit); `stop_grace_period`, `terminationGracePeriodSeconds`
  and the unit's `TimeoutStopSec` carry it, and a test in the source
  repository keeps them at or above the printed figure.

## Logs

Both containers log JSON to stderr. `SIXTA_DEBUG=1` admits debug records.

## Microsoft Foundry

`example.env`'s worked example is the Foundry shape with Entra ID: no key,
`AZURE_TOKEN_CREDENTIALS=prod`, the resource's public FQDN with the `/openai/v1/`
path, the deployment name as `SIXTA_LLM_MODEL`. The identity needs Cognitive
Services OpenAI User on the resource; the startup probe proves the identity
can mint a token, and a missing role shows as a 401 on the first request.
