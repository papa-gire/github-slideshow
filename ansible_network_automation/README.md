# Ansible Network Automation Playbook

This playbook automates common network device tasks:
1.  **Configuration Management**: Applies base configurations and interface settings to network devices.
2.  **Configuration Backup**: Backs up the current running/active configuration of network devices.

It is designed to work with Cisco IOS and Juniper Junos devices, but can be extended for other vendors.

## Directory Structure

```
ansible_network_automation/
├── playbook.yml                # Main Ansible playbook
├── inventory.ini               # Example inventory file
├── host_vars/                  # Directory for host-specific variables
│   ├── router1.example.com.yml # Example host variables for a Cisco router
│   └── junos_switch1.example.com.yml # Example host variables for a Junos switch
├── group_vars/                 # Directory for group-specific variables
│   ├── cisco_devices.yml       # Example group variables for Cisco devices
│   └── juniper_devices.yml     # Example group variables for Juniper devices
├── roles/
│   ├── configure_device/       # Role for applying configurations
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── templates/
│   │       └── interface_config.j2 # Template for interface configurations
│   └── backup_device/          # Role for backing up configurations
│       └── tasks/
│           └── main.yml
├── network_backups/            # Default directory where backups will be stored (created on run)
├── build/                      # Default directory for temporary files (created on run)
└── README.md                   # This file
```

## Prerequisites

1.  **Ansible Installed**: Ensure Ansible is installed on the machine running the playbook.
    (See: [https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html))

2.  **Python Libraries for Network Modules**: Some Ansible network modules require specific Python libraries (e.g., `paramiko` for SSH, `ncclient` for NETCONF). Install them as needed.
    ```bash
    pip install paramiko ncclient jmespath
    ```

3.  **Ansible Collections**: Install the necessary Ansible collections for network automation.
    ```bash
    ansible-galaxy collection install ansible.netcommon
    ansible-galaxy collection install cisco.ios
    ansible-galaxy collection install junipernetworks.junos
    ansible-galaxy collection install ansible.utils  # Useful for filters like ipaddr
    ```

4.  **Network Device Accessibility**:
    *   The control node (where Ansible runs) must have network connectivity (SSH, NETCONF) to the target devices.
    *   Credentials (username/password/SSH keys) must be valid for accessing the devices.

## Setup and Configuration

1.  **Clone/Download Playbook**: Get the playbook files onto your Ansible control node.

2.  **Inventory (`inventory.ini`)**:
    *   Edit `inventory.ini` to define your network devices.
    *   Add your routers and switches under appropriate groups (e.g., `[cisco_routers]`, `[juniper_switches]`).
    *   Specify `ansible_host` (IP address or FQDN) for each device.
    *   **Crucially, ensure `ansible_network_os` is correctly set for each device group or individual host.** This tells Ansible which network module collection to use (e.g., `cisco.ios.ios`, `junipernetworks.junos.junos`). Examples are provided in `inventory.ini` and `group_vars`.

3.  **Credentials**:
    *   **Ansible Vault (Recommended)**: Store all sensitive data like SSH passwords, enable passwords, etc., in Ansible Vault.
        *   Update `group_vars/cisco_devices.yml` and `group_vars/juniper_devices.yml` (and any `host_vars`) to reference vault variables for `ansible_password`, `ansible_become_password`, etc.
        *   Example: `ansible_password: "{{ vault_my_password }}"`
        *   Create vault files (e.g., `ansible-vault create group_vars/vault.yml`) and define these variables.
    *   **Plain Text (Not Recommended for Production)**: You can temporarily set credentials directly in `inventory.ini` or `group_vars` files, but this is insecure.

4.  **Host Variables (`host_vars/`)**:
    *   For each device you want to configure interfaces on, create a YAML file in the `host_vars` directory named `<inventory_hostname>.yml` (e.g., `host_vars/router1.example.com.yml`).
    *   Define the `interfaces` variable within these files as shown in the examples. This data is used by the `interface_config.j2` template.
        ```yaml
        # Example for host_vars/my_router.yml
        interfaces:
          - name: GigabitEthernet0/0
            description: "WAN Link"
            ip_address: 10.0.0.1
            subnet_mask: 255.255.255.252
            state: "no shutdown"
          - name: GigabitEthernet0/1
            description: "LAN Interface"
            ip_address: 192.168.1.1
            subnet_mask: 255.255.255.0
            state: "no shutdown"
        ```

