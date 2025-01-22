FROM ubuntu:22.04

USER root

ENV POD_IDENTITY="node1" \
    PATRONI_TTL=60 \
    PATRONI_LOOP_WAIT=10 \
    PATRONI_RETRY_TIMEOUT=40 \
    PATRONI_MAXIMUM_LAG_ON_FAILOVER=1048576 \
    PATRONI_SYNCHRONOUS_MODE="false" \
    PG_CLUST_NAME="common" \
    PG_MAX_CONNECTIONS=200 \
    PG_CONF_MAX_PREPARED_TRANSACTIONS=200 \
    PATRONICTL_CONFIG_FILE="/patroni/pg_node.yml" \
    PG_BIN_DIR="/usr/lib/postgresql/16/bin/" \
    POSTGRESQL_VERSION=16 \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    EDITOR=/usr/bin/vi \
    PATH="/usr/lib/postgresql/16/bin/:${PATH}"

# Official CentOS repos contain libprotobuf-c 1.0.2, but decoderbufs require 1.1+, thus,
# we craft a custom build of protobuf-c and publish it at this repo.
# Remove this line after moving to the next CentOS releases.
COPY scripts/archive_wal.sh /opt/scripts/archive_wal.sh
ADD ./scripts/pip.conf /root/.pip/pip.conf
COPY ./scripts/postgresql.conf /tmp/postgresql.conf
COPY ./scripts/fix_permission.sh /usr/libexec/fix-permissions
ADD ./scripts/* /

RUN echo "deb [trusted=yes] http://apt.postgresql.org/pub/repos/apt jammy-pgdg main" >> /etc/apt/sources.list.d/pgdg.list
RUN ls -la /etc/apt/
RUN apt-get -y update
RUN apt-get -o DPkg::Options::="--force-confnew" -y dist-upgrade
RUN apt-get update && \
    apt-get install -y --allow-downgrades gcc-12 cpp-12 gcc-12-base libgcc-12-dev libstdc++6 libgcc-s1 libnsl2
RUN apt-get --no-install-recommends install -y python3.11 python3-pip python3-dev libpq-dev cython3 wget curl

# rename 'tape' group to 'postgres' and creating postgres user - hask for ubuntu
RUN groupmod -n postgres tape
RUN adduser -uid 26 -gid 26 postgres

ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get --no-install-recommends install -y postgresql-16 postgresql-contrib-16 postgresql-server-dev-16 postgresql-plpython3-16 postgresql-16-hypopg postgresql-16-powa postgresql-16-orafce\
    hostname gettext jq vim \
    postgresql-16-cron postgresql-16-repack postgresql-16-pgaudit postgresql-16-pg-stat-kcache postgresql-16-pg-qualstats postgresql-16-set-user postgresql-16-postgis pgbackrest \
    postgresql-16-pg-wait-sampling postgresql-16-pg-track-settings postgresql-16-pg-hint-plan postgresql-16-pgnodemx postgresql-16-decoderbufs

# Install LDAP utilities including openldap-clients and necessary libraries
RUN apt-get update && apt-get install -y \
    ldap-utils \
    libldap-2.5-0 \
    libsasl2-modules-gssapi-mit \
    libldap-common \
    && rm -rf /var/lib/apt/lists/*


RUN localedef -i en_US -f UTF-8 en_US.UTF-8 && \
    localedef -i es_PE -f UTF-8 es_PE.UTF-8 && \
    localedef -i es_ES -f UTF-8 es_ES.UTF-8

# Install pgsentinel and pg_dbms_stats
RUN apt update && apt-get install -y git make gcc && \
    git clone https://github.com/pgsentinel/pgsentinel.git && \
    cd pgsentinel && \
    git checkout 0218c2147daab0d2dbbf08433cb480163d321839 && \
    cd src && make install && \
    cd ../.. && git clone --depth 1 --branch REL14_0 https://github.com/ossc-db/pg_dbms_stats.git && \
    cd pg_dbms_stats && sed -i 's/$(MAJORVERSION)/14/g' Makefile && \
    make install && \
    apt-get purge -y --auto-remove git make gcc && \
    cd .. && rm -rf pgsentinel

RUN apt-get install -y alien

RUN cat /root/.pip/pip.conf
RUN python3 -m pip install -U setuptools==70.0.0
RUN python3 -m pip install psutil patroni[kubernetes,etcd]==3.3.2 psycopg2-binary==2.9.5 requests python-dateutil urllib3 six prettytable --no-cache
RUN mv /var/lib/postgresql /var/lib/pgsql

RUN mkdir /patroni && chmod -R 777 /patroni/ && \
    chmod +x /usr/libexec/fix-permissions && \
    /usr/libexec/fix-permissions /var/run/postgresql && \
    /usr/libexec/fix-permissions /var/lib/pgsql && \
    mkdir -p /var/lib/pgsql/data/ && \
    chown -R postgres:postgres /var/lib/pgsql && \
    chmod +x /*.py && \
    chmod +x /*.sh && \
    chmod 777 /opt/scripts/archive_wal.sh && \
    ln -s /usr/bin/python3 /usr/bin/python

# Volumes are defined to support read-only root file system
VOLUME /etc
VOLUME /patroni
VOLUME /run/postgresql

WORKDIR /patroni
ENTRYPOINT ["/start.sh"]

USER 26
EXPOSE 5432
EXPOSE 8008
