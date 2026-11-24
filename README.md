# Oravil CI images

Images that Oravil's CI jobs start from, so a job does not install its toolchain on every run.

| Image | Holds | Used by |
|---|---|---|
| `ghcr.io/oravil/ci-php:8.4` | Ubuntu 24.04, PHP 8.4 (the php-builder build that `shivammathur/setup-php` installs on a self-hosted runner), the same ini defaults, Composer, git, unzip, curl | `oravil/mobile-wallet`, `oravil/orcash-wallet` |

Rules:
- Nothing secret goes in an image: this repository and its packages are public, so the self-hosted runner pulls them without credentials.
- Consumers pin a digest (`ghcr.io/oravil/ci-php:8.4@sha256:...`), never a bare tag. A new build reaches a repository only through a pull request that changes that digest, with its CI green.
- Coverage drivers (Xdebug, PCOV) are off. A repository that starts collecting coverage in CI needs its own image or step.
- `php/Dockerfile` fails its own build when an extension a consumer asks for is missing.

Publishing: a push to `main`, a manual run of the `build` workflow, or the monthly schedule builds and pushes `8.4` and `8.4-YYYYMMDD`; the run summary prints the digest. A pull request builds and reports only.
