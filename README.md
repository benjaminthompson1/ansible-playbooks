# 📘 Ansible Playbook Repository

This repository contains a collection of **Ansible playbooks** designed for managing and automating tasks on both **z/OS** and **Linux** systems.

## 📁 Directory Structure
```
ansible-playbooks/
├── ansible-playbooks-zos/
│   ├── post_uuid.yml
│   └── zos_ping.yml
└── ansible-playbooks-linux/
    ├── backup_z31c_volumes.yml
    ├── optimize_zpdt_network.yml
    └── provision_volumes.yml
```

## 🔧 Requirements

- Ansible 2.9 or higher
- Python 3.6 or higher (for z/OS)
- SSH access to the target hosts

## 🚀 Getting Started

1. Clone the repository:

    ```sh
    git clone https://github.com/yourusername/ansible-playbooks.git
    ```

2. Change to the appropriate playbook directory (either `ansible-playbooks-zos` or `ansible-playbooks-linux`) depending on the target system:

    ```sh
    cd ansible-playbooks/ansible-playbooks-zos
    ```

    or

    ```sh
    cd ansible-playbooks/ansible-playbooks-linux
    ```

3. Modify the `inventories/inventory.yml` file to include the target hosts for your playbooks.

4. Run the desired playbook:

    ```sh
    ansible-playbook -i inventories/inventory.yml playbook.yml
    ```

## 📚 Playbook Descriptions

### z/OS Playbooks

- **`post_uuid.yml`** – POSTs a z/OS UUID via z/OSMF.
- **`zos_ping.yml`** – Pings a z/OS system to test network connectivity.

### Linux Playbooks

- **`backup_z31c_volumes.yml`** – Backs up z31c volumes and keeps the latest N backups.
- **`optimize_zpdt_network.yml`** – Optimizes the zpd network.
- **`provision_volumes.yml`** – Provisions volumes.

## 🤝 Contributing

Contributions are welcome! Please submit pull requests with your changes, and make sure to update the documentation accordingly.