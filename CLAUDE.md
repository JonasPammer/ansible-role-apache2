# CLAUDE.md

## Project Overview

This is an Ansible role (`jonaspammer.apache2`) for installing and configuring Apache2 HTTP server across multiple Linux distributions. The role handles OS-specific variations through a sophisticated variable lookup system and supports creating virtual hosts with safety validation.

**Minimum Ansible Version:** 2.11
**License:** MIT
**Based on:** geerlingguy's apache2 role with significant enhancements

## Development Commands

### Testing

```bash
# Run all tests (linting + molecule on default distro)
tox

# Test on specific distribution
MOLECULE_DISTRO=ubuntu2204 tox
MOLECULE_DISTRO=debian12 tox
MOLECULE_DISTRO=rockylinux9 tox

# Run tests with specific Ansible version
tox -e py3-ansible-9  # Ansible 9 (core 2.16)
tox -e py3-ansible-8  # Ansible 8 (core 2.15)

# Debug failed tests (keeps container alive)
MOLECULE_DESTROY=never MOLECULE_DISTRO=debian12 tox

# Show installed package versions during test
CI=true tox
```

### Linting

```bash
# Run all pre-commit hooks
pre-commit run --all-files

# Install pre-commit hooks locally
pre-commit install

# Run only yamllint
yamllint .

# Run only ansible-lint (part of molecule test)
ansible-lint
```

### Molecule Container Debugging

When a molecule test fails and `MOLECULE_DESTROY=never` was used:

```bash
# Find container name
docker ps

# Enter container
docker exec -it <container-id> /bin/bash

# Debug files are stored at:
# - /var/tmp/vars.yml (hostvars)
# - /var/tmp/environment.yml

# Clean up when done
docker stop <container-id>
docker container prune
```

### Development Environment

```bash
# Install development dependencies
python3 -m venv venv
source venv/bin/activate
python3 -m pip install -r requirements-dev.txt

# Or use the VS Code devcontainer (supports Docker-in-Docker for molecule)
# See DEVELOPMENT.adoc for full setup instructions
```

## Architecture

### Multi-OS Support Pattern

This role uses a **hierarchical variable lookup system** to handle OS differences. Variables are defined in `vars/main.yml` and `defaults/main.yml` with this lookup order:

1. `ansible_distribution` + `_` + `ansible_distribution_major_version` (e.g., `Amazon_NA`)
2. `ansible_os_family` + `_` + `ansible_distribution_major_version`
3. `ansible_distribution` (e.g., `Debian`, `RedHat`)
4. `ansible_os_family`
5. `default`

**Example from vars/main.yml:**
```yaml
_apache__service:
  RedHat: httpd
  Debian: apache2
  Suse: apache2
  Solaris: apache2
```

This pattern is used extensively for:
- Service names (`apache__service`)
- Daemon executables (`apache__daemon`, `apache__daemon_dir`)
- Configuration paths (`apache__server_root_dir`, `apache__server_conf_dir`)
- Package names (`apache_packages`)

### Task Execution Flow

The main task flow in `tasks/main.yml`:

1. **Validation:** `assert.yml` - validates all required variables
2. **OS-Specific Setup:** `setup-{Debian|RedHat|Suse|Solaris}.yml` - installs packages
3. **Favicon Setup:** Copies default favicon to `/var/www/html/`
4. **Version Detection:** Runs `apache2ctl -v` to determine Apache 2.2 vs 2.4
5. **Version-Specific Variables:** Includes `apache-22.yml` or `apache-24.yml`
6. **Port Configuration:** Updates `apache__ports_configuration_file`
7. **OS-Specific Configuration:** `configure-{Debian|RedHat|Suse|Solaris}.yml`
8. **Service Management:** Ensures apache state and enabled status

### OS-Specific Configuration Differences

**Debian/Ubuntu:**
- Uses `a2enmod`/`a2dismod` style (symlinks in `mods-enabled/`)
- VirtualHosts in `sites-available/` with symlinks to `sites-enabled/`
- Configuration structure: `/etc/apache2/` with separate directories

