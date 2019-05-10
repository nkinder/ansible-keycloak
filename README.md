# Ansible Keycloak Role

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

## Usage

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

  `ansible-playbook ... --extra-vars "keycloak_force_install=yes"`

## Controlling the location of the Keycloak archive

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

## Variables

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

- `keycloak_tls_pkcs12` (path to PKCS12 bundle)
- `keycloak_tls_pkcs12_passphrase` (use ansible-vault to protect)
- `keycloak_tls_pkcs12_alias` (name of key/cert in `keycloak_tls_pkcs12`)

For TLS via key/cert files, the following additional variables must be
provided:

- `keycloak_tls_cert` (path to TLS server cert file)
- `keycloak_tls_key` (path to TLS server key file)

**NOTE:** the source TLS files (`keycloak_tls_pkcs12`,
`keycloak_tls_cert`, `keycloak_tls_key`) used to create the Keycloak
keystore may reside either on the Ansible controller (i.e. local) or
on the remote target host. By default the role assumes the source TLS
files are local. If however the source TLS files are located on the
remote target set the variable `keycloak_tls_files_on_target` to True.

You can control timeout values with the following variables:

`keycloak_startup_timeout`: Number of seconds to wait for Keycloak to
start.

`keycloak_jboss_config_connect_timeout`: Number of milliseconds to
wait for jboss configuration utilityto connect to wildfly server.

`keycloak_jboss_config_command_timeout`: Number of seconds to wait for
jboss configuration utility to complete each command executed in
configuration file

See `roles/keycloak/defaults/main.yml` for a list of other variable
defaults that one may want to override.

## Testing

In-tree tests are provided that use molecule to test the role against
docker containers.  These tests are designed to be used by CI, but
they can also be run locally to test it out while developing.

### Role name issue

> **WARNING**: For in-tree testing the directory name **must** match the
  Ansible Galaxy role name

> **NOTE**: In these examples it is assumed the repo has been cloned
  into `~/src/`

Ansible expects a role to have a prescribed directory structure. For
example if the role is named `my_role` Ansible will expect to find a
directory named `my_role` in the `ANSIBLE_ROLES_PATH` with
sub-directories similar to this:

```
my_role/
├── defaults
├── files
├── handlers
├── meta
├── tasks
├── templates
├── tests
└── vars
```

By convention most Galaxy roles are created in a git repo whose
top level directory contains the above role sub-directories thus the
directory created when cloning the git repo becomes the role
name. Using the above example the git repo would be named `my_role`.
However, most git repositories **are not named identically as their
Galaxy role name**. If you try to run `molecule test` at top level of
a git tree cloned with the default repo name you will get a `role not
found` error like this:

```shell
ERROR! the role 'nkinder.keycloak' was not found in /home/$USER/src/ansible-keycloak/molecule/default/roles:/tmp/molecule/ansible-keycloak/default/roles:/home/$USER/src:/home/$USER/src/ansible-keycloak/molecule/default

The error appears to have been in '/home/$USER/src/ansible-keycloak/molecule/default/playbook.yml': line 6, column 7, but may
be elsewhere in the file depending on the exact syntax problem.

The offending line appears to be:

  roles:
    - role: nkinder.keycloak
      ^ here
```
The solution to this is simple, when cloning the git repo provide a
directory name that matches the role name in the
`molecule/default/playbook.yml` file.

To determine the role name molecule will use find it's definition in
`molecule/default/playbook.yml`:

```yaml
  roles:
    - role: nkinder.keycloak
```

Hence the role name molecule will use is `nkinder.keycloak`.

Thus to clone this repo do this:

```shell
$ cd ~/src
$ git clone git@github.com:nkinder/ansible-keycloak.git nkinder.keycloak
```

Of course if you're cloning from a github fork you'll need to adjust
the repo URL accordingly.

You'll need to cd into the cloned git repo:

```shell
$ cd nkinder.keycloak
```

When you run `molecule test` from the top level directory of the git
repo molecule will get the current working directory path and from
that find the *parent* directory path, it will then set the
`ANSIBLE_ROLES_PATH` to include the parent directory of the role
repo. Thus when ansible runs it will find your role via the **directory
name** of your git repo. For example:

