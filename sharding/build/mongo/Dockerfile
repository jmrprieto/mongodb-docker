FROM alpine:3.9
RUN apk add --no-cache mongodb && \
    rm /usr/bin/mongod /usr/bin/mongos 
USER mongodb
