# Don't use osixia/light-baseimage
# sources: https://github.com/osixia/docker-light-baseimage
##FROM osixia/light-baseimage:1.3.2
ARG DEBIAN_CODENAME=buster
FROM artifactory.shared-backend.dev.ue1.aws.okcupid.com:443/docker-remote/debian:${DEBIAN_CODENAME}-slim AS baseimage

COPY baseimage /container
RUN /container/build.sh

ENV LANG="en_US.UTF-8" \
    LANGUAGE="en_US:en" \
    LC_ALL="en_US.UTF-8"

ENTRYPOINT ["/container/tool/run"]

FROM baseimage
ARG DEBIAN_CODENAME
ARG OPENLDAP_PACKAGE_VERSION=2.4.57

ARG LDAP_OPENLDAP_GID
ARG LDAP_OPENLDAP_UID

ARG PQCHECKER_VERSION=2.0.0
ARG PQCHECKER_SHA256=151895a59f14d7a0c98fe84ff8f8f91a74e0f3a03e6c824ba53bae1eb30ca374
#ARG PQCHECKER_URL="http://www.meddeb.net/pub/pqchecker/deb/8/pqchecker_${PQCHECKER_VERSION}_amd64.deb"
ARG PQCHECKER_URL="https://public-ba1dca76-0e08-11ed-8c49-97827e72b5d1.s3.amazonaws.com/pqchecker_${PQCHECKER_VERSION}_amd64.deb"

# Add openldap user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
# If explicit uid or gid is given, use it.
RUN if [ -z "${LDAP_OPENLDAP_GID}" ]; then groupadd -g 911 -r openldap; else groupadd -r -g ${LDAP_OPENLDAP_GID} openldap; fi \
    && if [ -z "${LDAP_OPENLDAP_UID}" ]; then useradd -u 911 -r -g openldap openldap; else useradd -r -g openldap -u ${LDAP_OPENLDAP_UID} openldap; fi

# Add ${DEBIAN_CODENAME}-backports in preparation for downloading newer openldap components, especially sladp
RUN echo "deb http://ftp.debian.org/debian ${DEBIAN_CODENAME}-backports main" >> /etc/apt/sources.list

# Install OpenLDAP, ldap-utils and ssl-tools from the (backported) baseimage and clean apt-get files
# sources: https://github.com/osixia/docker-light-baseimage/blob/master/image/tool/add-service-available
#          https://github.com/osixia/docker-light-baseimage/blob/master/image/service-available/:ssl-tools/download.sh
RUN echo "path-include /usr/share/doc/krb5*" >> /etc/dpkg/dpkg.cfg.d/docker && apt-get -y update \
    && /container/tool/add-service-available :ssl-tools \
    && LC_ALL=C DEBIAN_FRONTEND=noninteractive apt-get -t ${DEBIAN_CODENAME}-backports install -y --no-install-recommends \
    ca-certificates \
    curl \
    ldap-utils=${OPENLDAP_PACKAGE_VERSION}\* \
    libsasl2-modules \
    libsasl2-modules-db \
    libsasl2-modules-gssapi-mit \
    libsasl2-modules-ldap \
    libsasl2-modules-otp \
    libsasl2-modules-sql \
    openssl \
    slapd=${OPENLDAP_PACKAGE_VERSION}\* \
    slapd-contrib=${OPENLDAP_PACKAGE_VERSION}\* \
    krb5-kdc-ldap \
    && curl -o pqchecker.deb -SL "${PQCHECKER_URL}" \
    && echo "${PQCHECKER_SHA256} *pqchecker.deb" | sha256sum -c - \
    && dpkg -i pqchecker.deb \
    && rm pqchecker.deb \
    && update-ca-certificates \
    && apt-get remove -y --purge --auto-remove curl ca-certificates \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Add service directory to /container/service
ADD service /container/service

# Use baseimage install-service script
# https://github.com/osixia/docker-light-baseimage/blob/master/image/tool/install-service
RUN /container/tool/install-service

# Add default env variables
ADD environment /container/environment/99-default

# Expose default ldap and ldaps ports
EXPOSE 389 636

# Put ldap config and database dir in a volume to persist data.
# VOLUME /etc/ldap/slapd.d /var/lib/ldap
