# Copyright (c) HashiCorp, Inc.
# SPDX-License-Identifier: MPL-2.0

# This Dockerfile creates a production release image for the project. This
# downloads the release from releases.hashicorp.com and therefore requires that
# the release is published before building the Docker image.
#
# We don't rebuild the software because we want the exact checksums and
# binary signatures to match the software and our builds aren't fully
# reproducible currently.
ARG BASE_IMAGE
FROM $BASE_IMAGE as build

# This is the release of Vault to pull in.
ARG VAULT_VERSION

# This is the location of the releases.
ENV HASHICORP_RELEASES=https://releases.hashicorp.com

# all scripts are 0755 (rwx r-x r-x)
COPY --chmod=0755 rootfs/usr/local/bin/* /usr/local/bin/

# all non-scripts are 0644 (rw- r-- r--)
COPY --chmod=0644 rootfs/etc/systemd/system/* /etc/systemd/system/

# Enable systemd.
RUN apt-get update \
    && apt-get install -y systemd systemd-sysv \
    && apt-get clean \
    && find /lib/systemd/system/sysinit.target.wants/ -name "systemd-tmpfiles-setup.service" -delete \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* \
    && rm -rf /lib/systemd/system/multi-user.target.wants/* \
    && rm -rf /etc/systemd/system/*.wants/* \
    && rm -rf /lib/systemd/system/local-fs.target.wants/* \
    && rm -rf /lib/systemd/system/sockets.target.wants/*udev* \
    && rm -rf /lib/systemd/system/sockets.target.wants/*initctl* \
    && echo "ReadKMsg=no" >> /etc/systemd/journald.conf

RUN systemctl enable vault
RUN systemctl enable vault-bootstrap

RUN addgroup vault \
 && adduser --system --group vault \
 && mkdir -p /vault/data \
 && mkdir -p /vault/tls \
 && mkdir -p /etc/vault.d

# Set up certificates, base tools, and Vault.
# libc6 is needed to symlink the shared libraries for ARM builds
RUN set -eux \
    && apt update && apt install -y ca-certificates curl gnupg openssl \
        jq libc6 tzdata wget unzip coreutils \
    && update-ca-certificates \
    && gpg --keyserver keyserver.ubuntu.com --recv-keys C874011F0AB405110D02105534365D9472D7468F \
    && mkdir -p /tmp/build \
    && cd /tmp/build \
    && binArch="$(uname -m)" \
    &&  case "${binArch}" in \
            aarch64) vaultArch='arm64' ;; \
            armhf) vaultArch='arm' ;; \
            x86) vaultArch='386' ;; \
            x86_64) vaultArch='amd64' ;; \
            *) echo >&2 "error: unsupported architecture: ${binArch} (see ${HASHICORP_RELEASES}/vault/${VAULT_VERSION}/)" && exit 1 ;; \
        esac \
    && wget ${HASHICORP_RELEASES}/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_${vaultArch}.zip \
    && wget ${HASHICORP_RELEASES}/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_SHA256SUMS \
    && wget ${HASHICORP_RELEASES}/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_SHA256SUMS.sig \
    && gpg --batch --verify vault_${VAULT_VERSION}_SHA256SUMS.sig vault_${VAULT_VERSION}_SHA256SUMS \
    && grep vault_${VAULT_VERSION}_linux_${vaultArch}.zip vault_${VAULT_VERSION}_SHA256SUMS | sha256sum -c \
    && unzip -d /tmp/build vault_${VAULT_VERSION}_linux_${vaultArch}.zip \
    && cp /tmp/build/vault /bin/vault \
    && if [ -f /tmp/build/EULA.txt ]; then mkdir -p /usr/share/doc/vault; mv /tmp/build/EULA.txt /usr/share/doc/vault/EULA.txt; fi \
    && if [ -f /tmp/build/TermsOfEvaluation.txt ]; then mkdir -p /usr/share/doc/vault; mv /tmp/build/TermsOfEvaluation.txt /usr/share/doc/vault/TermsOfEvaluation.txt; fi \
    && cd /tmp \
    && rm -rf /tmp/build \
    && cd /vault/tls \
    && openssl req -out tls.crt -new -keyout tls.key -newkey rsa:4096 -nodes -sha256 -x509 -subj "/O=HashiCorp/CN=Vault" -days 1095 \
    && chmod 600 /vault/tls/tls.crt /vault/tls/tls.key \
    && cd /tmp \
    && gpgconf --kill all \
    && apt remove -y gnupg openssl \
    && rm -rf /root/.gnupg \
# tiny smoke test to ensure the binary we downloaded runs
    && vault version

# Now that /etc/vault.d and vault is installed copy it over.
COPY --chmod=0644 rootfs/etc/vault.d/* /etc/vault.d/
RUN touch /etc/vault.d/vault.env && chown -R vault:vault /vault /etc/vault.d

# squash down to one compressed layer, without any lingering whiteout files etc
FROM scratch
COPY --from=build / /
# add metadata, must be done after the squashing
# first tell systemd that it is in docker (it will check for the container env)
# https://systemd.io/CONTAINER_INTERFACE/
ENV container docker
ENV DEBIAN_FRONTEND noninteractive
# Expose the vault data directory as a volume since there's mutable state in there.
VOLUME /vault/data
EXPOSE 8200 8201

# systemd exits on SIGRTMIN+3, not SIGTERM (which re-executes it)
# https://bugzilla.redhat.com/show_bug.cgi?id=1201657
STOPSIGNAL SIGRTMIN+3
# NOTE: this is *only* for documentation, the entrypoint is overridden later
ENTRYPOINT [ "/usr/local/bin/vault-entrypoint", "/sbin/init" ]
