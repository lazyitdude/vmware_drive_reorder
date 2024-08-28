
# role_vmware_guest_advanced_settings
---

This role will do the following steps:

* Check that advanced settings is either set or not set on VM guest host.
* if set, it will do nothing.
* if not set, it will set the advanced settings on the VM guest host.
* if set, and the value is a empty string, the setting will be removed. (more discussion below in variable section)
* A typical advanced settings is `disk.enableUUID` is set to `TRUE`.

## Requirements

The following requirements are need to ensure the role runs correctly.

### Python Modules

* The python modules `PyVmomi` and `aiohttp` are required on the target hosts or execution enviroment.

### Dynamic Inventory

The dynamic or static inventory needs to have the following key and value:

```yaml
vmware_guest_name: actual_name_of_vmware_host_in_vsphere
```

Example:

```yaml
vmware_guest_name: ECTXTAPTR106
```

Sample dynamic inventory source file:

```yaml
plugin: community.vmware.vmware_vm_inventory
strict: false
validate_certs: false
with_nested_properties: true
properties:
  - runtime
  - guest
  - config
filters:
  - runtime.powerState == 'poweredOn'
  - guest.guestState == 'running'
  - guest.hostName is not search('_')
  - guest.hostName | split('.') | first | lower == config.name | lower
  - guest.guestId is search('rhel|oracle|sles')
groups:
  vmware_hosts: true
keyed_groups:
  - key: ('RedHat' if guest.guestId is search('rhel'))
    separator: ''
  - key: ('OracleLinux' if guest.guestId is search('oracle'))
    separator: ''
  - key: ('SuSE' if guest.guestId is search('sles'))
    separator: ''
  - key: ('excluded_hosts' if guest.hostName | lower is search('exclude_server_name_prefix|exclude_server_name_prefix|exclude_vm_name|exclude_vm_name'))
    prefix: vmware
  - key: ('appliances' if config.annotation | lower is search('appliance|vmware vrealize'))
    prefix: vmware
hostnames:
  - config.name | lower
compose:
  vmware_guest_name: config.name
  ansible_host: config.name | lower
  vmware_host_ip: guest.ipAddress
  vcenter_hostname: "'vsphere_hostname'"
```

### Collections

* Set the following in `collections/requirements.yml`:

  ```yaml
  ---
  collections:
    - name: centerpoint.general
    - name: community.general
    - name: community.vmware
    - name: vmware.vmware_rest
  ```

### Roles Requirements File

```yaml
---
# from a repository
- src: https://centerpoint-automation@dev.azure.com/centerpoint-automation/CNP.Automation/_git/role_vmware_guest_find
  name: role_vmware_guest_find
  scm: git

- src: https://centerpoint-automation@dev.azure.com/centerpoint-automation/CNP.Automation/_git/role_vmware_guest_advanced_settings
  name: role_vmware_guest_advanced_settings
  scm: git

...

```

### Other roles needed

* In playbook or another role that calls this role, do a `role_vmware_guest_find` to populate variables so it can know where to search and/or add advanced settings.

  ```yaml
  - name: vmware_vm_find | Find VMware folder where the target host resides on VMware
    ansible.builtin.include_role:
      name: role_vmware_guest_find
    vars:
      role_vmware_guest_find__hostname: "{{ hostvars[inventory_hostname]['vmware_guest_name'] }}"
  ```

## Role Variables
---

* Variable provided by `ansible.builtin.include_role` `vars` or in AAP Job Template Extra Variables field and check the box `Prompt on Launch` so it can be changed at run time:

| Job Template Variables | Comments |
|:-|:-|
| **role_vmware_guest_advanced_settings__custom_settings**<br><span style="color:purple">Text</span><br><span style="color:red">Required</span> / <span style="color:blue">String</span> | A table of advanced settings to perform on the host before disk operation where made.<br><br>A dictionary of key and value pairs.<br><br>_Example_:<br>`role_vmware_guest_advanced_settings__custom_settings:`<br>  `  - key: disk.EnableUUID` <br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`value: "TRUE" `<br>  `  - key: tools.remindInstall`<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`value: "TRUE"`<br>  `  - key: some.parameter`<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`value: ""`<span style="color:blue">&larr;(empty string will delete parameter)</span> |

* The last variable example:

```yaml
role_vmware_guest_advanced_settings__custom_settings:
  - key: disk.EnableUUID
    value: 'TRUE'
  - key: tools.remindInstall
    value: 'TRUE'
  - key: some.parameter
    value: ''

```

| :point_up: USAGE NOTE |
|:--|
| The last key and value above will remove the `some.parameter` from the configuration. |



## Dependencies
---

* See requirements above

## Example Playbook
---

  ```yaml
  ---
  # site.yml for role_vmware_guest_advanced_settings

  - name: Project role_vmware_guest_advanced_settings
    hosts: all
    gather_facts: true
    vars:
      role_vmware_guest_advanced_settings__custom_settings:
        - key: disk.EnableUUID
          value: 'TRUE'
        - key: tools.remindInstall
          value: 'TRUE'
        - key: some.parameter
          value: ''

    tasks:

      - name: Find the host - case of hostname doesn't matter
        ansible.builtin.include_role:
          name: role_vmware_guest_find
        vars:
          role_vmware_guest_find__hostname: "{{ hostvars[inventory_hostname]['vmware_guest_name'] }}"
        no_log: true

      - name: Call role_vmware_guest_advanced_settings
        ansible.builtin.include_role:
          name: role_vmware_guest_advanced_settings
        vars:
          role_vmware_guest_advanced_settings__custom_settings:
            - key: disk.EnableUUID
              value: true

  ...

  ```

## License
---

Private

## Author Information
---

* Original by Red Hat Consulting

> NOTE:  this code was built by Red Hat Consulting, if there is a issue, reach out to Red Hat Consulting not Red Hat Support.