5.  **Group Variables (`group_vars/`)**:
    *   Modify `group_vars/cisco_devices.yml` and `group_vars/juniper_devices.yml` to set common variables for your device groups (e.g., `ansible_user`, default `ansible_network_os`, connection parameters).

6.  **Playbook Variables (`playbook.yml`)**:
    *   `backup_dir`: Change if you want backups stored elsewhere (default: `./network_backups`).
    *   `build_dir`: Change if you want temporary configuration files stored elsewhere (default: `./build`).
    *   The `ansible_user` and `ansible_password` in `playbook.yml` are global fallbacks but are better defined in inventory or group_vars.

## Running the Playbook

*   **Run both configuration and backup for all devices**:
    ```bash
    ansible-playbook -i inventory.ini playbook.yml
    ```
    If using Ansible Vault, you'll be prompted for the vault password, or you can use `--ask-vault-pass` or environment variables.

*   **Run for a specific group or host**:
    ```bash
    ansible-playbook -i inventory.ini playbook.yml --limit cisco_routers
    ansible-playbook -i inventory.ini playbook.yml --limit router1.example.com
    ```

*   **Only run backup tasks**:
    The playbook uses boolean variables `manage_config` and `manage_backup` (defaulting to `true`) to control role execution.
    ```bash
    ansible-playbook -i inventory.ini playbook.yml -e "manage_config=false"
    ```

*   **Only run configuration tasks**:
    ```bash
    ansible-playbook -i inventory.ini playbook.yml -e "manage_backup=false"
    ```

*   **Dry Run (Check Mode)**: See what changes would be made without actually applying them.
    ```bash
    ansible-playbook -i inventory.ini playbook.yml --check
    ```
    *Note: Check mode support can vary between network modules. Some modules might still make limited read-only connections.*

## Customization

*   **Adding Support for Other Network OS**:
    1.  Install the relevant Ansible collection (e.g., `arista.eos`, `vyos.vyos`).
    2.  Update `playbook.yml`'s `ansible_network_os_mappings` if using the dynamic assignment, or define `ansible_network_os` in your inventory/group_vars for the new devices.
    3.  Add new tasks or conditional blocks in the roles for the specific OS, using its modules (e.g., `arista.eos.eos_config`).
    4.  You may need to adjust the `interface_config.j2` template or create new templates if the configuration syntax differs significantly.

*   **Modifying Configurations**:
    *   **Base Configs**: Edit `roles/configure_device/tasks/main.yml` to change or add commands for base configurations.
    *   **Interface Configs**: Modify `roles/configure_device/templates/interface_config.j2` to change how interface configurations are generated. Add more parameters (e.g., VLANs, speed, duplex) as needed and update your `host_vars` files accordingly.
    *   **New Configuration Types**: Add new tasks to `roles/configure_device/tasks/main.yml` for other configurations (e.g., VLANs, ACLs, routing protocols) using appropriate Ansible modules.

*   **Modifying Backups**:
    *   Edit `roles/backup_device/tasks/main.yml` to change backup commands or file naming conventions.
    *   The `ansible.netcommon.net_backup` module is included as a generic example; it might require the `ansible.utils` collection for full functionality or specific arguments depending on the target OS.

*   **Credentials Management**:
    *   Implement SSH key-based authentication for enhanced security. Update `ansible_user` and potentially `ansible_ssh_private_key_file` in your inventory or `group_vars`.

## Troubleshooting

*   **Verbosity**: Use `-v`, `-vv`, or `-vvv` with `ansible-playbook` for more detailed output.
*   **Ansible Network Debug Guide**: [https://docs.ansible.com/ansible/latest/network/user_guide/network_debug_troubleshooting.html](https://docs.ansible.com/ansible/latest/network/user_guide/network_debug_troubleshooting.html)
*   **Check Module Documentation**: Refer to the documentation for specific network modules (e.g., `cisco.ios.ios_config`, `junipernetworks.junos.junos_config`) on the Ansible documentation website or Ansible Galaxy.
*   **Connectivity**: Ensure basic SSH/NETCONF connectivity to the device from the control node using native tools (e.g., `ssh user@device`, `netcat -vz device_ip ssh_port`).
*   **Python Environment**: Issues can sometimes arise from the Python environment or missing libraries on the control node.

This playbook provides a foundational framework. Adapt and extend it to meet your specific network automation requirements.
Remember to test thoroughly in a lab environment before running against production devices.
