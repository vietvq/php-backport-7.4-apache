# php-backport: PHP 7.4 + Apache with the CVE-2026-109618/109619 fix

A Docker image of PHP 7.4 (Apache, mod_php) compiled from source, from the
official php-src backport branch
[`PHP-7.4-CVE-2026-109618-109619`](https://github.com/php/php-src/tree/PHP-7.4-CVE-2026-109618-109619).

It works as a drop-in replacement for `php:7.4-apache`: same layout, same
`docker-php-ext-install` / `docker-php-ext-configure` / `docker-php-ext-enable`
helpers, so existing `FROM php:7.4-*-apache` Dockerfiles keep working.

> [!WARNING]
> **This is a stopgap, not a long-term fix.** It closes the two unauthenticated
> RCEs from the [21 Oct 2026 PHP advisory](https://www.php.net/archive/2026.php#2026-10-21-5)
> and nothing else.
> - PHP 7.4 has been end-of-life since Nov 2022. Later PHP security fixes are
>   **not** included, e.g. CVE-2023-0662 (multipart DoS), CVE-2023-3823 (XXE),
>   CVE-2023-3824 (phar), CVE-2023-0567 (`password_verify`), and everything from 2024 on.
> - Debian bullseye (the base OS) is end-of-life too: Apache, OpenSSL, libxml2,
>   curl etc. get no more security updates.
>
> Plan the move to a supported PHP 8.x (8.2.35 / 8.3.36 / 8.4.27 / 8.5.12 or later).

## What's inside

| | |
|---|---|
| PHP source | `github.com/php/php-src`, branch `PHP-7.4-CVE-2026-109618-109619`, commit [`712e60f9`](https://github.com/php/php-src/commit/712e60f9edfd03f0960fc4fc3c2c375f42b90aa2) (7.4.33 + "Avoid state corruption via bailout") |
| Fixes | [GHSA-w8hw-w9fx-j79w](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w) (CVE-2026-109618), [GHSA-6wmf-4rv5-p3rp](https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp) (CVE-2026-109619) |
| `php -v` | `7.4.34-dev` (`PHP_VERSION_ID` 70434), the version set in the branch's `configure.ac` |
| Base OS | `debian:bullseye-slim`, apt pointed at `archive.debian.org` |
| SAPI | Apache 2.4 + mod_php (`apache2handler`), mpm_prefork |
| Built-in extensions | Same as the official 7.4 image (curl, mbstring, mysqlnd, openssl, pdo_sqlite, sodium, …). Add more with `docker-php-ext-install`. |

## Build

```bash
docker build -t php-backport:7.4-apache .
```

The first build takes about 5 minutes (full PHP compile).

The tag `php-backport:7.4-apache` exists only on the machine that built it. To
use it elsewhere, build it there or push it to your own registry:

```bash
docker tag php-backport:7.4-apache registry.example.com/php-backport:7.4-apache
docker push registry.example.com/php-backport:7.4-apache
```

## Use

```dockerfile
FROM php-backport:7.4-apache        # instead of php:7.4-apache

RUN docker-php-ext-install pdo_mysql intl
```

- Don't build dependent images with `--pull` while the base is local-only.
  `--pull` forces a registry lookup, and the build fails.
- After deploying, **recreate** the containers (`docker compose up -d --force-recreate`).
  Reloading isn't enough: old workers can keep the vulnerable state.

To build everything in one step instead, copy this Dockerfile's stages into
the app's Dockerfile (multi-stage, `FROM php-base` for the app stage). The
scripts then have to be copied with their paths adjusted, e.g. to
`docker/php74-apache/`.

## Verify

```bash
docker run --rm php-backport:7.4-apache php -v
# PHP 7.4.34-dev (cli) ...

docker run --rm php-backport:7.4-apache sh -c 'echo $PHP_GIT_COMMIT'
docker image inspect php-backport:7.4-apache \
  --format '{{index .Config.Labels "org.opencontainers.image.revision"}}'
# 712e60f9edfd03f0960fc4fc3c2c375f42b90aa2
```

To check that the compiled source really contains the patch, compare the
files the commit touches with the source tarball kept in the image:

```bash
git clone --depth 1 -b PHP-7.4-CVE-2026-109618-109619 https://github.com/php/php-src.git
mkdir verify
docker run --rm php-backport:7.4-apache cat /usr/src/php.tar.xz | tar -xJf - -C verify
cd php-src
for f in $(git show --name-only --format= 712e60f9); do
  cmp -s "$f" "../verify/php/$f" && echo "same $f" || echo "DIFF $f"
done
# all 17 files should print "same"
```

## How it is built

The Dockerfile is the last official
[docker-library/php](https://github.com/docker-library/php) `7.4/bullseye/apache`
Dockerfile (from before 7.4 was removed in `f2999de8`), with two changes:

1. **Source from Git instead of php.net.** A `source` stage fetches the pinned
   commit and checks that it is on the backport branch. It then generates a
   release-style tarball the way PHP's own `scripts/dev/makedist` does
   (`buildconf --force`, `scripts/dev/genfiles`, PEAR installer). The result
   becomes `/usr/src/php.tar.xz`, exactly where `docker-php-source` expects it.
2. **apt uses `archive.debian.org`.** bullseye is being removed from
   `deb.debian.org`, and its security mirror already returns 404s.

Everything else is unchanged: configure flags, CFLAGS, Apache setup and the
helper scripts.

### Trust

- PHP code comes only from the official php-src repo, pinned by full commit SHA.
  The branch has no signed release tarball, unlike normal PHP releases.
- Debian packages are verified by apt against Debian's signing keys.
- The PEAR installer (`install-pear-nozlib.phar`) is downloaded over HTTPS with
  no checksum, the same as `makedist`.

## Updating

If the backport branch gets new commits:

```bash
git ls-remote https://github.com/php/php-src.git refs/heads/PHP-7.4-CVE-2026-109618-109619
```

If the SHA differs from `PHP_GIT_COMMIT` in the Dockerfile, update it (or pass
`--build-arg PHP_GIT_COMMIT=<sha>`), rebuild, and recreate the containers.

| Build arg | Default |
|---|---|
| `PHP_GIT_REPO` | `https://github.com/php/php-src.git` |
| `PHP_GIT_BRANCH` | `PHP-7.4-CVE-2026-109618-109619` |
| `PHP_GIT_COMMIT` | `712e60f9edfd03f0960fc4fc3c2c375f42b90aa2` |

## Files

| File | |
|---|---|
| `Dockerfile` | Image definition (stages `source` and the final image) |
| `docker-php-source`, `docker-php-ext-*`, `docker-php-entrypoint`, `apache2-foreground` | Unchanged helper scripts from docker-library/php |
| `LICENSE` | MIT license of docker-library/php, which the Dockerfile and scripts are derived from |

## License

The Dockerfile and helper scripts are derived from
[docker-library/php](https://github.com/docker-library/php) and are under its
MIT license (see `LICENSE`). PHP itself, which the image contains, is under the
[PHP License](https://www.php.net/license/).
