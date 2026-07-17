ARG BUILD_FROM
# hadolint ignore=DL3006
FROM $BUILD_FROM

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN mkdir -p /opt/app
COPY . /opt
WORKDIR /opt/app

ARG BUILD_ARCH=amd64
ARG NIGHTSCOUT_VERSION=15.0.7

ENV NODE_OPTIONS="--max-old-space-size=2048"

# hadolint ignore=DL3008
RUN \
    URL="http://archive.ubuntu.com/ubuntu/" \
    && if [ "${BUILD_ARCH}" = "aarch64" ]; then \
        URL="http://ports.ubuntu.com/ubuntu-ports/"; \
    fi \
    && echo "deb ${URL} bionic universe" \
        > /etc/apt/sources.list.d/bionic-universe.list \
    \
    && echo "deb ${URL} bionic main" \
        > /etc/apt/sources.list.d/bionic-main.list \
    \
    && apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates curl gnupg \
    && mkdir -p /etc/apt/keyrings \
    && curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_20.x nodistro main" | tee /etc/apt/sources.list.d/nodesource.list \
    \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        mongodb-server \
        nodejs \
        git \
        webpack \
        build-essential \
        python2.7 \
        nginx \
        libnginx-mod-http-lua \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    \
    && git clone --depth 1 --branch ${NIGHTSCOUT_VERSION} https://github.com/nightscout/cgm-remote-monitor.git /opt/app \
    \
    && cd /opt/app \
    # 1. Отключаем генерацию карт кода (Source Maps) для экономии ОЗУ
    && sed -i "s/devtool: .*,/devtool: false,/g" webpack/webpack.config.js \
    && sed -i "s/target: 'web',/target: 'web', resolve: { fallback: { 'util': false, 'crypto': false, 'stream': false } },/g" webpack/webpack.config.js \
    \
    && npm install --legacy-peer-deps \
    # 2. Принудительно собираем Webpack ОДИН РАЗ прямо сейчас
    && ./node_modules/.bin/webpack --mode production --config webpack/webpack.config.js \
    && npm run-script post-generate-keys \
    && npm run env \
    # 3. ХАК: Вырезаем postinstall из package.json, чтобы контейнер не пытался запускать Webpack на старте
    && sed -i 's/"postinstall":.*/"postinstall": "echo skip",/g' package.json \
    && npm audit fix --force || true

RUN npm install jsdom@21.1.1

VOLUME ["/data/db"]

# Копируем файловую систему (проверяем оба варианта расположения)
COPY . /

# Патчим nginx, чтобы не падал без SSL
RUN if [ -f /etc/cont-init.d/nginx.sh ]; then sed -i 's/exit 1/exit 0/g' /etc/cont-init.d/nginx.sh; fi

ARG BUILD_DATE
ARG BUILD_REF
ARG BUILD_VERSION

LABEL \
    io.hass.name="Nightscout" \
    io.hass.description="Nightscout acts as a web-based CGM (Continuous Glucose Monitor) to allow multiple caregivers to remotely view a patients glucose data in realtime." \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version=${BUILD_VERSION} \
    maintainer="Marcio Granzotto <oscensores@gmail.com>"
