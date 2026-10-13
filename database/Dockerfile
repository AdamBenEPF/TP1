FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr

COPY ./sql/ /docker-entrypoint-initdb.d/
