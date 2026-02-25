FROM ghcr.io/railwayapp/function-bun:1.3.0

WORKDIR /app
COPY ./ais-relay.cjs .
RUN ["bun","install","ws"]

ENV AISSTREAM_API_KEY=f77839d374dbaf19eeaa76003447c1184b308522
ENV RELAY_AUTH_HEADER=x-relay-key
ENV RELAY_SHARED_SECRET=wm_relay_prod_1234

CMD ["bun","ais-relay.cjs"]
