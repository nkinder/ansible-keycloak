Ansible Keycloak Role
=====================

This role is designed to deploy Keycloak on a systemd managed system
for testing and development purposes.  The role will install Keycloak
from an official downloaded Keycloak zip archive on the target system.

The Keycloak server will be configured as a systemd service named
`keycloak`, running as a `keycloak` system user.  The role will handle
the creation of the system user and the service.

This role also handles some of the initial Keycloak server configuration.
This includes configuring what ports to listen on, creating an initial
admin user, and configuring TLS via self-signed or provided certificates.
Firewall configuration is also handled.

Usage
-----
Choose a playbook for whichever TLS scenario you are interested in.  The
following example playbooks are included in this repo:

- **`tls-self-signed.yml`** - TLS via auto-generated self-signed certificate
- **`tls-cert-key.yml`**    - TLS via provided cert/key files
- **`tls-pkcs12.yml`**      - TLS via provided PKCS12 bundle

All of the example playbooks use ansible-vault for security sensitive
variables.  The examples use `password` for all secrets, including the
vault password itself.  New secrets should be generated and updated in
the playbooks to avoid using the hard-coded example.  Instructions for
doing this are in comments in the example playbooks.

To execute your chosen playbook, be sure to add the --ask-vault-pass
option as in this example:

  `ansible-playbook --ask-vault-pass -i <inventory/host list> <playbook>`

If a deployment of the same version of Keycloak exists on the target
system, the playbook will fail to prevent losing data within Keycloak's
database.  A force variable can be set to overwrite the existing deployment.
This can be either be set as a variable in the playbook, or added on the
command line as an extra-var:

  `ansible-playbook ... --extra-vars "force=yes"`

Controlling the location of the Keycloak archive
------------------------------------------------

By default the Keycloak archive will be downloaded to the local system
where the playbook is being run from. It will reside in the directory
specified by the variable `keycloak_local_download_dest`. When the
archive is extracted on the target system the archive will be read on
the local system and the files created on the destination target. This
behavior has the advantage of downloading the archive only once and
not storing the archive on the target but it incurs a network penalty
of transferring the archive contents again for every target during the
extraction process. If you have a slow upload network link on the host
running the playbook the non-local extraction may be untenable.

Alternately you have the option to download the archive directly to
the target and extract the archive on the target, this is controlled
by the `keycloak_archive_on_target` variable. This has the advantage
of transferring the archive data only once. The archive extraction
will be fast because there is no network traffic during the extraction
because all data is local to the target. However it will leave the
archive on the target after extraction.

To change the location of the archive on the command line add this
argument to your playbook command line:

  `-e "{keycloak_archive_on_target: True}"`
  
Variables
---------
You can choose what version of Keycloak to install by setting the following
variable.  This will be used to determine the download URL to use from
https://www.keycloak.org/downloads.html:

- `keycloak_version` (default: `4.8.2.Final`)

The following variable always needs to be provided, as the role does
not hardcode a default:

- `keycloak_admin_password` (use ansible-vault to protect)

You can choose a non-default admin account name by setting the following
variable:

- `keycloak_admin_user` (default: `admin`)

To configure the interface and ports that the Keycloak server listens
on, the following variables can be set:

- `keycloak_bind_address` (default: `0.0.0.0`)
- `keycloak_http_port` (default: `8080`)
- `keycloak_https_port` (default: `8443`)

For TLS via PKCS12 bundle, the following additional variables must be
provided:

- `keycloak_tls_ca_certificate` (local path to CA cert file)
- `keycloak_tls_pkcs12` (local path to PKCS12 bundle)
- `keycloak_tls_pkcs12_passphrase` (use ansible-vault to protect)

For TLS via key/cert files, the following additional variables must be
provided:

- `keycloak_tls_ca_certificate` (local path to CA cert file)
- `keycloak_tls_cert` (local path to TLS server cert file)
- `keycloak_tls_key` (local path to TLS server key file)

See `roles/keycloak/defaults/main.yml` for a list of other variable
defaults that one may want to override.

TODO
----
- Add example playbook that uses ansible-freeipa to create keycloak service and get cert
- Add example playbook that creates realm/client using keycloak_client module
- Add ability to configure IdM as an identity backend for keycloak via SSSD provider
