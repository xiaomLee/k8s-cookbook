FROM alpine:3.14.0

COPY html /app/html
COPY entrypoint.sh /app
WORKDIR /app

RUN chmod +x entrypoint.sh
CMD sh -e entrypoint.sh