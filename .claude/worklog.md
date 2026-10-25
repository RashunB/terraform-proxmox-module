# terraform-proxmox-module worklog

Goal: a module that validates and tests itself in CI, documents only what works, and ships as a tagged release the homelab consumes by SHA. Work items in order. Each item names its done condition. Record the commit SHA next to each item when it lands.

This log is the module half of the homelab's prod prep. The consumer side is item 9 in `Homelab-Platform/.claude/worklog.md`.

Implementation of these items is authorized. Stop for review before item 8.

## 1. Fix the static-IP output path

`outputs.tf` calls `toset(var.ip_config_address)` on a string, which fails at plan ("cannot convert string to set"). Replace it with a list that strips the prefix length: `[split("/", var.ip_config_address)[0]]`. Both conditional branches then return `list(string)`.

Add `validation` blocks in `variables.tf`:
- `ip_config_address` is `dhcp` or a valid IPv4 CIDR (`can(cidrhost(var.ip_config_address, 0))`).
- A static address requires `vm_count == 1`, since every VM in the call would share it (cross-variable validation, available since Terraform 1.9).

Done when: a scratch root with `ip_config_address = "192.0.2.50/24"` passes `validate` and `plan` against mock providers (see item 3), and `vm_count = 2` with a static address fails validation with a clear message.

## 2. Remove the environment-specific gateway default

`ip_config_gateway` defaults to `192.168.0.1`. Change it to `default = null` and add a validation that requires a gateway whenever `ip_config_address != "dhcp"`.

This changes the public interface. Strict semver makes it a major bump (see item 8). The homelab consumer runs DHCP and is unaffected.

Done when: static without a gateway fails validation, and DHCP with no gateway passes.

## 3. Add `terraform test` with mock providers

Create `tests/` with `.tftest.hcl` files. Declare `mock_provider "proxmox" {}` and `mock_provider "proxmox" { alias = "root" }`. Use `override_data` on `data.proxmox_virtual_environment_vms.templates` to control template matches. Use `override_resource` on `proxmox_virtual_environment_vm.vms` to supply `ipv4_addresses`, because mocked computed lists come back empty and break `primary_ip`.

Cases, all `command = plan` unless an output needs `apply`:
- Defaults: machine `pc`, bios `seabios`, no EFI disk.
- Non-empty `pcie_devices`: machine `q35`, bios `ovmf`, one EFI disk.
- Tags: defaults plus `vm_tag_list`, `vm_group`, and `vm_name_prefix`, deduplicated.
- Naming: `vm_count = 2`, `vm_count_offset = 3` yields `-3` and `-4`.
- Precondition: zero matching templates and two matching templates both fail (`expect_failures`).
- Outputs: DHCP returns `ipv4_addresses[1][0]`, static returns the address without its prefix length.
- Validations from items 1 and 2 fail where expected.

Done when: `terraform init -backend=false && terraform test` passes locally with no Proxmox host. This replaces the scratch-root validation described in `CLAUDE.md`. Update that section.

## 4. Add `examples/`

Two root modules that call the module by relative path (`source = "../.."`) and declare both provider aliases:
- `examples/basic`: DHCP, two VMs.
- `examples/static-gpu`: static address, one extra disk, one PCIe mapping.

Secrets come in as variables. Values use `example.com` and `192.0.2.0/24`.

Done when: `terraform -chdir=examples/<name> init -backend=false && terraform -chdir=examples/<name> validate` passes for both, and the README usage blocks match the example code.

## 5. Replace the CI workflow

`.github/workflows/ci.yml` is a copy of the homelab workflow. It formats and lints `terraform/`, validates three homelab stacks, and points Trivy at a `trivy.yaml` that does not exist here. Rewrite it for this repo:
- `fmt`: `terraform fmt -check -recursive`
- `tflint`: from the repo root
- `test`: `terraform init -backend=false && terraform test`
- `examples`: matrix `init -backend=false` + `validate` over `examples/*`
- `docs`: `terraform-docs -c .terraform-docs.yaml --output-check .`
- `trivy`: `scan-type: config`, `scan-ref: .`, no `trivy-config`
- `gitleaks` with `fetch-depth: 0`, and `actionlint`

Keep the existing SHA-pinned action references. Add `hashicorp/setup-terraform` with a version at or above `required_version`.

Done when: every job passes on a PR, and an intentional fmt error fails `fmt`.

## 6. Repo hygiene

- Finish the `.gitignore` cleanup already in progress (homelab Ansible and `rhce/` entries). Commit `CLAUDE.md`, `.claude/settings.json` and this worklog.
- Commit the `.gitleaksignore` deletion. Its only entry fingerprints a homelab commit (`fe21e68`) that this repo's history does not contain. Confirm with a full-history `gitleaks git` run that reports zero findings.
- The `.pre-commit-config.yaml` change that drops `files: ^terraform/` from `terraform_trivy` lands with this item.

Done when: `git status` is clean apart from intended work, and `pre-commit run --all-files` passes.

## 7. README accuracy and voice

- Static IP: describe the corrected output (address without prefix length) and the new validations.
- Usage examples: link `examples/` and pin `?ref=` to the release from item 8.
- Voice pass per `CLAUDE.md` Documentation style: remove "you" ("a VM you cannot log into", "To supply your own user-data") and passive constructions ("are named", "is stored", "is rendered", "is generated"). Leave the generated tf-docs block alone.

Done when: no "you", no em dashes, and every behavior the README states is covered by a test from item 3.

## 8. Release (stop for review first)

`v1.0.0` points at `8d9465e`, the initial split commit, which predates the `ip_config_*` and network variables. Its README examples fail with "unsupported argument".

- Tag `v2.0.0` on the commit that completes items 1 to 7 (major, because item 2 changes a default).
- Write release notes listing the interface changes.
- Leave `v1.0.0` in place, since moving a published tag breaks anyone who pinned it.

Done when: `v2.0.0` is tagged with release notes. The consumer bump is item 9 in `Homelab-Platform/.claude/worklog.md`.
