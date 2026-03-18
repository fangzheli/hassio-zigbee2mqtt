ARG BUILD_FROM
FROM $BUILD_FROM as base

ENV LANG C.UTF-8
ARG BUILD_VERSION

RUN apk add --no-cache socat tini nodejs eudev && \
    # Validation to confirm that curl is installed in the base image.
    curl --version || (echo "curl missing" && exit 1)

# Dependencies and build
FROM base as dependencies_and_build

RUN apk add --no-cache --virtual .buildtools npm make gcc g++ linux-headers udev git python3 && \
    npm install -g pnpm@10.12.1 && \
    if [ "${BUILD_VERSION}" = "edge" ]; \
    then \
        echo "Installing Custom Edge version with custom zigbee-herdsman" && \
        # Clone zigbee-herdsman
        git clone -b feat/blz --single-branch --depth 1 \
        https://github.com/fangzheli/zigbee-herdsman.git /zigbee-herdsman && \
        cd /zigbee-herdsman && \
        pnpm install --frozen-lockfile && \
        pnpm run build && \
        cd / && \
        # Clone zigbee2mqtt
        git clone -b feat/blz-local-dev --single-branch --depth 1 \
        https://github.com/fangzheli/zigbee2mqtt.git /app && \
        mkdir /app/dist && \
        jq -n --arg commit $(eval cd /app;git rev-parse --short HEAD) '$commit' > /app/dist/.hash ; \
    else \
        echo "Installing Stable version ${BUILD_VERSION%-*}" && \
        curl -sL -o "/app.tar.gz" \
        "https://github.com/Koenkk/zigbee2mqtt/archive/${BUILD_VERSION%-*}.tar.gz" && \
        tar xzvf "/app.tar.gz"  && rm "/app.tar.gz" && \
        mv -v "zigbee2mqtt-${BUILD_VERSION%-*}" app && \
        mkdir /app/dist && \
        jq -n --arg commit ${BUILD_VERSION%-*} '$commit' > /app/dist/.hash ; \
    fi; \
    echo "Installed Zigbee2MQTT @ version $(cat /app/dist/.hash)" && \
    cd /app && \
    if [ "${BUILD_VERSION}" = "edge" ]; then \
        pnpm install --no-optional --unsafe-perm; \
    else \
        pnpm install --frozen-lockfile --no-optional --unsafe-perm; \
    fi && \
    pnpm run build && \
    rm -rf node_modules && \
    if [ "${BUILD_VERSION}" = "edge" ]; then \
        pnpm install --no-optional --unsafe-perm --prod; \
    else \
        pnpm install --frozen-lockfile --no-optional --unsafe-perm --prod; \
    fi && \
    # force serialport to rebuild on current platform instead of using prebuilds
    rm -rf `find ./node_modules/.pnpm/ -wholename "*/@serialport/bindings-cpp/prebuilds" -type d` && \
    pnpm rebuild @serialport/bindings-cpp

# Release
FROM base as release

WORKDIR /app

COPY rootfs /
COPY --from=dependencies_and_build /app/node_modules ./node_modules
COPY --from=dependencies_and_build /app/dist ./dist
COPY --from=dependencies_and_build /app/package.json /app/LICENSE /app/index.js ./

ENV NODE_ENV production

HEALTHCHECK --interval=1m --timeout=3s --start-period=30s --start-interval=3s \
  CMD curl -f http://localhost:8099/

ENTRYPOINT [ "/sbin/tini", "--", "/docker-entrypoint.sh"]
