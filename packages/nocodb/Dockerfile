ARG ACCESS_TOKEN

###########
# Litestream Builder
###########
FROM golang:alpine3.18 as lt-builder

WORKDIR /usr/src/

RUN apk add --no-cache git make musl-dev gcc

# build litestream
RUN git clone https://github.com/benbjohnson/litestream.git litestream
RUN cd litestream ; go install ./cmd/litestream

RUN cp $GOPATH/bin/litestream /usr/src/lt



###########
# Builder
###########
FROM node:16.17.0-alpine3.15 as builder
ARG ACCESS_TOKEN
ENV TOKEN_FOR_PACKAGES=${ACCESS_TOKEN}

WORKDIR /usr/src/app

# install node-gyp dependencies
RUN apk add --no-cache python3 make g++

# Copy application dependency manifests to the container image.
# A wildcard is used to ensure both package.json AND package-lock.json are copied.
# Copying this separately prevents re-running npm ci on every code change.
COPY ./package*.json ./
COPY ./docker/main.js ./docker/main.js
#COPY ./docker/start.sh /usr/src/appEntry/start.sh
COPY ./docker/start-litestream.sh /usr/src/appEntry/start.sh
COPY ./src/lib/public/css/*.css ./docker/public/css/
COPY ./src/lib/public/js/*.js ./docker/public/js/
COPY ./src/lib/public/favicon.ico ./docker/public/

# install production dependencies,
# reduce node_module size with modclean & removing sqlite deps,
# package built code into app.tar.gz & add execute permission to start.sh
RUN printf "@balance-my-hormones:registry=https://npm.pkg.github.com/\nalways-auth=true\n//npm.pkg.github.com/:_authToken=%s" $TOKEN_FOR_PACKAGES >> .npmrc

RUN npm ci --omit=dev --quiet \
    && npx modclean --patterns="default:*" --ignore="@balance-my-hormones/nc-lib-gui/**,@balance-my-hormones/incident-handling/**,typeorm/**,highlight.js/**,cli-highlight/**,nc-lib-gui/**,dayjs/**,express-status-monitor/**,@azure/msal-node/dist/**" --run  \
    && rm -rf ./node_modules/sqlite3/deps \
    && tar -czf ../appEntry/app.tar.gz ./* \
    && chmod +x /usr/src/appEntry/start.sh

##########
# Runner
##########
FROM alpine:3.15
WORKDIR /usr/src/app

ENV NC_DOCKER 0.6
ENV NODE_ENV production
ENV PORT 8080
ENV NC_TOOL_DIR=/usr/app/data/

RUN apk --update --no-cache add \
    nodejs \
    tar \
    dumb-init

# Copy litestream binary build
COPY  --from=lt-builder /usr/src/lt /usr/src/appEntry/litestream
# Copy packaged production code & main entry file
COPY --from=builder /usr/src/appEntry/ /usr/src/appEntry/
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 8080

# Start Nocodb
ENTRYPOINT ["/entrypoint.sh"]
