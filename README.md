# 📘 Ansible Playbook Repository

This repository contains a collection of **Ansible playbooks** designed for managing and automating tasks on both **z/OS** and **Linux** systems.

## 📁 Directory Structure
```
ansible-playbook-repository/
├── ansible-playbooks-zos/
│ ├── cics-egui-setup.yml
│ ├── COBOL_Customization_Setup.yml
│ ├── configure_jes_checkpoint.yml
│ ├── copy_ibmprods_ispplib.yml
│ ├── create_jobcard_rexx.yml
│ ├── rmf-configuration.yml
│ ├── smpe_zfs_setup.yml
│ ├── update_mq_parameters.yml
│ ├── zos_automount_setup.yml
│ ├── zos_dataset_commander_customization.yml
│ ├── zos_housekeeping.yml
│ ├── zos_ping.yml
│ ├── zos-proclib-setup.yml
│ └── zos_system_configuration.yml
└── ansible-playbooks-linux/
  ├── backup_z25c_volumes.yml
  ├── install_docker.yml
  ├── optimize_zpd_network.yml
  ├── provision_volumes.yml
  ├── restore_backup.yml
  ├── start_containers_and_services.yml
  ├── stop_containers_and_services.yml
  └── vault.yml
```

## 🔧 Requirements

- Ansible 2.9 or higher
- Python 3.6 or higher (for z/OS)
- SSH access to the target hosts

## 🚀 Getting Started

1. Clone the repository:

    ```
    git clone https://github.com/yourusername/ansible-playbook-repository.git
    ```

2. Change to the appropriate playbook directory (either `ansible-playbooks-zos` or `ansible-playbooks-linux`) depending on the target system:

    ```
    cd ansible-playbook-repository/ansible-playbooks-zos
    ```

    or

    ```
    cd ansible-playbook-repository/ansible-playbooks-linux
    ```

3. Modify the `inventory.ini` file to include the target hosts for your playbooks.

4. Run the desired playbook:

    ```
    ansible-playbook -i inventory.ini playbook1.yml
    ```

## 📚 Playbook Descriptions

### z/OS Playbooks

- `cics-egui-setup.yml`: Configures the CICS EGUI (Enhanced General User Interface) product for use.
- `COBOL_Customization_Setup.yml`: Customizes the COBOL installation with user-defined parameters.
- `configure_jes_checkpoint.yml`: Configures the JES checkpoint service for job restart capabilities.
- `copy_ibmprods_ispplib.yml`: Copies the IBMPRODS ISPPLIB (IBM Program Products) dataset to the target system.
- `create_jobcard_rexx.yml`: Generates JCL (Job Control Language) jobcards using REXX (Restructured Extended Executor).
- `rmf-configuration.yml`: Configures the Resource Measurement Facility (RMF) for performance monitoring and reporting.
- `smpe_zfs_setup.yml`: Configures the SMP/E (System Modification Program/Extended) ZFS (z/OS File System) distribution environment.
- `update_mq_parameters.yml`: Updates the configuration parameters for IBM MQ (Message Queue) on z/OS.
- `zos_automount_setup.yml`: Configures automount support for file systems on z/OS.
- `zos_dataset_commander_customization.yml`: Customizes the z/OS Dataset Commander product with user-defined parameters.
- `zos_housekeeping.yml`: Performs routine system housekeeping tasks on z/OS.
- `zos_ping.yml`: Pings a z/OS system to test network connectivity.
- `zos-proclib-setup.yml`: Configures the z/OS PROCLIB (Procedure Library) dataset for user-defined procedures.
- `zos_system_configuration.yml`: Configures various system parameters and settings on z/OS.

### Linux Playbooks

- `backup_z25c_volumes.yml`: Backs up the z25c volumes.
- `install_docker.yml`: Install Docker CE.
- `optimize_zpd_network.yml`: Optimizes the zpd network.
- `provision_volumes.yml`: Provisions volumes.
- `restore_backup.yml`: Restores a backup.
- `start_containers_and_services.yml`: Starts containers and services.
- `stop_containers_and_services.yml`: Stops containers and services.
- `vault.yml`: Configures a Vault instance.

## 🤝 Contributing

Contributions are welcome! Please submit pull requests with your changes, and make sure to update the documentation accordingly.
Please let me know if you need further assistance or if there are any specific formatting elements you would like to include.