**RedHat/CentOS/Rocky:**
- Modules configured via files in `conf.modules.d/`
- VirtualHosts directly in `conf.d/`
- Configuration structure: `/etc/httpd/conf/` and `/etc/httpd/conf.d/`
- RHEL 8+ requires running `/usr/libexec/httpd-ssl-gencerts` for localhost SSL cert

**Key Task Files:**
- `configure-Debian.yml` - manages symlinks for mods and sites
- `configure-RedHat.yml` - creates module config files, generates SSL certs
- `configure-Suse.yml` - similar to RedHat
- `configure-Solaris.yml` - Solaris-specific configuration

### VirtualHost Generation Safety System

The role implements a **rollback mechanism** to ensure Apache never enters a broken state:

**Process in `_generate_vhost_standard.yml`:**

1. **Flush handlers** - ensure Apache is in known state
2. **Backup** - if vhost file exists, copy to tempfile
3. **Generate** - create new vhost from template (`templates/vhosts.conf.j2`)
4. **Test** - run `apache2ctl -t` syntax check
5. **Rollback on failure:**
   - If backup exists: restore old file
   - If no backup: delete generated file
6. **Fail** - if syntax check failed
7. **Notify restart handler** - only if syntax check passed and file changed

This prevents broken configurations from being deployed.

### VirtualHost Template System

The `vhosts.conf.j2` template handles both HTTP and HTTPS vhosts:

- **Jinja2 Macros:** `MACRO_LOGGING` for consistent log configuration
- **Apache Version Detection:** Uses `apache__vhosts_version` to render 2.2 vs 2.4 syntax
  - 2.2: `Order allow,deny` / `Allow from all`
  - 2.4: `Require all granted`
- **DocumentRoot handling:** Optional with defaults for `AllowOverride` and `Options`
- **SSL Configuration:** Separate loop for `apache_vhosts_ssl` with SSL directives
- **Extra Parameters:** Both vhost types support `extra_parameters` for custom directives

### Facts/Variables Exported by Role

These variables are set during role execution and can be used by other roles:

- `apache__service` - Service name (httpd/apache2)
- `apache__daemon` - Daemon executable name
- `apache__daemon_dir` - Directory containing daemon
- `apache__server_root_dir` - Apache configuration root
- `apache__server_conf_dir` - Directory for configuration includes
- `apache__primary_configuration_file` - Main config file path
- `apache__ports_configuration_file` - Ports configuration file
- `apache__default_log_dir` - Default log directory
- `apache_version` - Detected Apache version (e.g., "2.4.52")

## Supported Platforms

**Actively Tested Distributions (in CI):**
- Rocky Linux 8, 9
- Fedora 39
- Debian 11 (Bullseye), 12 (Bookworm)
- Ubuntu 20.04 (Focal), 22.04 (Jammy)

**Testing Strategy:**
- Matrix of distributions × Ansible versions (6, 7, 8, 9)
- Uses `geerlingguy/docker-*-ansible` images
- Parallel execution (max 4 concurrent)
- Weekly scheduled runs + PR/push triggers

## Key Role Features

### Allow Duplicates

The role has `allow_duplicates: true` in `meta/main.yml`, enabling it to be called multiple times in a playbook with different variables. Primary use case is creating multiple separate virtualhost files:

```yaml
- include_role: "apache2"
  vars:
    apache_vhosts_filename: "myapp.conf"
    apache_vhosts:
      - servername: "www.myapp.dev"
```

### Module Management

**Debian/Ubuntu:** Modules are enabled/disabled via symlinks
- Enabled: `mods-enabled/*.load` → `mods-available/*.load`
- Task creates/removes symlinks based on `apache_mods_enabled`/`apache_mods_disabled`

**RedHat/CentOS:** Modules configured via conf files
- Creates `conf.modules.d/99-ansible-{module}.conf` with LoadModule directive
- Removes files for disabled modules

### SSL Certificate Handling

