FROM postgres:17.2-alpine

COPY ./scripts/* /docker-entrypoint-initdb.d/
USER root
RUN chown postgres:postgres /docker-entrypoint-initdb.d/*.sql
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["postgres"]
