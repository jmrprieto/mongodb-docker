FROM mongo:4.4
RUN rm /usr/bin/mongo /usr/bin/mongod 
USER mongodb
CMD ["/usr/bin/mongos", "--config", "/etc/mongos.conf"]