- `apache_ignore_missing_ssl_certificate: true` (default) - generates SSL vhosts even if cert files don't exist
- When false: checks certificate existence with `stat` before generating vhost
- RHEL 8+: auto-generates localhost certificate if missing

## Project Structure

```
├── defaults/main.yml          # User-configurable variables with OS-specific defaults
├── vars/main.yml              # Internal variables (OS path/name mappings)
├── vars/apache-22.yml         # Apache 2.2 specific settings
├── vars/apache-24.yml         # Apache 2.4 specific settings
├── tasks/
│   ├── main.yml              # Main task orchestration
│   ├── assert.yml            # Variable validation
│   ├── setup-{OS}.yml        # OS-specific installation
│   ├── configure-{OS}.yml    # OS-specific configuration
│   └── _generate_vhost_standard.yml  # Safe vhost generation with rollback
├── templates/
│   └── vhosts.conf.j2        # VirtualHost Jinja2 template
├── handlers/main.yml          # Apache restart handler
├── files/favicon.ico          # Default favicon
├── molecule/
│   ├── default/
│   │   ├── molecule.yml      # Molecule configuration
│   │   ├── converge.yml      # Test playbook (multiple role invocations)
│   │   └── verify.yml        # Verification tasks
│   └── resources/
│       ├── prepare.yml       # Prepares test environment (bootstrap role)
│       └── debug.yml         # Debug output helper
└── .github/workflows/
    └── ci.yml                # GitHub Actions CI pipeline
```

## CI/CD Pipeline

**`.github/workflows/ci.yml`:**
- **Lint Job:** yamllint only (runs first)
- **Molecule Job:** Matrix of 7 distributions
- **Manual Dispatch:** Allows selecting specific distro + ansible version with debug mode
- **Artifacts:** Uploads `/var/tmp/vars.yml` and `/var/tmp/environment.yml` from tests
- **Tmate Debug:** On manual runs, opens interactive shell if tests fail

**`.github/workflows/release-to-galaxy.yml`:**
- Triggered on tag push (tags must NOT start with `v`)
- Imports role to Ansible Galaxy

## Important Notes

### Debian Configuration Philosophy

Debian's Apache2 differs significantly from upstream. It uses a modular configuration system:
- Separate directories for `mods-available`, `mods-enabled`, `conf-available`, `conf-enabled`, `sites-available`, `sites-enabled`
- Symlinks used to enable/disable features
- Main config: `/etc/apache2/apache2.conf` includes all enabled configs

See README.adoc section "Facts/Variables defined by this role" for full directory tree examples.

### CookieCutter Synchronization

This project is templated from https://github.com/JonasPammer/cookiecutter-ansible-role and kept in sync using `cruft`:

```bash
cruft update
```

Many files (CI workflows, pre-commit config, etc.) originate from the template. Consider if changes should be made to the template instead.

### Conventional Commits

Core contributors must follow conventional commits. PRs are squash-merged, so casual contributors don't need to worry about commit format.

### Pre-commit Hooks

The project uses extensive pre-commit hooks (see `.pre-commit-config.yaml`):
- commitlint (conventional commits)
- Various file checks (JSON, YAML, TOML, XML)
- detect-secrets
- prettier (markdown, JSON, YAML formatting)
- yamllint
- Python tools: pyupgrade, black, reorder-python-imports, flake8, mypy

Hooks run automatically on pre-commit.ci for PRs.

## Testing Behavior

**Tox Environments:**
- `pre-commit` - runs pre-commit hooks
- `py3-ansible-{6,7,8,9}` - runs molecule with specific Ansible version

**Molecule Behavior:**
- Always runs `ansible-lint` during lint phase
- Destroys container after test unless `MOLECULE_DESTROY=never`
- Uses privileged Docker containers with systemd support
- Installs role dependencies from `requirements.yml` (jonaspammer.bootstrap)

**Environment Variables:**
- `MOLECULE_DISTRO` - Docker image suffix (e.g., `debian12`, `ubuntu2204`)
- `MOLECULE_DESTROY` - Set to `never` to keep container for debugging
- `TOX_ENVNAME` - Tox environment name (used in container naming)
