# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **ProxmoxVE Helper-Scripts** repository - a community-driven collection of automation scripts for Proxmox Virtual Environment. Originally created by tteck, now maintained by the community. The project provides one-command installations for 400+ applications via LXC containers and VMs.

Website: https://helper-scripts.com

## Repository Structure

### Core Script Directories

- **ct/** - Container creation scripts (400+ bash scripts)
  - Run on Proxmox host, create and configure LXC containers
  - Example: `ct/docker.sh`, `ct/homeassistant.sh`
  - Each script sources `misc/build.func` for orchestration

- **install/** - Installation scripts (400+ bash scripts)
  - Run inside containers after creation
  - Install and configure the actual applications
  - Example: `install/docker-install.sh`
  - Each script sources from stdin via `$FUNCTIONS_FILE_PATH`

- **vm/** - Virtual machine creation scripts
  - Similar to ct/ but creates full VMs instead of containers
  - Examples: `debian-vm.sh`, `haos-vm.sh`, `ubuntu2404-vm.sh`

- **misc/** - Core library functions
  - `build.func` - Main orchestrator for container creation, storage, variables
  - `install.func` - Container setup (OS updates, package management)
  - `tools.func` - Tool installation helpers (GitHub releases, repos)
  - `core.func` - UI/messaging (colors, spinners, validation)
  - `error_handler.func` - Error handling and signal management

- **tools/** - Utility scripts and helpers
  - `addon/` - Post-install addon scripts
  - `headers/` - Script header files
  - `pve/` - Proxmox-specific utilities

### Frontend & API

- **frontend/** - Next.js 15.5 website (helper-scripts.com)
  - Bun package manager
  - TypeScript + React 19
  - Tailwind CSS, shadcn/ui components
  - Commands:
    - `cd frontend && bun install` - Install dependencies
    - `bun dev` - Start development server with Turbopack
    - `bun build` - Production build
    - `bun lint` - Run ESLint with fixes
    - `bun typecheck` - TypeScript validation

- **api/** - Go 1.24 backend API
  - MongoDB integration for data storage
  - Gorilla Mux router, CORS enabled
  - Commands:
    - `cd api && go run main.go` - Run API server
    - `go build` - Build binary

### Documentation

- **docs/** - Comprehensive documentation
  - `contribution/` - Contributing guidelines, templates, fork setup
  - `ct/DETAILED_GUIDE.md` - Complete reference for container scripts
  - `install/DETAILED_GUIDE.md` - Complete reference for install scripts
  - `TECHNICAL_REFERENCE.md` - Architecture deep-dive
  - `DEV_MODE.md` - Debugging modes and development workflow
  - `EXIT_CODES.md` - Exit code reference

## Script Architecture

### Container Creation Flow

```
START: bash ct/docker.sh
  ↓
1. Set APP variable and defaults (var_cpu, var_ram, var_disk, var_os, var_version)
  ↓
2. Source build.func from GitHub (raw.githubusercontent.com)
  ↓
3. Run: header_info, variables, color, catch_errors
  ↓
4. install_script() → Show mode menu:
   - Mode 0: Default settings
   - Mode 1: Advanced (19-step wizard)
   - Mode 2: User defaults
   - Mode 3: App defaults
   - Mode 4: Settings menu
  ↓
5. build_container() → Create LXC, run install script inside
  ↓
6. update_script() → Define update logic for the app
  ↓
7. Display success message with access URL
```

### Script Templates

**Container Script (`ct/app.sh`):**
```bash
#!/usr/bin/env bash
source <(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/misc/build.func)
# Copyright and license headers...

APP="AppName"
var_cpu="${var_cpu:-2}"
var_ram="${var_ram:-2048}"
var_disk="${var_disk:-8}"
var_os="${var_os:-debian}"
var_version="${var_version:-12}"

header_info "$APP"
variables
color
catch_errors

function update_script() {
  # Update logic here
}

start
build_container
description

msg_ok "Completed Successfully!\n"
echo -e "Access URL: https://${IP}:port"
```

**Install Script (`install/app-install.sh`):**
```bash
#!/usr/bin/env bash
# Copyright and license headers...

source /dev/stdin <<<"$FUNCTIONS_FILE_PATH"
color
verb_ip6
catch_errors
setting_up_container
network_check
update_os

# Installation logic here
msg_info "Installing AppName"
# ... installation commands ...
msg_ok "Installed AppName"

motd_ssh
customize

msg_info "Cleaning up"
# ... cleanup ...
msg_ok "Cleaned"
```

## Development Workflow

### Testing Scripts Locally

**Important:** When developing, you must modify URLs to point to your fork:

1. **Fork the repository** to your GitHub account

2. **Clone your fork:**
   ```bash
   git clone https://github.com/YOUR_USERNAME/ProxmoxVE.git
   cd ProxmoxVE
   git checkout -b feature/my-new-app
   ```

3. **Modify paths during development:**
   - In `ct/myapp.sh`: Change source URL to your fork/branch
     ```bash
     source <(curl -fsSL https://raw.githubusercontent.com/YOUR_USERNAME/ProxmoxVE/refs/heads/BRANCH/misc/build.func)
     ```
   - In `misc/build.func` and `misc/install.func`: Update internal URLs to your fork

4. **Test on Proxmox:**
   ```bash
   bash ct/myapp.sh
   ```

5. **Before submitting PR:** Revert all URLs back to:
   ```
   https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/
   ```

### Development Modes

Use `dev_mode` environment variable for debugging:

```bash
# Enable early MOTD/SSH access
export dev_mode="motd"
bash ct/myapp.sh

# Keep container on failure, enable tracing
export dev_mode="keep,trace"
bash ct/myapp.sh

# Verbose output with pause points
export var_verbose="yes"
export dev_mode="pause,logs"
bash ct/myapp.sh
```

Available modes: `motd`, `keep`, `trace`, `pause`, `logs`, `breakpoint`

### Creating New Scripts

1. **Copy templates:**
   ```bash
   cp docs/contribution/templates_ct/AppName.sh ct/myapp.sh
   cp docs/contribution/templates_install/AppName-install.sh install/myapp-install.sh
   ```

2. **Follow coding standards:**
   - Read `docs/contribution/CONTRIBUTING.md`
   - Use proper shebang: `#!/usr/bin/env bash`
   - Include copyright header
   - Use `msg_info`, `msg_ok`, `msg_error` for output
   - Implement proper error handling
   - Define `update_script()` function in ct/ scripts

3. **Test thoroughly:**
   - Test with default mode
   - Test with advanced mode
   - Test update functionality
   - Verify cleanup on failure

4. **Submit PR:**
   - Ensure URLs point to main repo
   - Include clear commit message
   - Reference any related issues

## Common Commands

### Testing & Validation

```bash
# Test a container script (on Proxmox host)
bash ct/docker.sh

# Test with verbose output
var_verbose="yes" bash ct/docker.sh

# Test with development mode
dev_mode="keep,trace" bash ct/docker.sh

# Run install script directly inside container
pct enter CTID
bash /tmp/install-script.sh
```

### Frontend Development

```bash
cd frontend
bun install                 # Install dependencies
bun dev                     # Development server (localhost:3000)
bun build                   # Production build
bun lint                    # Lint and fix
bun typecheck              # Type checking
```

### API Development

```bash
cd api
cp .env.example .env        # Configure environment
go run main.go              # Run API server
go build                    # Build binary
```

### Repository Management

```bash
# Sync fork with upstream
git fetch upstream
git rebase upstream/main
git push -f origin main

# Run formatting checks (if available locally)
shellcheck ct/*.sh
shfmt -w ct/*.sh
```

## Important Conventions

### Variable System

Scripts use a hierarchical defaults system:

1. **Built-in defaults** (in ct/ script)
2. **Global user defaults** (`/usr/local/community-scripts/default.vars`)
3. **App-specific defaults** (`/usr/local/community-scripts/defaults/app.vars`)
4. **Environment variables** (highest precedence)

Common variables:
- `var_cpu` - CPU cores
- `var_ram` - RAM in MB
- `var_disk` - Disk size in GB
- `var_os` - OS type (debian, ubuntu, alpine)
- `var_version` - OS version
- `var_unprivileged` - Unprivileged container (0/1)
- `var_hostname` - Container hostname
- `var_tags` - Proxmox tags

### Naming Conventions

- Container scripts: `ct/appname.sh` (lowercase, no spaces)
- Install scripts: `install/appname-install.sh`
- Function names: `snake_case`
- Variables: `var_*` for user-configurable, `UPPERCASE` for constants

### Update Functions

Every ct/ script must implement `update_script()` function that:
1. Updates base system packages
2. Updates the application
3. Handles service restarts
4. Displays success message
5. Exits with `exit` after completion

## CI/CD & Automation

The repository has extensive GitHub Actions workflows:

- **script_format.yml** - Enforces bash script formatting
- **script-test.yml** - Tests scripts for common issues
- **frontend-cicd.yml** - Builds and tests frontend
- **changelog-pr.yml** - Automatically updates CHANGELOG.md
- **crawl-versions.yaml** - Checks for new application versions
- **autolabeler.yml** - Auto-labels PRs based on changes

When contributing, ensure your changes pass all CI checks.

## Security & Best Practices

- Never hardcode credentials or API keys
- Always validate user input
- Use `$STD` variable to suppress output in non-verbose mode
- Implement proper cleanup in error handlers
- Test scripts in isolated environments first
- Use unprivileged containers when possible (`var_unprivileged=1`)
- Follow principle of least privilege

## Resources

- **Main Website:** https://helper-scripts.com
- **GitHub Repo:** https://github.com/community-scripts/ProxmoxVE
- **Discord:** https://discord.gg/3AnUqsXnmK
- **Contribution Guide:** docs/contribution/README.md
- **Detailed Guides:**
  - Container scripts: docs/ct/DETAILED_GUIDE.md
  - Install scripts: docs/install/DETAILED_GUIDE.md
- **Technical Reference:** docs/TECHNICAL_REFERENCE.md

## Notes for Claude Code

- When modifying scripts, always check if templates in docs/contribution/templates_* have been updated
- Pay attention to the URL paths - they differ between development and production
- The build.func library is central to how everything works - reference it when unclear
- Scripts are designed to run on Proxmox hosts, not generic Linux systems
- The frontend and API are separate from the bash scripts - they power the website
- Always test in dev_mode before suggesting changes that could break containers
