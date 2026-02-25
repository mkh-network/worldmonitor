FROM ghcr.io/railwayapp/function-bun:1.3.0

WORKDIR /app
RUN bun install ws
COPY ./ais-relay.cjs .
ENV AISSTREAM_API_KEY=$AISSTREAM_API_KEY
ENV RELAY_AUTH_HEADER=$RELAY_AUTH_HEADER
ENV RELAY_SHARED_SECRET=$RELAY_SHARED_SECRET

CMD ["bun", "ais-relay.cjs"]
