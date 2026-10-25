# Proxmox VE VM Terraform module

Terraform module that clones cloud-init enabled VMs from a tagged Proxmox VE template using the [bpg/proxmox](https://registry.terraform.io/providers/bpg/proxmox/latest) provider. Each VM gets its own rendered cloud-init user-data snippet, a single NIC with DHCP or static IPv4, optional extra disks, and optional PCIe passthrough. VM tags are derived from the inputs so the result can feed a Proxmox-backed Ansible dynamic inventory.

## Prerequisites

The module resolves its source template at plan time by looking for a VM with `template = true` and both the `template` tag and the tag in `template_os_tag` (default `default`). Exactly one VM across the cluster must match, otherwise a precondition fails and lists what it found. The template must live on `proxmox_node_name`, carry its boot disk on `virtio0`, and have cloud-init and the QEMU guest agent available. Clones are linked (`full = false`), so the template's storage must support linked clones and the template cannot be removed while clones exist.

`datastore_files` must have the `snippets` content type enabled. The provider uploads snippets over SSH, so the default `proxmox` provider needs a working `ssh` block.

The module declares two provider configurations. Template lookup and snippet upload use the default `proxmox` provider. The VM resource uses the `proxmox.root` alias, which lets VM creation run under separate, higher-privileged credentials. Both keys can point at the same configuration if one credential set covers everything.

## Usage

```hcl
provider "proxmox" {
  endpoint  = "https://pve.example.com:8006/"
  api_token = var.proxmox_api_token

  ssh {
    agent    = true
    username = "root"
  }
}

provider "proxmox" {
  alias    = "root"
  endpoint = "https://pve.example.com:8006/"
  username = "root@pam"
  password = var.proxmox_root_password
}

module "web" {
  source = "git::https://github.com/RashunB/terraform-proxmox-module.git?ref=v1.0.0"

  providers = {
    proxmox      = proxmox
    proxmox.root = proxmox.root
  }

  proxmox_node_name = "pve"
  vm_name_prefix    = "web"
  vm_group          = "webservers"
  vm_default_user   = "ansible"
  ssh_public_key    = var.ssh_public_key

  vm_count = 2
  cpu      = 4
  memory   = 4096
}
```

This creates `web-1` and `web-2` on node `pve`, tagged `terraform`, `webservers` and `web`, on `vmbr0` with DHCP.

### Static IP, extra disk and GPU passthrough

```hcl
module "gpu_worker" {
  source = "git::https://github.com/RashunB/terraform-proxmox-module.git?ref=v1.0.0"

  providers = {
    proxmox      = proxmox
    proxmox.root = proxmox.root
  }

  proxmox_node_name = "pve"
  vm_name_prefix    = "gpu"
  vm_group          = "compute"
  vm_default_user   = "ansible"
  ssh_public_key    = var.ssh_public_key
  template_os_tag   = "debian12"

  ip_config_address = "192.168.0.50/24"
  ip_config_gateway = "192.168.0.1"

  additional_disks = {
    virtio1 = {
      datastore_id = "vmdata"
      size         = 200
    }
  }

  pcie_devices = {
    gpu = {
      device  = "hostpci0"
      mapping = "gpu"
    }
  }
}
```

## Behavior

**Naming.** VMs are named `<vm_name_prefix>-<n>`, where `n` starts at `vm_count_offset` (default `1`). Use the offset to split one name series across several module calls with different sizing: a call with `vm_count = 2` produces `-1` and `-2`, and a second call with `vm_count_offset = 3` continues at `-3`. The cloud-init snippet is stored as `<name>-cloud-config.yaml` in `datastore_files`, so names must be unique per datastore.

**Tags.** The applied tag set is `vm_default_tag_list` + `vm_tag_list` + `vm_group` + `vm_name_prefix`, deduplicated. `terraform` is included by default to mark the VM as Terraform-managed.

**Cloud-init.** The bundled [`templates/cloud-init.yml.tpl`](templates/cloud-init.yml.tpl) sets the hostname and FQDN from `personal_domain`, creates `vm_default_user` with passwordless sudo and `ssh_public_key` as its only credential, disables SSH password auth, upgrades packages (rebooting if required) and enables `qemu-guest-agent`. `ssh_public_key` defaults to empty, so leaving it unset produces a VM you cannot log into. To supply your own user-data, set `cloud_init_user_data_path`. The file is rendered with `templatefile()` and receives `hostname`, `domain`, `default_user` and `ssh_public_key`. Network configuration is not part of the snippet; Proxmox generates it from `ip_config_*` and `dns_servers`.

**Networking.** One NIC on `network_device_bridge` using `network_device_model`. `ip_config_address` is `dhcp` or an address in CIDR notation, and `ip_config_gateway` is applied only for a static address. The static address is shared by every VM in the call, so use `vm_count = 1` with static addressing.

**Disks.** `additional_disks` is keyed by interface name (`virtio1`, `scsi1`, ...). Do not use `virtio0`, which is the cloned boot disk. Setting `path_in_datastore` attaches an existing volume and ignores `size`, `iothread` and `discard`.

**PCIe passthrough.** A non-empty `pcie_devices` switches the VM to `q35` with OVMF firmware and adds a 4M EFI disk on `datastore_infra`, so the template's boot disk must be UEFI-bootable. `device` is the slot name (`hostpci0` to `hostpci15`) and `mapping` is a cluster resource mapping name. Raw PCI IDs are not exposed. Without passthrough the VM uses `pc` and SeaBIOS.

**Fixed settings.** CPU type `host` (live migration only between identical CPUs), OS type `l26`, boot order `virtio0`, guest agent enabled, and `stop_on_destroy = true` so destroy does not wait on a guest shutdown.

**Outputs.** `vm_id` is the list of VMIDs. With DHCP, `primary_ip` is the first IPv4 address the guest agent reports on the first interface after loopback, so apply blocks until the agent is up. With a static address it returns the configured value.

## Development

[pre-commit](https://pre-commit.com) runs `terraform fmt`, TFLint, terraform-docs, Trivy and gitleaks:

```sh
pre-commit install
pre-commit run --all-files
```

Everything between the `BEGIN_TF_DOCS` and `END_TF_DOCS` markers below is generated by [terraform-docs](https://terraform-docs.io) from `.terraform-docs.yaml`. Edit variable and output descriptions in the `.tf` files rather than the table.
<!-- BEGIN_TF_DOCS -->
## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 1.15 |
| <a name="requirement_proxmox"></a> [proxmox](#requirement\_proxmox) | >= 0.113.1 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_proxmox"></a> [proxmox](#provider\_proxmox) | 0.113.1 |
| <a name="provider_proxmox.root"></a> [proxmox.root](#provider\_proxmox.root) | 0.113.1 |

## Modules

No modules.

## Resources

| Name | Type |
|------|------|
| [proxmox_virtual_environment_file.cloud_config](https://registry.terraform.io/providers/bpg/proxmox/latest/docs/resources/virtual_environment_file) | resource |
| [proxmox_virtual_environment_vm.vms](https://registry.terraform.io/providers/bpg/proxmox/latest/docs/resources/virtual_environment_vm) | resource |
| [proxmox_virtual_environment_vms.templates](https://registry.terraform.io/providers/bpg/proxmox/latest/docs/data-sources/virtual_environment_vms) | data source |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_additional_disks"></a> [additional\_disks](#input\_additional\_disks) | Map of disks keyed by interface name. | <pre>map(object({<br/>    datastore_id      = string<br/>    size              = optional(number, 1)<br/>    iothread          = optional(bool, true)<br/>    cache             = optional(string, "none")<br/>    discard           = optional(string, "ignore")<br/>    file_format       = optional(string, "raw")<br/>    path_in_datastore = optional(string, null)<br/>    backup            = optional(bool, false)<br/>    replicate         = optional(bool, false)<br/>    serial            = optional(string, null)<br/>  }))</pre> | `{}` | no |
| <a name="input_cloud_init_user_data_path"></a> [cloud\_init\_user\_data\_path](#input\_cloud\_init\_user\_data\_path) | Path to cloud-init .tpl file. If null, the module default is used. | `string` | `null` | no |
| <a name="input_cpu"></a> [cpu](#input\_cpu) | Number of CPUs | `number` | `2` | no |
| <a name="input_datastore_files"></a> [datastore\_files](#input\_datastore\_files) | Proxmox datastore to store file components provisioned by Terraform | `string` | `"vmfiles"` | no |
| <a name="input_datastore_infra"></a> [datastore\_infra](#input\_datastore\_infra) | Proxmox datastore to store infrastructure components provisioned by Terraform | `string` | `"vmdata"` | no |
| <a name="input_dns_servers"></a> [dns\_servers](#input\_dns\_servers) | List of DNS servers written to VMs cloud-init network config | `list(string)` | <pre>[<br/>  "1.1.1.1",<br/>  "8.8.8.8"<br/>]</pre> | no |
| <a name="input_ip_config_address"></a> [ip\_config\_address](#input\_ip\_config\_address) | Static IP address for the vm | `string` | `"dhcp"` | no |
| <a name="input_ip_config_gateway"></a> [ip\_config\_gateway](#input\_ip\_config\_gateway) | Gateway to use when a static IP address is specified | `string` | `"192.168.0.1"` | no |
| <a name="input_memory"></a> [memory](#input\_memory) | Total amount of memory in MBs | `number` | `2048` | no |
| <a name="input_network_device_bridge"></a> [network\_device\_bridge](#input\_network\_device\_bridge) | Network device bridge | `string` | `"vmbr0"` | no |
| <a name="input_network_device_model"></a> [network\_device\_model](#input\_network\_device\_model) | Network device model | `string` | `"virtio"` | no |
| <a name="input_pcie_devices"></a> [pcie\_devices](#input\_pcie\_devices) | Map object to mount a PCIE device. | <pre>map(object({<br/>    device  = optional(string, null)<br/>    mapping = optional(string, null)<br/>    pcie    = optional(bool, true)<br/>    rombar  = optional(bool, true)<br/>  }))</pre> | `{}` | no |
| <a name="input_personal_domain"></a> [personal\_domain](#input\_personal\_domain) | Personal domain to use for VM hostname | `string` | `"home.lab"` | no |
| <a name="input_proxmox_node_name"></a> [proxmox\_node\_name](#input\_proxmox\_node\_name) | Proxmox host node name | `string` | n/a | yes |
| <a name="input_ssh_public_key"></a> [ssh\_public\_key](#input\_ssh\_public\_key) | Public SSH key for the VM default user | `string` | `""` | no |
| <a name="input_template_os_tag"></a> [template\_os\_tag](#input\_template\_os\_tag) | os tag of the template to clone. Defaults to default (ubuntu24) from \_base. | `string` | `"default"` | no |
| <a name="input_vm_count"></a> [vm\_count](#input\_vm\_count) | Count for VMs to create. Defaults to one | `number` | `1` | no |
| <a name="input_vm_count_offset"></a> [vm\_count\_offset](#input\_vm\_count\_offset) | VM count offset. Used for deploying vms with the same name, but different configurations. This offsets the count to keep it consecutive across two deployments | `number` | `1` | no |
| <a name="input_vm_default_tag_list"></a> [vm\_default\_tag\_list](#input\_vm\_default\_tag\_list) | Default set of tags used for organizing and grouping for ansible inventory. Terraform tag is used as default tag to signal managed\_nodes | `list(string)` | <pre>[<br/>  "terraform"<br/>]</pre> | no |
| <a name="input_vm_default_user"></a> [vm\_default\_user](#input\_vm\_default\_user) | Default user for VM setup and configuration | `string` | n/a | yes |
| <a name="input_vm_group"></a> [vm\_group](#input\_vm\_group) | Group to initially put the VM in | `string` | n/a | yes |
| <a name="input_vm_name_prefix"></a> [vm\_name\_prefix](#input\_vm\_name\_prefix) | Prefix for the VM name | `string` | n/a | yes |
| <a name="input_vm_tag_list"></a> [vm\_tag\_list](#input\_vm\_tag\_list) | A set of tags used for organizing and grouping for ansible inventory. Terraform tag is used as default tag to signal managed\_nodes | `list(string)` | `[]` | no |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_primary_ip"></a> [primary\_ip](#output\_primary\_ip) | First non-loopback IPv4 address(es) of VM(s) |
| <a name="output_vm_id"></a> [vm\_id](#output\_vm\_id) | Proxmox VMID(s) |
<!-- END_TF_DOCS -->
