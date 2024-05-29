# This file is moved to the root directory before building the image

# base node image
FROM node:20-bookworm-slim as base

# set for base and all layer that inherit from it
ENV NODE_ENV production

# Install openssl for Prisma
RUN apt-get update && apt-get install -y openssl ca-certificates

# Install all node_modules, including dev dependencies
FROM base as deps

WORKDIR /app

ADD package.json package-lock.json .npmrc ./
RUN npm install --include=dev

# Setup production node_modules
FROM base as production-deps

WORKDIR /app

COPY --from=deps /app/node_modules /app/node_modules
ADD package.json package-lock.json .npmrc ./
RUN npm prune --omit=dev

# Build the app
FROM base as build

ARG COMMIT_SHA
ENV COMMIT_SHA=$COMMIT_SHA

WORKDIR /app

COPY --from=deps /app/node_modules /app/node_modules

ADD prisma .
RUN npx prisma generate

ADD . .

# Mount the secret and set it as an environment variable and run the build
RUN --mount=type=secret,id=SENTRY_AUTH_TOKEN \
    export SENTRY_AUTH_TOKEN=$(cat /run/secrets/SENTRY_AUTH_TOKEN) && \
    npm run build

# Finally, build the production image with minimal footprint
FROM base
ENV PORT="3000"
ENV NODE_ENV="production"
# For WAL support: https://github.com/prisma/prisma-engines/issues/4675#issuecomment-1914383246
ENV PRISMA_SCHEMA_DISABLE_ADVISORY_LOCK = "1"

WORKDIR /app

RUN addgroup --system --gid 1001 remix
RUN adduser --system --uid 1001 remix
USER remix

COPY --from=production-deps --chown=remix:remix /app/node_modules /app/node_modules
COPY --from=build --chown=remix:remix /app/node_modules/.prisma /app/node_modules/.prisma

COPY --from=build --chown=remix:remix /app/server-build /app/server-build
COPY --from=build --chown=remix:remix /app/build /app/build
COPY --from=build --chown=remix:remix /app/package.json /app/package.json
COPY --from=build --chown=remix:remix /app/prisma /app/prisma
COPY --from=build --chown=remix:remix /app/app/components/ui/icons /app/app/components/ui/icons

ADD . .

EXPOSE ${PORT}
RUN npx prisma migrate deploy
ENTRYPOINT [ "npm", "run", "start"]
