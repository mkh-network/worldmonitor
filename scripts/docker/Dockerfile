FROM ghcr.io/railwayapp/function-bun:1.3.0

RUN ["bun","install","ws"]

WORKDIR /app

COPY ./ais-relay.cjs .

RUN ["bun","ais-relay.cjs"]
