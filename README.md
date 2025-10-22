# Slurm Deployment with Ansible

This Ansible playbook deploys a Slurm cluster with munge authentication, NFS-shared /home directory, and full accounting support via slurmdbd.

## Files

- `ansible.cfg` - Ansible configuration that imports the inventory
- `inventory.ini` - Ansible inventory with head and worker nodes
- `deploy_slurm.yml` - Main playbook for deployment
- `slurm.conf.j2` - Jinja2 template for Slurm configuration
- `slurmdbd.conf.j2` - Jinja2 template for Slurm Database Daemon configuration

## Cluster Configuration

- **Head Node**: basil.caelum.ci.dev (runs slurmctld)
- **Worker Nodes**:
  - orithia.caelum.ci.dev
  - ainia.caelum.ci.dev
  - phoebe.caelum.ci.dev

## What the Playbook Does

1. Sets hostname on each node using `hostnamectl` to match inventory name
2. Installs munge and slurm packages on all nodes
3. Installs slurmctld, slurmdbd, MariaDB, and NFS server on the head node
4. Creates `/var/spool/slurmctld` with ownership `slurm:slurm` and permissions `775`
5. Configures NFS export of `/home` with `async` and `no_root_squash` options
6. Sets up MariaDB database and creates `slurm_acct_db` database with user
7. Generates `slurmdbd.conf` and starts slurmdbd service
8. Initializes accounting database with cluster, account, and user
9. Generates munge key on head node
10. Copies munge key from head node to all workers
11. Installs NFS client on workers and mounts `/home` from head node (persistent via fstab)
12. Runs `slurmd -C` on each worker to gather node configuration
13. Runs `uname -m` on each worker to determine architecture feature
14. Generates `slurm.conf` with compute node information and accounting enabled
15. Distributes configuration to all nodes
16. Starts all Slurm, munge, and NFS services

## Usage

Run the playbook with:

```bash
ansible-playbook deploy_slurm.yml
```

The inventory is automatically loaded from `inventory.ini` via `ansible.cfg`.

## Requirements

- Ansible installed on control machine
- SSH access to all nodes with sudo privileges
- Ubuntu/Debian-based systems (uses apt)
- Internet access to download packages

## Post-Deployment

After deployment, verify the cluster status:

```bash
# On head node
sinfo
scontrol show nodes
```

### Verify Accounting

Check that accounting is working:

```bash
# Check slurmdbd is running
systemctl status slurmdbd

# View cluster and accounts
sacctmgr show cluster
sacctmgr show account
sacctmgr show user

# View accounting information
sacct
scontrol show config | grep AccountingStorage
```

The playbook automatically creates:
- Cluster: `compute`
- Account: `ocaml` (on cluster `compute`)
- User: `mte24` (associated with account `ocaml`)

### Database Credentials

- **Database**: `slurm_acct_db`
- **User**: `slurm`
- **Password**: `slurmdbpass` (change this in the playbook for production use)

The database password is set in two places:
- `deploy_slurm.yml` (MySQL user creation task)
- `slurmdbd.conf.j2` (StoragePass parameter)
