# Adapted from ../toolchain-specification/pipeline generator/out/demonstrator/rdfc/Dockerfile,
# trimmed to just Node.js since this pipeline has no Python/Java processors.
FROM node:24-bookworm-slim AS rdfc

WORKDIR /workspace/pipeline

# Copy package manifests first so `npm install` is cached unless dependencies change.
COPY package.json package-lock.json ./
COPY patches ./patches
RUN npm install

# Copy the rest of the pipeline definition.
COPY pipeline.ttl ./
COPY processors ./processors

# ex:HttpServerIn listens here for posted source-a JSON-LD measurements.
EXPOSE 9000

CMD ["npx", "rdfc", "pipeline.ttl"]
