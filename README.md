# pod-reader-sync

Ansible-managed deployment of [reader-sync](https://github.com/YOUR_USERNAME/reader-sync) for automatic e-reader synchronization.

## Overview

This pod installs and configures reader-sync on macOS. It clones the git repository, generates configuration from templates, and sets up launchd to automatically sync when e-readers are connected.

**This pod runs locally.** It is designed to be deployed and executed by a VM manager, or run manually on the target machine.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  vm-manager (optional)                                      │
│  └── group_vars/all.yml    ← Environment-specific values    │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ passes variables
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  pod-reader-sync (this repo)                                │
│  ├── vars/default.yml      ← Sensible defaults              │
│  ├── templates/*.j2        ← Config structure               │
│  └── install.yml           ← Renders and installs           │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ generates config.json
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  reader-sync (application)                                  │
│  ├── config.json           ← Generated, git-ignored         │
│  └── reader-sync.sh        ← Reads config at runtime        │
└─────────────────────────────────────────────────────────────┘
```

## Supported Platforms

| Platform | Support | Notes |
|----------|---------|-------|
| macOS | ✅ | Full support with launchd |
| Linux | ❌ | Not supported (requires launchd) |

## Prerequisites

- macOS
- Git
- jq (`brew install jq`)
- Calibre (optional, for Kindle AZW3 conversion)
- Ansible 2.10+

## Playbooks

| Playbook | Description |
|----------|-------------|
| `install.yml` | Clone repo, generate config, install launchd agent |
| `upgrade.yml` | Pull latest and reload launchd |
| `uninstall.yml` | Unload launchd and optionally remove repo |
| `status.yml` | Check installation and launchd status |

## Usage

### Standalone (Manual)

```bash
# Install with defaults
ansible-playbook install.yml

# Install with custom values
ansible-playbook install.yml \
  -e "git_repo=git@github.com:myuser/reader-sync.git" \
  -e "prod_api_url=http://192.168.1.100:8000" \
  -e "smb_share_url=smb://mynas/data"

# Check status
ansible-playbook status.yml

# Upgrade
ansible-playbook upgrade.yml

# Uninstall
ansible-playbook uninstall.yml
ansible-playbook uninstall.yml -e "remove_repo=true"  # Remove everything
```

### GitOps (VM Manager)

When deployed via vm-manager, configuration comes from group_vars:

```yaml
# vm-manager/group_vars/all.yml
smb_share_url: "smb://synology/data"
smb_mount_point: "/Volumes/data"
prod_api_url: "http://synology:8000"

# vm-manager/group_vars/macbook.yml
git_repo: "git@github.com:myuser/reader-sync.git"
```

The vm-manager passes these as extra vars when running the pod.

## Configuration Variables

### Deployment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `git_repo` | `git@github.com:...` | Git repository URL |
| `git_branch` | `main` | Branch to checkout |
| `install_path` | `~/repos/reader-sync` | Where to clone repo |
| `launchd_label` | `com.reader-sync` | launchd service name |

### Application Variables

These are templated into `config.json`:

| Variable | Default | Description |
|----------|---------|-------------|
| `prod_api_url` | `http://synology:8000` | Production API URL |
| `dev_api_url` | `http://localhost:8000` | Development API URL |
| `prod_input_dir` | `/Volumes/data/prod/input` | Prod input directory |
| `prod_newspaper_dir` | `/Volumes/data/prod/publications` | Prod output directory |
| `smb_enabled` | `true` | Enable SMB auto-mount |
| `smb_share_url` | `smb://synology/data` | SMB share URL |
| `smb_mount_point` | `/Volumes/data` | Where share mounts |
| `poll_interval` | `2` | Seconds between status checks |
| `poll_timeout` | `900` | Max wait for tasks |
| `auto_eject` | `true` | Eject device after sync |

## First-Time Device Trust

After installation, new e-readers must be trusted manually:

```bash
cd ~/repos/reader-sync
./reader-sync.sh --env=prod
```

This creates a marker file on the device for future recognition.

## Files

```
pod-reader-sync/
├── README.md
├── ansible.cfg
├── pod.yml
├── install.yml
├── upgrade.yml
├── uninstall.yml
├── status.yml
├── vars/
│   ├── default.yml          ← Defaults (override via vm-manager)
│   └── platform/
│       └── darwin.yml       ← macOS-specific paths
└── templates/
    ├── config.json.j2            ← Application config template
    └── com.reader-sync.plist.j2  ← launchd template
```

## Related Projects

- [reader-sync](https://github.com/YOUR_USERNAME/reader-sync) - The application this pod deploys
- [worker-celery](https://github.com/YOUR_USERNAME/worker-celery) - Background worker for PKS
- [pod-worker-celery](https://github.com/YOUR_USERNAME/pod-worker-celery) - Worker deployment pod
