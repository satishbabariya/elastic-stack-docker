ARG ELK_VERSION

FROM docker.elastic.co/apm/apm-server:${ELK_VERSION}

COPY ./config/apm-server.yml /usr/share/apm-server/apm-server.yml
USER root
RUN chown root:apm-server /usr/share/apm-server/apm-server.yml
USER apm-server