# RHEL SBOM & CVE Scan (Syft + Grype)

Ansible playbook that installs [Syft](https://github.com/anchore/syft) and [Grype](https://github.com/anchore/grype) on RHEL hosts, generates SBOM and CVE artifacts, and renders an HTML report.

## What it does

1. Installs `curl`/`tar` and the Syft + Grype CLIs via official Anchore install scripts
2. Optionally updates the Grype vulnerability database
3. Scans a configurable filesystem path (default `/`) with Syft
4. Scans the SBOM with Grype for known CVEs
5. Writes JSON/CycloneDX artifacts on each host
6. Renders `report.html` on each host
7. Fetches all artifacts to the control node under `playbooks/artifacts/<hostname>/`

## Layout

```
aap_sbom/
├── ansible.cfg
├── group_vars/all.yml
├── inventory/hosts.yml
├── playbooks/sbom_scan.yml
└── roles/
    ├── syft_grype/      # Install tools
    ├── sbom_scan/       # Run scans, write JSON artifacts
    └── html_report/     # Render HTML from Grype output
```

## Requirements

- Ansible 2.14+ on the control node
- Passwordless `sudo` on target RHEL hosts
- Outbound HTTPS to `get.anchore.io` (tool install) and Grype DB endpoints
- Scanning `/` requires root and can take several minutes on busy systems

## Quick start

1. Edit [`inventory/hosts.yml`](inventory/hosts.yml) with your RHEL host(s).
2. Adjust variables in [`group_vars/all.yml`](group_vars/all.yml) if needed.
3. Run from the project root:

```bash
cd aap_sbom
ansible-playbook playbooks/sbom_scan.yml
```

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `sbom_scan_path` | `/` | Filesystem path Syft scans (`dir:` target) |
| `sbom_remote_output_dir` | `/var/lib/sbom-scan` | Base directory for artifacts on each host |
| `sbom_local_artifact_root` | `playbooks/artifacts` | Control-node destination for fetched files |
| `grype_db_update` | `true` | Run `grype db update` before scanning |
| `syft_version` | `""` | Pin Syft version (empty = latest) |
| `grype_version` | `""` | Pin Grype version (empty = latest) |

Example: scan `/opt/myapp` instead of the full OS:

```yaml
# group_vars/all.yml
sbom_scan_path: /opt/myapp
```

## Output artifacts

On each managed host (`/var/lib/sbom-scan/<hostname>/`):

| File | Description |
|------|-------------|
| `sbom.syft.json` | Syft-native SBOM (used as Grype input) |
| `sbom.cyclonedx.json` | CycloneDX SBOM for compliance tooling |
| `grype-report.json` | CVE matches from Grype |
| `scan-metadata.json` | Host and scan summary metadata |
| `report.html` | Human-readable HTML report |

After the playbook completes, copies are also under:

```
playbooks/artifacts/<hostname>/
```

Open `report.html` in a browser to review severity counts and CVE details.

## AAP / AWX usage

Import this project as an SCM inventory/playbook source. Set extra variables in a Job Template to override `sbom_scan_path` or artifact paths. Ensure execution environments include Ansible core and can reach Anchore endpoints.

## Security notes

- Install scripts are fetched over HTTPS from Anchore; pin versions in production with `syft_version` / `grype_version`.
- Artifact directories are created with mode `0750` and owned by root.
- Full root filesystem scans may surface sensitive package paths; restrict artifact access accordingly.
