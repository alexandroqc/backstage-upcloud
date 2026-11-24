# This dockerfile builds an image for the backend package.
#
# It is self-contained: the backend is built inside the image from the repo
# source, so the only input is the repository checkout. Build it with the root
# of the repo as the docker context:
#
#   docker build . -f packages/backend/Dockerfile
#
# Both stages use the same Node version, so native modules built in the build
# stage match the runtime stage.

# ---------------------------------------------------------------------------
# Build stage: install all workspace dependencies and build the backend bundle.
# ---------------------------------------------------------------------------
FROM node:24-trixie-slim AS build

RUN apt-get update && \
    apt-get install -y --no-install-recommends libsqlite3-dev python3 build-essential && \
    rm -rf /var/lib/apt/lists/*

ENV HUSKY=0
WORKDIR /app
COPY . .
RUN yarn install --immutable
RUN yarn workspace example-backend build

# ---------------------------------------------------------------------------
# Runtime stage.
# ---------------------------------------------------------------------------
FROM node:24-trixie-slim

# Install sqlite3 dependencies. You can skip this if you don't use sqlite3 in the image,
# in which case you should also move better-sqlite3 to "devDependencies" in package.json.
# Additionally, we install dependencies for `techdocs.generator.runIn: local`.
# https://backstage.io/docs/features/techdocs/getting-started#disabling-docker-in-docker-situation-optional
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends libsqlite3-dev python3 python3-pip python3-venv build-essential && \
    yarn config set python /usr/bin/python3

# Set up a virtual environment for mkdocs-techdocs-core.
ENV VIRTUAL_ENV=/opt/venv
RUN python3 -m venv $VIRTUAL_ENV
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

RUN pip3 install mkdocs-techdocs-core==1.1.7

# From here on we use the least-privileged `node` user to run the backend.
WORKDIR /app
RUN chown node:node /app
# The yarn cache mount below creates only its leaf directory; the parent
# /home/node/.yarn/berry is created as root otherwise, and yarn needs to write
# its index directory next to the cache.
RUN mkdir -p /home/node/.yarn/berry && chown -R node:node /home/node/.yarn
USER node

# This switches many Node.js dependencies to production mode.
ENV NODE_ENV=production

# Copy Yarn 3 configuration, release, and plugins
COPY --chown=node:node .yarn ./.yarn
COPY --chown=node:node .yarnrc.yml ./

# Copy repo skeleton first, to avoid unnecessary docker cache invalidation.
# The skeleton contains the package.json of each package in the monorepo,
# and along with yarn.lock and the root package.json, that's enough to run yarn install.
COPY --chown=node:node --from=build /app/packages/backend/dist/skeleton.tar.gz ./
COPY --chown=node:node yarn.lock package.json ./
RUN tar xzf skeleton.tar.gz && rm skeleton.tar.gz

RUN --mount=type=cache,target=/home/node/.yarn/berry/cache,sharing=locked,uid=1000,gid=1000 \
    yarn workspaces focus --all --production

# Then copy the backend bundle and the app configuration files.
COPY --chown=node:node --from=build /app/packages/backend/dist/bundle.tar.gz ./
COPY --chown=node:node app-config*.yaml ./
RUN tar xzf bundle.tar.gz && rm bundle.tar.gz

CMD ["node", "packages/backend", "--config", "app-config.yaml"]
