[![Version on Galaxy](https://img.shields.io/badge/available%20on%20ansible%20galaxy-jonaspammer.apache2-brightgreen)](https://galaxy.ansible.com/jonaspammer/apache2) [![Testing CI](https://github.com/JonasPammer/ansible-role-apache2/actions/workflows/ci.yml/badge.svg)](https://github.com/JonasPammer/ansible-role-apache2/actions/workflows/ci.yml)

An Ansible role for installing Apache2, enabling/disabling modules, configuring its defaults and creating virtual hosts.

# 🔎 Metadata

Below you can find information on…

- the role’s required Ansible version

- the role’s supported platforms

- the role’s [role dependencies](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html#role-dependencies)

**[meta/main.yml](meta/main.yml)**

    ---
    galaxy_info:
      role_name: "apache2"
      description: "An ansible role for installing Apache2, enabling/disabling modules, configuring its defaults and creating virtual hosts."

      author: "jonaspammer"
      license: "MIT"

      min_ansible_version: "2.11"
      platforms:
        - name: EL # (Enterprise Linux)
          versions:
            - "7" # actively tested: centos7
            - "8" # actively tested: rockylinux8, centos8
        - name: Fedora
          versions:
            - "35"
        - name: Debian
          versions:
            - buster # debian10 (actively tested)
            - bullseye # debian11 (actively tested)
        - name: Ubuntu
          versions:
            - xenial # ubuntu1604 (actively tested)
            - bionic # ubuntu1804 (actively tested)
            - focal # ubuntu2004 (actively tested)
        - name: Solaris
          versions:
            - "11.3" # warning: not actively tested

      galaxy_tags:
        - web
        - apache
        - webserver
        - html
        - httpd

    dependencies: []

    allow_duplicates: true

# 📌 Requirements

The Ansible User needs to be able to `become`.

If you are using SSL/TLS ([programlisting_title](#apache_vhosts_ssl)), you will need to provide your own certificate and key files.

If you are using Apache with PHP, I recommend using the [geerlingguy.php](https://github.com/geerlingguy/ansible-role-php/) role to install PHP, and you can either use `mod_php` (by adding the proper package, e.g. `libapache2-mod-php5` for Ubuntu, to `php_packages`), or by also using [`geerlingguy.apache-php-fpm` ](https://github.com/geerlingguy/ansible-role-apache-php-fpm) to connect Apache to PHP via FPM. Please consult the README’s of the linked roles for more specific information.

When targeting Solaris-based systems, the [`community.general` collection](https://galaxy.ansible.com/community/general) (containing the `pkg5` module) must be installed on the Ansible controller.

When targeting Suse-based systems, [`community.general` collection](https://galaxy.ansible.com/community/general) (containing the `zypper` module) must be installed on the Ansible controller.

# 📜 Role Variables

    apache_mods_enabled:
      - rewrite
      - ssl
    apache_mods_disabled: []

(Debian/RHEL only) Apache mods to enable or disable (these will be symlinked into the appropriate location). Consult the `mods-available` (Debian) / `conf.modules.d` (RHEL) directory inside [apache’s root directory](#apache__server_root_dir) for all the available mods.

    apache_listen_ip: "*"
    apache_listen_port: 80
    apache_listen_port_ssl: 443

The IP address and ports on which apache should be listening. Useful if you have another service (like a reverse proxy) listening on port 80 or 443 and need to change the defaults.

    apache_remove_default_vhost: false

On Debian/Ubuntu, a default virtualhost is included in Apache’s configuration. Set this to `true` to remove that default.

    apache_state: started

Set initial apache state. Recommended values: `started` or `stopped`

    apache_enabled: true

Set initial apache service status. Recommended values: `true` or `false`

    apache_restart_state: restarted

Sets the state to put apache in when a configuration change was made (i.e., when the `restart apache` handler has been called). Recommended values: `restarted` or `reloaded`

    apache_default_favicon: favicon.ico

Path to a file on the local Ansible Controller to be copied to the server and used by Apache as a default favicon.

## Role Variables used for installation

    apache_packages: [OS-specific by default, see /defaults directory]

A list of package names for installing Apache2 and most-necessary utilities.

    apache_packages_state: present

If you have enabled any additional repositories such as [`ondrej/apache2`](https://launchpad.net/~ondrej/+archive/ubuntu/apache2), [`EPEL`](https://fedoraproject.org/wiki/EPEL), or [`remi`](http://rpms.remirepo.net/), you may want an easy way to upgrade versions. To ensure so, set this to `latest`.

    apache_enablerepo: ""

(RHEL/CentOS only) The [repository](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_module.html#parameter-enablerepo) to use when installing Apache. If you’d like later versions of Apache than are available in the OS’s core repositories, use a repository like [EPEL](https://fedoraproject.org/wiki/EPEL) (which can be installed with the `repo-epel` role).

## Role Variables used to create Virtual Hosts

Head over to the [📚 Example Playbook Usages](#example_playbooks)-Section for examples showing how the produced VirtualHost-File may look like.

This role tries to ensure a **working** apache configuration by running [syntax tests for all configuration files (`-t`)](https://httpd.apache.org/docs/2.4/programs/httpd.html) and reverting the generated virtualhost if an error occurred.

    apache_create_vhosts: true
    apache_vhosts_filename: "vhosts.conf"
    apache_vhosts_template: "vhosts.conf.j2"

If set to `true`, a vhosts file managed by the variables of this role (see below), is created and placed in the Apache configuration folder. If set to `false`, you can place your own vhosts file into Apache’s configuration folder and skip the convenient (but more basic) one added by this role.

You can also override the template used and set a path to your own template, if you need to further customize the layout of your VirtualHost.

    apache_global_vhost_settings: |
      DirectoryIndex index.php index.html

This variable gets used **_outside any &lt;VirtualHost&gt; Directive_** in the generated virtualhost file.

You hereby change the configurations applied to Apache’s general context (instead of changing the configurations applied to, for example, a `<VirtualHost>`/ `<Directory>`/…).

A thing to understand with this default value is that **the `DirectoryIndex` does not _set_ but rather _append_** (Meaning we do not reverse any other configuration made), as noted on its Documentation page:

> Multiple `DirectoryIndex` directives within the same context will add to the list of resources to look for rather than replace.
>
> — https://httpd.apache.org/docs/2.4/mod/mod\_dir.html

    apache_vhosts:
      - servername: "local.dev"
        documentroot: "/var/www/html"

For each entry in this list, a `<VirtualHost>`-Directive listening to `{{ apache_listen_ip }}:{{ apache_listen_port }}` will be generated.

Each entry of a list may have the following properties (Consult the [📚 Example Playbook Usages](#example_playbooks)-Section for Examples. Consult the linked official documentation pages for the documentation of the actual Apache Directives they represent).

`servername` (required); `serveralias`; `serveradmin`; `documentroot`; `documentroot__allowoverride`
`AllowOverride`-Directive used inside the `<Directory>` of the `DocumentRoot`.
Defaults to the value of `apache_vhosts_default_documentroot__allowoverride`.

`documentroot__options`
`Options`-Directive used inside the `<Directory>` of the `DocumentRoot`.
Defaults to the value of `apache_vhosts_default_documentroot__options`.

[`logformat`](https://httpd.apache.org/docs/2.4/mod/mod_log_config.html#logformat); `loglevel`

<!-- -->

`errorlog`
Either a string (representing the path. does not get automatically quoted) or a complex data type:

`path`
Path. Gets enquoted in `"`.

`extra`
Additional String to append after `path`.

`extra_parameters`
This variable gets inserted as-is **before** the actual `ErrorLog` statement (with an indent of 2).

The use case for this parameter may be to enable Conditional Logs using `SetEnvIf` / `SetEnv` or setting a custom `LogFormat` for this ErrorLog [Apache’s core Documentation](https://httpd.apache.org/docs/2.4/logs.html).

<!-- -->

`customlogs`
Array of CustomLogs. Each Entry may either be a string (does not get automatically quoted) or a complex data type:

`path`
Path. Gets enquoted in `"`.

`extra`
Additional String to append after `path`. Does not get quoted (to allow for the complex additional optional parameters of CustomLog one may want to supply).

`extra_parameters`
This variable gets inserted as-is **before** the actual `CustomLog` statement (with an indent of 2).

The use case for this parameter may be to enable Conditional Logs using `SetEnvIf` / `SetEnv` or setting a custom `LogFormat` for this specifc CustomLog as per [Apache’s mod_log_config Documentation](https://httpd.apache.org/docs/2.4/logs.html).

`extra_parameters`
This variable gets inserted as-is into the very end of the looped `<VirtualHost>` (with an indent of 2).

<!-- -->

    apache_vhosts_ssl: []

For each entry in this list, a `<VirtualHost>`-Directive listening to `{{ apache_listen_ip }}:{{ apache_listen_port_ssl }}` will be generated.

Each entry of a list may have the following properties (Consult the [📚 Example Playbook Usages](#example_playbooks)-Section for Examples) (Consult the linked official documentation pages for the documentation of the actual Apache Directives they represent).

`servername` (required); `serveralias`; `serveradmin`; `documentroot`; `documentroot__allowoverride`
`AllowOverride`-Directive used inside the `<Directory>` of the `DocumentRoot`.
Defaults to `apache_vhosts_default_documentroot__allowoverride`.

`documentroot__options`
`Options`-Directive used inside the `<Directory>` of the `DocumentRoot`. Defaults to `apache_vhosts_default_documentroot__options`.

`no_actual_ssl`
If set to True, the `<VirtualHost>` will have no SSL\* Options. Used only when you want a http-to-https redirect you defined in `extra_parameters`.

[ssl_certificate_file](https://httpd.apache.org/docs/current/mod/mod_ssl.html#sslcertificatefile) (required); [ssl_certificate_key_file](https://httpd.apache.org/docs/current/mod/mod_ssl.html#sslcertificatekeyfile) (required); [ssl_certificate_chain_file](https://httpd.apache.org/docs/current/mod/mod_ssl.html#sslcertificatechainfile)
_Please note that this Deprecated._

[`logformat`](https://httpd.apache.org/docs/2.4/mod/mod_log_config.html#logformat); `loglevel`; `errorlog`
Equivalent of [apache_vhosts.errorlog](#apache_vhosts__errorlog).

`customlogs`
Array of CustomLogs. Equivalent of [apache_vhosts.customlogs](#apache_vhosts__customlogs).

`extra_parameters`
This variable gets inserted as-is into the very end of the looped `<VirtualHost>` (with an indent of 2).

<!-- -->

    apache_ignore_missing_ssl_certificate: true

If set to `false`, a given entry of `apache_vhosts_ssl` will only be generated if its `sslcertificatefile` exists.

    apache_ssl_protocol: "All -SSLv2 -SSLv3"
    apache_ssl_cipher_suite: "AES256+EECDH:AES256+EDH"

These variable are used as default for every `apache_vhosts_ssl`. They are named the same way as used in said Role variables (except for their prefix of course). Consult [ Apache’s Documentation](https://httpd.apache.org/docs/current/mod/mod_ssl.html) for the documentation of the actual Apache Directives they represent.

    apache_vhosts_default_documentroot__allowoverride: "All"
    apache_vhosts_default_documentroot__options: "-Indexes +FollowSymLinks"

# 📜 Facts/Variables defined by this role

Each variable listed in this section is dynamically defined when executing this role (and can only be overwritten using `ansible.builtin.set_facts`) _and_ is meant to be used not just internally.

**Example Usage outside this role:**

    # handlers file for roles.xyz
    - name: restart apache2
      ansible.builtin.service:
        name: "{{ apache__service | default('apache2') }}"
        state: restarted

Executable Name and Directory of the `apache2` command.

Directory containing all Apache2 configuration (in `/etc`).

When working with any of the below configuration values you need to remember:

> The Apache 2 web server configuration in **Debian is quite different to upstream’s suggested way** to configure the web server. This is because Debian’s default Apache2 installation attempts to make adding and removing modules, virtual hosts, and extra configuration directives as flexible as possible, in order to make automating the changes and administering the server as easy as possible.
>
> — Comment found in a Debian 10's /etc/apache2/apache2.conf

This means that the `apache__server_root_dir` **on Debian** looks like this:

**`tree /etc/apache2` of a fresh Debian 10 machine after apache2 install**

    .
    ├── apache2.conf
    ├── conf-available
    │   ├── charset.conf
    │   ├── localized-error-pages.conf
    │   ├── other-vhosts-access-log.conf
    │   ├── php7.4-fpm.conf
    │   ├── security.conf
    │   └── serve-cgi-bin.conf
    ├── conf-enabled
    │   ├── charset.conf -> ../conf-available/charset.conf
    │   └── …
    ├── envvars
    ├── magic
    ├── mods-available
    │   ├── access_compat.load
    │   ├── alias.load
    │   ├── alias.conf
    │   └── …
    ├── mods-enabled
    │   ├── access_compat.load -> ../mods-available/access_compat.load
    │   ├── alias.conf -> ../mods-available/alias.conf
    │   ├── alias.load -> ../mods-available/alias.load
    │   └── …
    ├── ports.conf
    ├── sites-available
    │   ├── 000-default.conf
    │   └── default-ssl.conf
    └── sites-enabled
        └── 000-default.conf -> ../sites-available/000-default.conf

While _on other systems it looks like this_:

**`tree /etc/apache2` of a fresh CentOS 8 machine after apache2 install**

    .
    ├── conf
    │   ├── httpd.conf
    │   └── magic
    ├── conf.d
    │   ├── autoindex.conf
    │   ├── ssl.conf
    │   ├── userdir.conf
    │   └── welcome.conf
    ├── conf.modules.d
    │   ├── 00-base.conf
    │   ├── 00-dav.conf
    │   ├── 00-lua.conf
    │   ├── 00-mpm.conf
    │   ├── 00-optional.conf
    │   ├── 00-proxy.conf
    │   ├── 00-ssl.conf
    │   ├── 00-systemd.conf
    │   ├── 01-cgi.conf
    │   ├── 10-h2.conf
    │   ├── 10-proxy_h2.conf
    │   └── README
    ├── logs -> ../../var/log/httpd
    │   └── …
    └── modules -> ../../usr/lib64/httpd/modules
        ├── mod_access_compat.so
        ├── mod_actions.so
        ├── mod_alias.so
        └── …

Apache2’s primary configuration file, which [ `Include`](http://httpd.apache.org/docs/2.4/mod/core.html#include)'s all the other files and contains some other Directives itself.

Taking a look into how what is Include’ed

Debian’s Apache2 Include Directives as found in `apache__primary_configuration_file_path`:

    # Include module configuration:
    IncludeOptional mods-enabled/*.load
    IncludeOptional mods-enabled/*.conf

    # Include list of ports to listen on
    Include ports.conf

    # Include of directories ignores editors' and dpkg's backup files,
    # Include generic snippets of statements

    IncludeOptional conf-enabled/*.conf
    # Include the virtual host configurations:
    IncludeOptional sites-enabled/*.conf

RHEL’s Apache2 Include Directives as found in `apache__primary_configuration_file_path` on a CentOS 8 Machine:

    # Dynamic Shared Object (DSO) Support
    #
    # To be able to use the functionality of a module which was built as a DSO you
    # have to place corresponding `LoadModule' lines at this location so the
    # directives contained in it are actually available _before_ they are used.
    # Statically compiled modules (those listed by `httpd -l') do not need
    # to be loaded here.
    #
    # Example:
    # LoadModule foo_module modules/mod_foo.so
    Include conf.modules.d/*.conf

    # Supplemental configuration:
    IncludeOptional conf.d/*.conf

Apache2 Configuration File that houses the directives used to determine listening ports for incoming connections.

On some systems this is the same as `apache__primary_configuration_file_path`, but on some it is an own file which is being [ `Include`](http://httpd.apache.org/docs/2.4/mod/core.html#include)-ed by said `apache__primary_configuration_file_path`.

Directory which houses all [ `Include`](http://httpd.apache.org/docs/2.4/mod/core.html#include)-ed files.

This directory may not be `Include`-ed itself but have sub-directories that are being `Include`-ed. Consult the NOTE/TIP found in [sidebar_title](#apache__primary_configuration_file_path) to know what Directories are being `Include`-ed by default on different OS’es.

Directory in `/var` used by default for all virtual hosts.

The below output shows the typical default file contents of this folder for the major distros:

**RedHat**

    [root@instance-py3-ansible-5 /]# ls -l /var/log/httpd/
    total 8
    -rw-r--r-- 1 root root   0 Jun 11 11:16 access_log
    -rw-r--r-- 1 root root 980 Jun 11 11:16 error_log
    -rw-r--r-- 1 root root   0 Jun 11 11:16 ssl_access_log
    -rw-r--r-- 1 root root 328 Jun 11 11:16 ssl_error_log
    -rw-r--r-- 1 root root   0 Jun 11 11:16 ssl_request_log

**Debian**

    root@instance-py3-ansible-5-debian10:/# ls -l /var/log/apache2
    total 4
    -rw-r----- 1 root adm     0 Aug 29 10:17 access.log
    -rw-r----- 1 root adm  2133 Aug 29 10:18 error.log
    -rw-r--r-- 1 root root    0 Aug 29 10:18 local2-error.log
    -rw-r----- 1 root adm     0 Aug 29 10:17 other_vhosts_access.log

# 🏷️ Tags

Tasks are tagged with the following [tags](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tags.html#adding-tags-to-roles):

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th style="text-align: left;">Tag</th>
<th style="text-align: left;">Purpose</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td colspan="2" style="text-align: left;"><p>This role does not have officially documented tags yet.</p></td>
</tr>
</tbody>
</table>

You can use Ansible to skip tasks, or only run certain tasks by using these tags. By default, all tasks are run when no tags are specified.

# 👫 Dependencies

# 📚 Example Playbook Usages

This role is part of [ many compatible purpose-specific roles of mine](https://github.com/JonasPammer/ansible-roles).

The machine needs to be prepared. In CI, this is done in `molecule/resources/prepare.yml` which sources its soft dependencies from `requirements.yml`:

**[molecule/resources/prepare.yml](molecule/resources/prepare.yml)**

    ---
    - name: prepare
      hosts: all
      become: true
      gather_facts: false

      roles:
        - role: jonaspammer.bootstrap
        #    - role: jonaspammer.core_dependencies

The following diagram is a compilation of the "soft dependencies" of this role as well as the recursive tree of their soft dependencies.

![requirements.yml dependency graph of jonaspammer.apache2](https://raw.githubusercontent.com/JonasPammer/ansible-roles/master/graphs/dependencies_apache2.svg)

- The following yaml:

      roles:
        - jonaspammer.apache2

  generates the following VirtualHost:

      # Ansible managed
      DirectoryIndex index.php index.html
      <VirtualHost *:80>
          ServerName local.dev
          DocumentRoot "/var/www/html"

          <Directory "/var/www/html">
              AllowOverride All
              Options -Indexes +FollowSymLinks
              Require all granted
          </Directory>
      </VirtualHost>

  For Reference, this is the default vhost shipped with Debian/Ubuntu systems (which can be removed by setting `apache_remove_default_vhost` to true)

      <VirtualHost *:80>
              ServerAdmin webmaster@localhost
              DocumentRoot /var/www/html

              ErrorLog ${APACHE_LOG_DIR}/error.log
              CustomLog ${APACHE_LOG_DIR}/access.log combined
      </VirtualHost>

Given no role configuration, the deviance’s from just installing Apache2 yourself are

- certain modules get activated by default (``).

- the system will have the above demonstrated VirtualHost

- On initial install, a file with the name of `favicon.ico` (sourced from [programlisting_title](#apache_default_favicon)) will be placed into `/var/www/html` if there was no file with said name before. This favicon, by default, resembles the Ansible logo as found on Wikimedia.

_Please note that this role does **not** delete the contents of `/var/www/html` (not even if it got created by/after apache2 initial install)._

- The following yaml:

      roles:
        - jonaspammer.apache2

      vars:
        apache_vhost_filename: "local2.dev.conf"
        apache_vhosts:
          - servername: "wwww.local2.dev"
            loglevel: info
            errorlog: "{{ apache__default_log_dir }}/local2-error.log"
            customlog:
              path: "${{ apache__default_log_dir }}/local2-access.log"
              extra: "combined"

  generates the following VirtualHost:

      # Ansible managed.

      TODO

The pipe symbol at the end of a line in YAML signifies that any indented text that follows should be interpreted as a multi-line scalar value. See [yaml-multiline.info](https://yaml-multiline.info/) for interactive explanation.

- The following yaml:

      roles:
        - jonaspammer.apache2

      vars:
        apache_vhost_filename: "myvhost.conf"
        apache_vhosts:
          - servername: "www.local.dev"
            serveralias: "local.dev"
            documentroot: "/var/www/html"
            extra_parameters: |
                # Redirect all requests to 'www' subdomain. Apache 2.4+
                RewriteEngine On
                RewriteCond %{HTTP_HOST} !^www\. [NC]
                RewriteRule ^(.*)$ %{REQUEST_SCHEME}://www.%{HTTP_HOST}%{REQUEST_URI} [R=302,L]

  generates the following VirtualHost:

      # Ansible managed.

      TODO

- The following yaml:

      roles:
        - jonaspammer.apache2

      vars:
        apache_vhost_filename: "myvhost.conf"
        apache_vhosts:
          - servername: "srvcmk.intra.jonaspammer.com"
            extra_parameters: |
              Redirect / {{ checkmk_site_url }}

  generates the following VirtualHost:

      # Ansible managed.
      DirectoryIndex index.php index.html
      <VirtualHost *:80>
          ServerName srvcmk.intra.jonaspammer.com

          Redirect / http://srvcmk.intra.jonaspammer.at/master
      </VirtualHost>

_The apache2 role may be executed multiple times in a play, with the primary purpose of [this allowance](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html#using-allow-duplicates-true) being to be able to create virtualhosts._

    - tasks:
        #...
        - name: Generate Apache2 VirtualHost.
          ansible.builtin.include_role: "apache2"
          vars:
            apache_vhost_filename: "myapp.conf"
            apache_vhosts:
              - servername: "www.myapp.dev"
                serveralias: "myapp.dev"
                DocumentRoot: "/opt/myapp"
        #...

# 📝 Development

[![Conventional Commits](https://img.shields.io/badge/Conventional%20Commits-1.0.0-yellow.svg)](https://conventionalcommits.org) [![pre-commit.ci status](https://results.pre-commit.ci/badge/github/JonasPammer/ansible-role-apache2/master.svg)](https://results.pre-commit.ci/latest/github/JonasPammer/ansible-role-apache2/master)

## 📌 Development Machine Dependencies

- Python 3.8 or greater

- Docker

## 📌 Development Dependencies

Development Dependencies are defined in a [pip requirements file](https://pip.pypa.io/en/stable/user_guide/#requirements-files) named `requirements-dev.txt`. Example Installation Instructions for Linux are shown below:

    # "optional": create a python virtualenv and activate it for the current shell session
    $ python3 -m venv venv
    $ source venv/bin/activate

    $ python3 -m pip install -r requirements-dev.txt

## ℹ️ Ansible Role Development Guidelines

Please take a look at my [ Ansible Role Development Guidelines](https://github.com/JonasPammer/cookiecutter-ansible-role/blob/master/ROLE_DEVELOPMENT_GUIDELINES.adoc).

If interested, I’ve also written down some [ General Ansible Role Development (Best) Practices](https://github.com/JonasPammer/cookiecutter-ansible-role/blob/master/ROLE_DEVELOPMENT_TIPS.adoc).

## 🔢 Versioning

Versions are defined using [Tags](https://git-scm.com/book/en/v2/Git-Basics-Tagging), which in turn are [recognized and used](https://galaxy.ansible.com/docs/contributing/version.html) by Ansible Galaxy.

**Versions must not start with `v`.**

When a new tag is pushed, [ a GitHub CI workflow](https://github.com/JonasPammer/ansible-role-apache2/actions/workflows/release-to-galaxy.yml) (![Release CI](https://github.com/JonasPammer/ansible-role-apache2/actions/workflows/release-to-galaxy.yml/badge.svg)) takes care of importing the role to my Ansible Galaxy Account.

## 🧪 Testing

Automatic Tests are run on each Contribution using GitHub Workflows.

The Tests primarily resolve around running [Molecule](https://molecule.readthedocs.io/en/latest/) on a varying set of linux distributions and using various ansible versions, as detailed in [JonasPammer/ansible-roles](https://github.com/JonasPammer/ansible-roles).

The molecule test also includes a step which lints all ansible playbooks using [`ansible-lint`](https://github.com/ansible/ansible-lint#readme) to check for best practices and behaviour that could potentially be improved.

To run the tests, simply run `tox` on the command line. You can pass an optional environment variable to define the distribution of the Docker container that will be spun up by molecule:

    $ MOLECULE_DISTRO=centos7 tox

For a list of possible values fed to `MOLECULE_DISTRO`, take a look at the matrix defined in [.github/workflows/ci.yml](.github/workflows/ci.yml).

### 🐛 Debugging a Molecule Container

1.  Run your molecule tests with the option `MOLECULE_DESTROY=never`, e.g.:

        $ MOLECULE_DESTROY=never MOLECULE_DISTRO=ubuntu1604 tox -e py3-ansible-5
        ...
          TASK [ansible-role-pip : (redacted).] ************************
          failed: [instance-py3-ansible-5] => changed=false
        ...
         ___________________________________ summary ____________________________________
          pre-commit: commands succeeded
        ERROR:   py3-ansible-5: commands failed

2.  Find out the name of the molecule-provisioned docker container:

        $ docker ps
        30e9b8d59cdf   geerlingguy/docker-debian10-ansible:latest   "/lib/systemd/systemd"   8 minutes ago   Up 8 minutes                                                                                                    instance-py3-ansible-5

3.  Get into a bash Shell of the container, and do your debugging:

        $ docker exec -it 30e9b8d59cdf /bin/bash

        root@instance-py3-ansible-2:/#
        root@instance-py3-ansible-2:/# python3 --version
        Python 3.8.10
        root@instance-py3-ansible-2:/# ...

    If the failure you try to debug is part of `verify.yml` step and not the actual `converge.yml`, you may want to know that the output of ansible’s modules (`vars`), hosts (`hostvars`) and environment variables have been stored into files on both the provisioner and inside the docker machine under: \* `/var/tmp/vars.yml` \* `/var/tmp/hostvars.yml` \* `/var/tmp/environment.yml` `grep`, `cat` or transfer these as you wish!

    You may also want to know that the files mentioned in the admonition above are attached to the **GitHub CI Artifacts** of a given Workflow run.
    This allows one to check the difference between runs and thus help in debugging what caused the bit-rot or failure in general. image::https://user-images.githubusercontent.com/32995541/178442403-e15264ca-433a-4bc7-95db-cfadb573db3c.png\[\]

4.  After you finished your debugging, exit it and destroy the container:

        root@instance-py3-ansible-2:/# exit

        $ docker stop 30e9b8d59cdf

        $ docker container rm 30e9b8d59cdf
        or
        $ docker container prune

## 🧃 TIP: Containerized Ideal Development Environment

This Project offers a definition for a "1-Click Containerized Development Environment".

This Container even allow one to run docker containers inside of them (Docker-In-Docker, dind), allowing for molecule execution.

To use it:

1.  Ensure you fullfill the [ the System requirements of Visual Studio Code Development Containers](https://code.visualstudio.com/docs/remote/containers#_system-requirements), optionally following the _Installation_-Section of the linked page section.
    This includes: Installing Docker, Installing Visual Studio Code itself, and Installing the necessary Extension.

2.  Clone the project to your machine

3.  Open the folder of the repo in Visual Studio Code (_File - Open Folder…_).

4.  If you get a prompt at the lower right corner informing you about the presence of the devcontainer definition, you can press the accompanying button to enter it. **Otherwise,** you can also execute the Visual Studio Command `Remote-Containers: Open Folder in Container` yourself (_View - Command Palette_ → _type in the mentioned command_).

I recommend using `Remote-Containers: Rebuild Without Cache and Reopen in Container` once here and there as the devcontainer feature does have some problems recognizing changes made to its definition properly some times.

You may need to configure your host system to enable the container to use your SSH Keys.

The procedure is described [ in the official devcontainer docs under "Sharing Git credentials with your container"](https://code.visualstudio.com/docs/remote/containers#_sharing-git-credentials-with-your-container).

## 🍪 CookieCutter

This Project shall be kept in sync with [the CookieCutter it was originally templated from](https://github.com/JonasPammer/cookiecutter-ansible-role) using [cruft](https://github.com/cruft/cruft) (if possible) or manual alteration (if needed) to the best extend possible.

> ![Official Example Usage of `cruft update`](https://raw.githubusercontent.com/cruft/cruft/master/art/example_update.gif)

### 🕗 Changelog

When a new tag is pushed, an appropriate GitHub Release will be created by the Repository Maintainer to provide a proper human change log with a title and description.

## ℹ️ General Linting and Styling Conventions

General Linting and Styling Conventions are [**automatically** held up to Standards](https://stackoverflow.blog/2020/07/20/linters-arent-in-your-way-theyre-on-your-side/) by various [`pre-commit`](https://pre-commit.com/) hooks, at least to some extend.

Automatic Execution of pre-commit is done on each Contribution using [`pre-commit.ci`](https://pre-commit.ci/)[\*](#note_pre-commit-ci). Pull Requests even automatically get fixed by the same tool, at least by hooks that automatically alter files.

Not to confuse: Although some pre-commit hooks may be able to warn you about script-analyzed flaws in syntax or even code to some extend (for which reason pre-commit’s hooks are **part of** the test suite), pre-commit itself does not run any real Test Suites. For Information on Testing, see [🧪 Testing](#testing).

Nevertheless, I recommend you to integrate pre-commit into your local development workflow yourself.

This can be done by cd’ing into the directory of your cloned project and running `pre-commit install`. Doing so will make git run pre-commit checks on every commit you make, aborting the commit themselves if a hook alarm’ed.

You can also, for example, execute pre-commit’s hooks at any time by running `pre-commit run --all-files`.

# 💪 Contributing

![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square) [![Open in Visual Studio Code](https://img.shields.io/static/v1?logo=visualstudiocode&label=&message=Open%20in%20Visual%20Studio%20Code&labelColor=2c2c32&color=007acc&logoColor=007acc)](https://open.vscode.dev/JonasPammer/ansible-role-apache2)

The following sections are generic in nature and are used to help new contributors. The actual "Development Documentation" of this project is found under [📝 Development](#development).

## 🤝 Preamble

First off, thank you for considering contributing to this Project.

Following these guidelines helps to communicate that you respect the time of the developers managing and developing this open source project. In return, they should reciprocate that respect in addressing your issue, assessing changes, and helping you finalize your pull requests.

## 🍪 CookieCutter

This Project owns many of its files to [the CookieCutter it was originally templated from](https://github.com/JonasPammer/cookiecutter-ansible-role).

Please check if the edit you have in mind is actually applicable to the template and if so make an appropriate change there instead. Your change may also be applicable partly to the template as well as partly to something specific to this project, in which case you would be creating multiple PRs.

## 💬 Conventional Commits

A casual contributor does not have to worry about following [_the spec_](https://github.com/JonasPammer/JonasPammer/blob/master/demystifying/conventional_commits.adoc) [_by definition_](https://www.conventionalcommits.org/en/v1.0.0/), as pull requests are being squash merged into one commit in the project. Only core contributors, i.e. those with rights to push to this project’s branches, must follow it (e.g. to allow for automatic version determination and changelog generation to work).

## 🚀 Getting Started

Contributions are made to this repo via Issues and Pull Requests (PRs). A few general guidelines that cover both:

- Search for existing Issues and PRs before creating your own.

- If you’ve never contributed before, see [ the first timer’s guide on Auth0’s blog](https://auth0.com/blog/a-first-timers-guide-to-an-open-source-project/) for resources and tips on how to get started.

### Issues

Issues should be used to report problems, request a new feature, or to discuss potential changes **before** a PR is created. When you [ create a new Issue](https://github.com/JonasPammer/ansible-role-apache2/issues/new), a template will be loaded that will guide you through collecting and providing the information we need to investigate.

If you find an Issue that addresses the problem you’re having, please add your own reproduction information to the existing issue **rather than creating a new one**. Adding a [reaction](https://github.blog/2016-03-10-add-reactions-to-pull-requests-issues-and-comments/) can also help be indicating to our maintainers that a particular problem is affecting more than just the reporter.

### Pull Requests

PRs to this Project are always welcome and can be a quick way to get your fix or improvement slated for the next release. [In general](https://blog.ploeh.dk/2015/01/15/10-tips-for-better-pull-requests/), PRs should:

- Only fix/add the functionality in question **OR** address wide-spread whitespace/style issues, not both.

- Add unit or integration tests for fixed or changed functionality (if a test suite already exists).

- **Address a single concern**

- **Include documentation** in the repo

- Be accompanied by a complete Pull Request template (loaded automatically when a PR is created).

For changes that address core functionality or would require breaking changes (e.g. a major release), it’s best to open an Issue to discuss your proposal first.

In general, we follow the "fork-and-pull" Git workflow

1.  Fork the repository to your own Github account

2.  Clone the project to your machine

3.  Create a branch locally with a succinct but descriptive name

4.  Commit changes to the branch

5.  Following any formatting and testing guidelines specific to this repo

6.  Push changes to your fork

7.  Open a PR in our repository and follow the PR template so that we can efficiently review the changes.

# 🗒 Changelog

Please refer to the [Release Page of this Repository](https://github.com/JonasPammer/ansible-role-apache2/releases) for a human changelog of the corresponding [Tags (Versions) of this Project](https://github.com/JonasPammer/ansible-role-apache2/tags).

Note that this Project adheres to Semantic Versioning. Please report any accidental breaking changes of a minor version update.

# ⚖️ License

**[LICENSE](LICENSE)**

    MIT License

    Copyright (c) 2022, Jonas Pammer

    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:

    The above copyright notice and this permission notice shall be included in all
    copies or substantial portions of the Software.

    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
    SOFTWARE.