```shell
ANSIBLE_ROLES_PATH: /tmp/molecule/ansible-keycloak/default/roles:/home/$USER/src
```

Because ansible is looking for a role named `nkinder.keycloak` it will
search the directories listed in `ANSIBLE_ROLES_PATH` and utilize the
files in finds in `/home/$USER/src/nkinder.keycloak`.

> **NOTE**: molecule will stage some files in a temporary directory
  for the duration of a test run, this is the
  `MOLECULE_EPHEMERAL_DIRECTORY`
  and in our examples it is `/tmp/molecule/ansible-keycloak/default`,
  hence the first path in the above `ANSIBLE_ROLES_PATH`.
    

> **NOTE**: git does not require the directory name of the top level
  directory in the repo match the name of the git repo.
  Even though the directory name of the cloned repo will be
  different than the repo name all git operations will work as
  expected because the repo URL is specified as a `remote` in
  `.git/config`.

Testing is best done by installing molecule in a virtualenv:

```shell
  $ virtualenv .venv
  $ source .venv/bin/activate
  $ pip install molecule docker
````

> **NOTE**: You **must** name the virtualenv directory `.venv` because
  molecule expects this.

> **WARNING**: If you had already cloned the repo under as the repo
  name instead of the role name you can rename the repo directory
  however if you had created `.venv` before the rename you must remove
  `.venv` and recreate it because the virtualenv will have embedded
  the old pathnames.

By default molecule uses docker to build and run test images. You will
need docker installed and have the docker daemon running:

```shell
# To install the docker package
$ sudo dnf install docker

#To start the Docker service use:
$ sudo systemctl start docker

# Verify that Docker was correctly installed and is running by running
# the Docker hello-world image.

$ sudo docker run hello-world

# To optionally start the Docker daemon at boot
$ sudo systemctl enable docker
```

It is required to run the tests as a user who is authorized to run the
`docker` command without using sudo.  This is typically accomplished
by adding your user to the 'docker' group on your system.

```shell
$ sudo groupadd docker
$ sudo gpasswd -a $USER docker
$ sudo systemctl restart docker

# Group membership is evaluated when you create a new login session.
# Unless you log out and back in again you'll need to use `newgrp`
# to add your `docker` group membership in the current login session.
$ newgrp docker
```

Additionally, there is a challenge around python-libselinux on
platforms that use SELinux.  If you are using a virtualenv, you need
to make sure that the selinux python module is available in the
virtualenv.  Even if it is installed on your ansible controller host
and the target host, some of the tasks that are delegated to the
locahost will use the virtualenv.  The selinux module can't be
installed via pip.  A workaround for this is to copy the entire
`selinux` directory from your system site-packages location
(/usr/lib64/$PY_VERSION/site-packages/selinux on x86_64) into the
virtualenv site-packages directory.  You will also need to copy the
selinux.so binary from file from site-packages directory (it is *not*
located inside the `selinux` directory). The name of the selunux.so
varies between Python2 and Python3.

In Python2 the basename of the .so is `_selinux.so`, for Python 2.7 on
x86_64 this would be this .so:

```
/usr/lib64/python2.7/site-packages/_selinux.so
```
In Python3 the basename of the .so appends cpython, the python
version, the arch and the OS, for Python 3.7 on x86_64 it would look
like this:

```
/usr/lib64/python3.7/site-packages/_selinux.cpython-37m-x86_64-linux-gnu.so
```

Once your virtualenv is properly set up, the tests can be run with these commands:

```
$ cd nkinder.keycloak # Your repo whose directory matches the Galaxy role name
$ molecule test
```

> **HINT**: Adding `--debug` to the `molecule` command can aid in
  diagnosing problems as well as capturing the output in a log file,
  you might want to do this:

```
$ molecule --debug test 2>&1 | tee molecule.log
```

By default, the test target will be the latest `centos` image from
Docker Hub.  You can test against a different image/tag like so:

```shell
$ MOLECULE_DISTRO="fedora:28" molecule test`
```

## TODO

- Add example playbook that uses ansible-freeipa to create keycloak service and get cert
- Add example playbook that creates realm/client using keycloak_client module
- Add ability to configure IdM as an identity backend for keycloak via SSSD provider
