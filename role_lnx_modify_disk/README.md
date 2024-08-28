# role_lnx_modify_disk

This role will do the following steps:

* Takes job template survey questions to determine if the root disk will grow to new size.
* It will determine if there are other volume groups that are sharing the same disk as the primary volume group which is usually `rootvg`.
  * true, a second vg does share disk with the primary vg, then this role will move the secondary vg away from the primary vg onto a new disk with appropriate size to handle it.
  * false, nothing will happen except for determining that primary volume group needs to expand for operations like server patching, leaping to a new version of the OS, and convert it to RHEL.
* After relocating the secondary to a new disk, it will clean off primary vg logical volumes from other disk and remove those disk if empty.
* The new secondary disk will be like `/dev/sdc` when it should be `/dev/sdb`, upon a reboot at any given time frame, Linux will see it's `/dev/sdb` and work normally, so `/dev/sdc` will seem like it has disappeared.
* Lastly, it will increase the logical volumes and filesystems to new sizes if variables are set to more than zero.

| :warning: WARNING |
|:--|
| Snapshots for the VM Guest will be removed. |

## Requirements

* The python module `PyVmomi` and `aiohttp` is required which is currently part of `Execution Environment` called `centerpoint_ee`.
* Set `gather_facts: false` in calling playbook if called by itself, because it will gather facts after the OS is determined.
* Role `role_vmware_guest_advanced_settings` is required so if disk is be removed from the VM guest host, so please see the variables that are required for this to happen. See `roles/requirements.yml` below. This will need to be added to the playbook in the variables box:

```yaml
role_vmware_guest_advanced_settings__custom_settings:
  - key: disk.EnableUUID
    value: true
```

## Role Variables

Setting the variables below will determine the outcome of what happens on the target VMware virtual guest.

#### Variable provided by Job Template Survey Questions or by calling roles:

| Job Template Variables | Comments |
|:-|:-|
| **role_lnx_modify_disk__debug_mode**<br><span style="color:purple">MultipleChoice (single select)</span><br><span style="color:blue">Boolean</span> | Turn on or off debug output.<br><br>_Choices:_<br>- `true` <br>- `false`<span style="color:blue">&larr;(default)</span> |
| **role_lnx_modify_disk__secondary_disk_grow_percentage**<br><span style="color:purple">MultipleChoice (single select)</span><br><span style="color:red">Required</span> / <span style="color:blue">String</span> | The disk created to hold the second volume group will be calculated for size from information about the volume and an addtional amount will be added to it when creating the disk for future growth.<br><br>_Choices:_<br>- `1.00` meaning no growth<br>- `1.10` meaning `10%` additional growth<br>- `1.20` meaning `20%` additional growth <br>- `1.30` meaning `30%` additional growth<span style="color:blue">&larr;(default)</span><br>- `1.40` meaning `40%` additional growth<br>- `1.50` meaning `50%` additional growth |
| **role_lnx_modify_disk__root_disk_size_gb_exact_size**<br><span style="color:purple">Integer</span><br><span style="color:red">Required</span> / <span style="color:blue">Integer</span> | Put in the number that is greater than the current size of the hard disk on VM Guest in Gigabytes. <br><br>For example, if the disk is 20GB and to increase it by 10GB, put in `30`.<br><br>_Default:_ `50` |
| **role_lnx_modify_disk__grow_gb_root_filesystem**<br><span style="color:purple">Integer</span><br><span style="color:red">Required</span> / <span style="color:blue">Integer</span> | Put in the number to grow the root filesystem `/` to increase it size by gigabytes.<br><br>_Default:_ `2` |
| **role_lnx_modify_disk__grow_gb_usr_filesystem**<br><span style="color:purple">Integer</span><br><span style="color:red">Required</span> / <span style="color:blue">Integer</span> | Put in the number to grow the root filesystem `/usr` to increase it size by gigabytes.<br><br>_Default:_ `2` |
| **role_lnx_modify_disk__grow_gb_var_filesystem**<br><span style="color:purple">Integer</span><br><span style="color:red">Required</span> / <span style="color:blue">Integer</span> | Put in the number to grow the root filesystem `/var` to increase it size by gigabytes.<br><br>_Default:_ `2` |
| **role_lnx_modify_disk__grow_gb_opt_filesystem**<br><span style="color:purple">Integer</span><br><span style="color:red">Required</span> / <span style="color:blue">Integer</span> | Put in the number to grow the root filesystem `/opt` to increase it size by gigabytes.<br><br>_Default:_ `0` |
| **role_lnx_modify_disk__grow_gb_home_filesystem**<br><span style="color:purple">Integer</span><br><span style="color:red">Required</span> / <span style="color:blue">Integer</span> | Put in the number to grow the root filesystem `/home` to increase it size by gigabytes.<br><br>_Default:_ `0` |
| **role_lnx_modify_disk__grow_gb_tmp_filesystem**<br><span style="color:purple">Integer</span><br><span style="color:red">Required</span> / <span style="color:blue">Integer</span> | Put in the number to grow the root filesystem `/tmp` to increase it size by gigabytes.<br><br>_Default:_ `0` |
| **role_lnx_modify_disk__grow_gb_tmp_filesystem**<br><span style="color:purple">Integer</span><br><span style="color:red">Required</span> / <span style="color:blue">Integer</span> | Put in the number to grow the root filesystem `/tmp` to increase it size by gigabytes.<br><br>_Default:_ `0` |

