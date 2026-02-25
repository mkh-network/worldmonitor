FROM ghcr.io/railwayapp/function-bun:1.3.0

RUN ["bun","install","ws"]

WORKDIR /app
ENV AISSTREAM_API_KEY=f77839d374dbaf19eeaa76003447c1184b308522
ENV RELAY_AUTH_HEADER=x-relay-key
ENV RELAY_SHARED_SECRET=wm_relay_prod_1234
COPY ./ais-relay.cjs .

RUN ["bun","ais-relay.cjs"]
