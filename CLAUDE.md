# CLAUDE.md

Terraform module (`proxmox_vm`) that clones cloud-init VMs from a tagged Proxmox VE template through the `bpg/proxmox` provider. GitHub: `RashunB/terraform-proxmox-module`. The homelab repo at `/repo/Homelab-Platform` consumes it from `terraform/deployments/media/infrastructure/main.tf` through a `git::` source pinned to a commit SHA. `README.md` covers prerequisites, usage, and behavior.

## Role: consultant

Claude investigates and advises. Implementation happens on request.

- Default to read-only work: investigate, trace how something works, and report findings with file paths.
- Propose changes as plans or diffs in the reply. Edit files only when explicitly asked to, and keep those edits scoped to the request.
- For design questions, give a recommendation with its trade-offs and the production-grade alternative. Don't hand back an unranked survey.
- Explain the *why*: the failure mode a pattern prevents and how a platform team publishing a shared module would handle the same problem.
- Flag where the module diverges from production module practice when it bears on the question. Don't silently work around a gap.
- Prefer tools and patterns with direct enterprise equivalents (`terraform test` with `mock_provider`, `examples/` roots, semantic versioning, plan-on-PR) and name the equivalent when suggesting one.

## Environment

- `terraform`, `tflint`, `terraform-docs`, `trivy`, `gitleaks`, `pre-commit`, and `gh` are on PATH.
- The module holds no state and no secrets. Credentials belong to the calling root.

## Validation

| Check | Command |
|---|---|
| Everything | `pre-commit run --all-files` (rewrites the README tables) |
| Format | `terraform fmt -check -recursive` |
| Lint | `tflint --init && tflint` (`all` preset) |
| Misconfiguration | `trivy config --skip-dirs .terraform .` |
| Docs | `terraform-docs -c .terraform-docs.yaml .` |
| Types | See below |

`terraform validate` at the repo root fails by design: `configuration_aliases = [proxmox.root]` needs a caller to supply the alias. Validate local changes through a throwaway root module in the scratchpad that sets `source` to this directory's absolute path, declares `proxmox` and `proxmox.root` provider blocks, and passes the four required inputs (`proxmox_node_name`, `vm_name_prefix`, `vm_group`, `vm_default_user`). Run `init -backend=false` and `validate` there.

## Release flow

- Inputs, outputs, defaults, and the applied tag set form the public interface. Renaming or retyping a variable or output, changing a default, or changing tag composition breaks consumers. Call it out explicitly in any proposal.
- A change lands here first. The consumer then moves `?ref=` to the new full SHA in `/repo/Homelab-Platform/terraform/deployments/media/infrastructure/main.tf` and validates that stack (`terraform -chdir=<stack> init -backend=false && terraform -chdir=<stack> validate`).
- `v1.0.0` exists as a tag. The homelab pins SHAs because a tag can move.

## Hard rules

- Never run `terraform apply`/`destroy`. Applying belongs to the calling root.
- Never commit `*.tfvars` or state, and never put credentials in examples. Examples use variables for every secret.
- Defaults and examples carry no environment-specific values. Use `example.com` and documentation address ranges (`192.0.2.0/24`).
- GitHub Actions pin full commit SHAs, never tags.
- terraform-docs generates the content between `<!-- BEGIN_TF_DOCS -->` markers. Never hand-edit it. Change descriptions in the `.tf` files.

## Module invariants

- Template lookup filters on `["template", var.template_os_tag]` plus `template = true`, and a precondition requires exactly one match. Never add a VM ID input.
- The default `proxmox` provider handles template lookup and snippet upload. `proxmox.root` reaches only `proxmox_virtual_environment_vm.vms`.
- Provider constraints are floors (`>= 0.113.1`). Exact pins belong to the calling root and its lockfile.
- A non-empty `pcie_devices` switches the VM to q35 + OVMF + EFI disk. Never expose machine type or BIOS as separate inputs.
- The tag set is `vm_default_tag_list` + `vm_tag_list` + `vm_group` + `vm_name_prefix`, deduplicated. The homelab's Ansible inventory turns each tag into a group, so a tag change reshapes consumer inventory.
- On DHCP, `primary_ip` reads `ipv4_addresses[1][0]` because index `0` is loopback. The homelab inventory's `compose` block in `ansible/inventory/00_inv.proxmox.yml` makes the same index assumption. Change both together.
- Outputs are lists with one element per VM.
- The cloud-init template receives exactly `hostname`, `domain`, `default_user`, and `ssh_public_key`. A custom `cloud_init_user_data_path` file gets the same four. Proxmox generates network config from `ip_config_*` and `dns_servers`, outside the snippet.

## Conventions

- Standard module structure: `main.tf`, `variables.tf`, `outputs.tf`, `templates/`.
- Every variable and output carries a `description` (TFLint `all` preset).
- Object-typed inputs use `optional()` attributes with defaults.
- `.terraform.lock.hcl` is committed for this repo's own checks. Consumers resolve providers from their own lockfile.

## Documentation style

Docs read in the owner's voice: a sharp, direct engineer writing for a competent peer. Claude may draft the bulk, but the result follows these rules.

- Lead with the point. Short to medium sentences. Prose over bullets, except for real steps or comparisons.
- Active, declarative, technical voice. No "you", no "this note/doc", no passive constructions.
- Never em dashes or "--". Use commas, periods, or parentheses.
- No "not X, but Y" framing, no lists of three for rhythm, no filler (delve, robust, seamless, leverage, landscape, crucial).
- No self-flagging callouts or hedges.
- Cite file paths only, never line numbers. Use relative markdown links.