#### Variables provided by Job Template, via credentials or inventory:

  | Variable | Type | Required | Defaults | Comments |
  |-|:-:|:-:|-|-|
  | `vcenter_hostname`     | String  | Yes | None | vSphere or ESX hostname or IP Address. |
  | `vcenter_username`     | String  | Yes | None | Username for vSphere or ESX Hypervisor. |
  | `vcenter_password`     | String  | Yes | None | Password for vSphere or ESX Hypervisor. |
  | `vcenter_certificates` | Boolean | Yes | `false` | Use SSL or disable. |

| :memo: NOTE |
|:--|
| Set the Job Template Limit to set which host is being modified by this playbook from the inventory. |

| :memo: NOTE |
|:--|
| Set `Execution Environment` to `centerpoint_ee` for the Job Tempate |

| :memo: NOTE |
|:--|
| Set the project source to `Execution Environment` to `minimal`. |

| :memo: NOTE |
|:--|
| Below, these values are provided AAP credentials using custom credential type |

#### Use the credential type to create a new credential called `Custom VMware vCenter Credential`


| Inputs | Description |
|:-|:-|
| **Name** |`Custom VMware vCenter Credential` |
| **Description** | `VMware Credential without vCenter hostname` |
| **Fields and Injectors** | See the tables below |

```yaml
fields:
  - id: username
    type: string
    label: vCenter username
  - id: password
    type: string
    label: vCenter password
    secret: true
  - id: verify_ssl
    type: boolean
    label: Verify SSL
required:
  - username
  - password
```

  **Injectors**:

```yaml
env:
  PKR_VAR_vsphere_user: '{{ username }}'
  PKR_VAR_vsphere_password: '{{ password }}'
extra_vars:
  vcenter_password: '{{ password }}'
  vcenter_username: '{{ username }}'
  vcenter_certificates: '{{ verify_ssl }}'
```

## Dependencies
---

* On the host or execution environment that is running the playbook, the python library called `PyVmomi` is required. The role will install this library if it's missing.

* Ansible Collections from `collections/requirements.yml`:

```yaml
---
collections:
  - name: centerpoint.general
  - name: community.general
  - name: community.vmware

...

```

* Calling Role from `roles/requirements.yml`:

```yaml
---
# from a repository
- src: git@ssh.dev.azure.com:v3/centerpoint-automation/CNP.Automation/role_lnx_modify_disk
  name: role_lnx_modify_disk
  scm: git

- src: https://centerpoint-automation@dev.azure.com/centerpoint-automation/CNP.Automation/_git/role_vmware_guest_find
  name: role_vmware_guest_find
  scm: git

- src: https://centerpoint-automation@dev.azure.com/centerpoint-automation/CNP.Automation/_git/role_vmware_guest_advanced_settings
  name: role_vmware_guest_advanced_settings
  scm: git

...

```

## Example Playbook
---

  ```yaml
  ---
  # site.yml for expand_disk

  - name: Project expand_disk
    hosts: all
    gather_facts: true
    vars:
      role_lnx_modify_disk__debug_mode: false
      role_lnx_modify_disk__secondary_disk_grow_percentage: 1.10
      role_lnx_modify_disk__root_disk_size_gb_exact_size: 80
      role_vmware_guest_advanced_settings__custom_settings:
        - key: disk.EnableUUID
          value: true

    tasks:

      - name: Grow the disk and volume group
        ansible.builtin.include_role:
          name: role_lnx_modify_disk

  ...

  ```

## License
---

Private

## Author Information
---

Scott Parker - Principal Consultant Red Hat
