# role_vmware_guest_find
---

This role will do the following steps:

* Will find a VMware guest host by trying in it's original passed to the role by variable `role_vmware_guest_find__hostname`.
* If it doesn't find it, it can be assumed it's not on this vCenter.
* If found, it will populate a number of role named variables to use in other playbooks or roles.

## Role Requirements
---

### Python Modules

* The python modules `PyVmomi` and `aiohttp` are required on the target hosts or execution enviroment.

## Requirements

The following requirements are need to ensure the role runs correctly.

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

* Variable provided by `ansible.builtin.include_role` `vars` or in AAP Job Template Extra Variables field:

| Job Template Variables | Comments |
|:-|:-|
| **role_vmware_guest_find__hostname**<br><span style="color:purple">Text</span><br><span style="color:red">Required</span> / <span style="color:blue">String</span> | Name of the guest as it appears in VMware vSphere.<br><br>_Examples_:<br>`role_vmware_guest_find__hostname: "{{ hostvars[inventory_hostname]['vmware_guest_name'] }}"`<br><br>`role_vmware_guest_find__hostname: ECTXTAPTR107` |

## Playbook Example
---

```yaml
---
# site.yml for vmware_vm_facts

- name: Project vmware_vm_facts
  hosts: all
  gather_facts: true

  tasks:

    - name: Find the host - case of hostname doesn't matter
      ansible.builtin.include_role:
        name: role_vmware_guest_find
      vars:
        role_vmware_guest_find__hostname: "{{ hostvars[inventory_hostname]['vmware_guest_name'] }}"

    - name: Print the findings
      ansible.builtin.debug:
        msg:
          - "role_vmware_guest_find__cluster: ........ {{ role_vmware_guest_find__cluster }}"
          - "role_vmware_guest_find__datacenter: ..... {{ role_vmware_guest_find__datacenter }}"
          - "role_vmware_guest_find__folders: ........ {{ role_vmware_guest_find__folders }}"
          - "role_vmware_guest_find__guest_name: ..... {{ role_vmware_guest_find__guest_name }}"
          - "role_vmware_guest_find__moid: ........... {{ role_vmware_guest_find__moid }}"
          - "role_vmware_guest_find__uuid: ........... {{ role_vmware_guest_find__uuid }}"
          - "role_vmware_guest_find__vm_name: ........ {{ role_vmware_guest_find__vm_name }}"
      when: role_vmware_guest_find__debug_mode | bool

    - name: Gather some info from a guest using those variable returned
      community.vmware.vmware_guest_info:
        hostname: "{{ vcenter_hostname }}"
        username: "{{ vcenter_username }}"
        password: "{{ vcenter_password }}"
        validate_certs: "{{ vcenter_certificates }}"
        datacenter: "{{ role_vmware_guest_find__datacenter }}"
        folder: "{{ role_vmware_guest_find__folders }}"
        name: "{{ role_vmware_guest_find__guest_name }}"
        schema: "vsphere"
      delegate_to: localhost
      register: role_vmware_guest_facts__info

    - name: Get extraConfig values
      ansible.builtin.set_fact:
        __extra_config_value: "{{ item.value }}"
        __extra_config_value_found: true
      loop: "{{ role_vmware_guest_facts__info }}"
      when: item.key == 'disk.EnableUUID'

...

```


## License
---

Private

## Author Information
---

* Original by Red Hat Consulting

> NOTE:  this code was built by Red Hat Consulting, if there is a issue, reach out to Red Hat Consulting not Red Hat Support.
