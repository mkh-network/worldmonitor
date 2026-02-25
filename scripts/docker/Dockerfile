FROM ghcr.io/railwayapp/function-bun:1.3.0

WORKDIR /app
COPY ./ais-relay.cjs .
RUN ["bun","install","ws"]
