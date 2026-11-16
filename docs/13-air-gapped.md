# 13 — Air-gapped install

WebArm24 runs without internet access. You give up the automatic pieces that need it, so plan these:

| Normally online | Air-gapped replacement |
|---|---|
| image pull from the registry | `docker save` / `docker load` tarball |
| weekly license check-in (renewed keys) | `LICENSE_PHONE_HOME=false` + manual key refresh |
| Let's Encrypt | `TLS_MODE=internal` or your own certificate ([05](05-domain-tls.md)) |
| AI provider APIs | a model gateway inside your network (`OPENROUTER_BASE_URL`, [06](06-ai-providers.md)) |
| `py_requirements` from PyPI | private mirror (`PIPELINE_PYTHON_PACKAGES_INDEX_URL`) or a wheelhouse |
| UI fonts and code editor from Google Fonts / cdnjs | allow those two hosts in your proxy, or accept the fallback fonts. A fully offline UI asset pack is on the roadmap |

## 1. On a connected machine

```bash
docker login registry.webarm24.online -u lic_… -p wreg_…
docker pull registry.webarm24.online/webarm/selfhosted:1.4.2
docker pull postgres:16-alpine && docker pull caddy:2.8 && docker pull alpine:3.20
docker save registry.webarm24.online/webarm/selfhosted:1.4.2 postgres:16-alpine caddy:2.8 alpine:3.20 \
  | gzip > webarm-images-1.4.2.tar.gz
sha256sum webarm-images-1.4.2.tar.gz > webarm-images-1.4.2.tar.gz.sha256
```
Transfer the tarball, its checksum, the bundle and `license.key` to the offline server.

## 2. On the offline server

```bash
sha256sum -c webarm-images-1.4.2.tar.gz.sha256
gunzip -c webarm-images-1.4.2.tar.gz | docker load
cd /opt/webarm
cp .env.example .env
sed -i 's/^WEBARM_TAG=.*/WEBARM_TAG=1.4.2/; s/^LICENSE_PHONE_HOME=.*/LICENSE_PHONE_HOME=false/' .env
echo "TLS_MODE=internal" >> .env
PUBLIC_HOST=automation.corp.local ADMIN_EMAILS=you@corp.local LICENSE_FILE=./license.key ./webarmctl install
```
`install` sees that the image is already present and skips the registry login and the pull. `preflight` warns that the registry is unreachable; that's expected.

## 3. Python packages offline (optional)

On a connected machine with the same Python as the image (3.12, manylinux x86_64), build a wheelhouse:
```bash
pip download --only-binary=:all: --python-version 3.12 --platform manylinux2014_x86_64 \
  -d wheelhouse pandas lxml openpyxl
```
Copy `wheelhouse/` into the `appdata` volume, for example with `docker cp wheelhouse webarm-app-1:/app/backend/data/wheelhouse`. Then set:
```bash
PIPELINE_PYTHON_PACKAGES_FIND_LINKS=/app/backend/data/wheelhouse
PIPELINE_PYTHON_PACKAGES_NO_INDEX=true
```

## 4. Renewing the key offline

When the Updates subscription renews:
1. A connected person downloads `license.key` from **Your licenses** on https://webarm24.online/self-hosted.
2. They carry it over to the offline server.
3. Run `./webarmctl license set license.key`.

The new key extends `updates_until`. Your current version keeps running in any case.

## 5. Upgrading offline

Repeat step 1 for the new tag. On the server, `docker load` it, then run `./webarmctl upgrade <tag>`. The upgrade checks the license against the image's build date offline.
