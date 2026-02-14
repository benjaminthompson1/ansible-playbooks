ansible-playbooks\README.md
</file_path>

<edit_description>
Rewrite README.md to reflect backup playbook changes
</edit_description>

```
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

- Ansible 2.9 or higher
- Python 3.6 or higher (for z/OS)
- SSH access to the target hosts
- `xz` utility (required by the `backup_z31c_volumes.yml` playbook)

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

- **`backup_z31c_volumes.yml`** – Backs up z31c volumes as a full backup, storing the resulting `.tar.xz` file with a timestamp.  
  The playbook now uses an **async tar job** that runs XZ with multi‑threading (`-T0 --ultra-fast`) and a progress checkpoint (`--checkpoint=10000`).  
  It keeps the most recent `N` backups (default 4) and deletes older archives automatically.

- **`optimize_zpdt_network.yml`** – Optimizes the zpdt network.
- **`provision_volumes.yml`** – Provisions volumes.

## ⚙️ Backup Playbook Highlights

- **Full backup**: No incremental logic – a fresh archive is created on each run.
- **Efficient**: Multi‑threaded XZ (`-T0 --ultra-fast`) plus async execution reduces CPU contention and keeps the control node responsive.
- **Retention**: Keeps only the latest `keep_backups` archives, freeing disk space automatically.
- **Easy restore**: The `.tar.xz` can be extracted on any Linux host with a single command:
  
  ```sh
  tar -xvf volumes-z31c-*.tar.xz -C /desired/restore/path
  ```

## 🤝 Contributing

Contributions are welcome! Please submit pull requests with your changes, and make sure to update the documentation accordingly.