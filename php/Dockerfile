# CI image for Oravil PHP repositories (mobile-wallet, orcash-wallet).
# It holds the same PHP build that shivammathur/setup-php installs on a self-hosted runner
# (shivammathur/php-builder, release, nts) plus the same ini values and Composer, so a CI job
# starts with PHP ready instead of installing it on every run (about 60 s per job, measured 2026-11-24).
# Coverage drivers (Xdebug, PCOV) are off, like `coverage: none` in setup-php: no repository collects coverage in CI.
FROM ubuntu:24.04

ARG PHP_VERSION=8.4
ARG COMPOSER_VERSION=2.11.2

LABEL org.opencontainers.image.source="https://github.com/oravil/ci-images" \
      org.opencontainers.image.description="Ubuntu 24.04 + PHP (php-builder) + Composer for Oravil CI jobs" \
      org.opencontainers.image.licenses="MIT"

# The three COMPOSER_* values are the ones setup-php exports to every later step.
ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8 \
    COMPOSER_PROCESS_TIMEOUT=0 \
    COMPOSER_NO_INTERACTION=1 \
    COMPOSER_NO_AUDIT=1

SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

# System packages the workflows installed with apt, then PHP from php-builder (what setup-php runs for 8.3+ on self-hosted).
RUN apt-get update -qq \
 && apt-get install -y -qq --no-install-recommends git unzip ca-certificates curl sudo \
 && curl -fsSL -o /tmp/install.sh https://github.com/shivammathur/php-builder/releases/latest/download/install.sh \
 && bash /tmp/install.sh self-hosted "${PHP_VERSION}" release nts \
 && echo "== coverage drivers loaded by the stock build:" && (php -m | grep -iE '^(xdebug|pcov)$' || echo "(none)") \
 && rm -rf /tmp/* /var/lib/apt/lists/*

# ini: `ini-file: production` plus setup-php's defaults (src/configs/ini/php.ini and jit.ini), then coverage drivers off.
RUN lib="/usr/lib/php/${PHP_VERSION}"; etc="/etc/php/${PHP_VERSION}"; \
    for ini in "$etc"/*/php.ini; do cp "$lib/php.ini-production" "$ini"; done; \
    if [ -e "$lib/php.ini-production.cli" ]; then cp "$lib/php.ini-production.cli" "$etc/cli/php.ini"; fi; \
    echo production > "$lib/php.ini-current"; \
    for ini in "$etc"/*/php.ini; do printf 'date.timezone=UTC\nmemory_limit=-1\n' >> "$ini"; done; \
    printf 'opcache.enable=1\nopcache.jit_buffer_size=256M\nopcache.jit=1235\n' >> "$etc/mods-available/pecl.ini"; \
    for ext in xdebug pcov; do \
      sed -Ei "/=(.*\/)?\"?${ext}(\.so)?\$/d" "$etc"/*/php.ini "$etc/mods-available/pecl.ini"; \
      find "$etc" -name "*${ext}.ini" -not -path "*mods-available*" -delete; \
    done

# Composer, pinned and checked against the checksum published beside it.
RUN curl -fsSL -o /usr/local/bin/composer "https://getcomposer.org/download/${COMPOSER_VERSION}/composer.phar" \
 && curl -fsSL -o /tmp/composer.sha256 "https://getcomposer.org/download/${COMPOSER_VERSION}/composer.phar.sha256sum" \
 && echo "$(cut -d' ' -f1 /tmp/composer.sha256)  /usr/local/bin/composer" | sha256sum -c - \
 && chmod 755 /usr/local/bin/composer && rm -f /tmp/composer.sha256

# The build fails here if an extension a repository asks for is missing or a coverage driver is still loaded.
RUN php -v && composer --version \
 && for e in pdo_pgsql bcmath intl pcntl redis zip mbstring; do php -m | grep -qix "$e" || { echo "missing extension: $e"; exit 1; }; done \
 && if php -m | grep -qiE '^(xdebug|pcov)$'; then echo "a coverage driver is still loaded"; exit 1; fi \
 && test "$(php -r 'echo ini_get("memory_limit");')" = "-1" \
 && test "$(php -r 'echo ini_get("date.timezone");')" = "UTC"
