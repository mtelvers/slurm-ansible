# Slurm Deployment with Ansible

This Ansible playbook deploys a Slurm cluster with munge authentication, NFS-shared /home directory, full accounting support via slurmdbd, cgroups-based resource management, and REST API access via slurmrestd.

## Files

- `ansible.cfg` - Ansible configuration that imports the inventory
- `inventory.ini` - Ansible inventory with head and worker nodes
- `deploy_slurm.yml` - Main playbook for deployment
- `slurm.conf.j2` - Jinja2 template for Slurm configuration
- `slurmdbd.conf.j2` - Jinja2 template for Slurm Database Daemon configuration
- `cgroup.conf.j2` - Jinja2 template for cgroups resource constraints

## Cluster Configuration

- **Head Node**: basil.caelum.ci.dev (runs slurmctld, slurmdbd, slurmrestd)
- **Worker Nodes**:
  - orithia.caelum.ci.dev
  - ainia.caelum.ci.dev
  - phoebe.caelum.ci.dev

### Resource Management with cgroups

The cluster uses Linux cgroups to enforce resource limits on jobs:

- **Process tracking**: `proctrack/cgroup`
- **Task management**: `task/cgroup` with affinity support
- **Job accounting**: `jobacct_gather/cgroup`
- **Cgroup constraints**:
  - `ConstrainCores=yes` - Enforce CPU core limits
  - `ConstrainDevices=yes` - Restrict device access
  - `ConstrainRAMSpace=yes` - Enforce memory limits
  - `ConstrainSwapSpace=yes` - Enforce swap limits
- **Default memory per node**: 16384 MB (16 GB)

This configuration is based on the guide at https://www.tunbury.org/2025/08/06/slurm-limits/

## What the Playbook Does

1. Sets hostname on each node using `hostnamectl` to match inventory name
2. Installs munge and slurm packages on all nodes
3. Installs slurmctld, slurmdbd, slurmrestd, MariaDB, and NFS server on the head node
4. Creates `/var/spool/slurmctld` with ownership `slurm:slurm` and permissions `775`
5. Configures NFS export of `/home` with `async` and `no_root_squash` options
6. Sets up MariaDB database and creates `slurm_acct_db` database with user
7. Generates `slurmdbd.conf` with JWT authentication and starts slurmdbd service
8. Initializes accounting database with cluster, account, and user
9. Generates munge key on head node
10. Generates JWT key for REST API authentication at `/var/spool/slurmctld/jwt_hs256.key`
11. Copies munge key from head node to all workers
12. Installs NFS client on workers and mounts `/home` from head node (persistent via fstab)
13. Runs `slurmd -C` on each worker to gather node configuration
14. Runs `uname -m` on each worker to determine architecture feature
15. Generates `slurm.conf` with compute node information, accounting, JWT authentication, and cgroups enabled
16. Generates `cgroup.conf` with resource constraint settings
17. Distributes configuration files to all nodes
18. Configures slurmrestd systemd service to run as `slurm` user with proper runtime directory
19. Starts all Slurm, munge, NFS, and slurmrestd services

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
- Cluster: `caelum`
- Account: `ocaml` (on cluster `caelum`)
- User: `mte24` (associated with account `ocaml`)

### Database Credentials

- **Database**: `slurm_acct_db`
- **User**: `slurm`
- **Password**: `slurmdbpass` (change this in the playbook for production use)

The database password is set in two places:
- `deploy_slurm.yml` (MySQL user creation task)
- `slurmdbd.conf.j2` (StoragePass parameter)

### Verify cgroups

Check that cgroups are properly configured:

```bash
# On a worker node, check cgroup version
cat /proc/cgroups

# View Slurm configuration
scontrol show config | grep -i cgroup
scontrol show config | grep -i proctrack
scontrol show config | grep -i task

# Submit a test job and check cgroup enforcement
srun --mem=1G hostname

# Check job memory limits
scontrol show job <job_id> | grep -i mem
```

Jobs that exceed their allocated memory will be terminated by the cgroup controller.

### Slurm REST API (slurmrestd)

The playbook deploys slurmrestd on the head node for REST API access to Slurm.

**Endpoints:**
- **TCP Socket**: `basil.caelum.ci.dev:6820` (remote access)
- **Unix Socket**: `/run/slurmrestd/slurmrestd.socket` (local access on head node)

**Authentication:**
The REST API uses JWT (JSON Web Token) authentication. The JWT key is stored at `/var/spool/slurmctld/jwt_hs256.key` on the head node.

**Security:**
- slurmrestd runs as the `slurm` user (required for munge authentication with slurmdbd)
- JWT authentication is enabled via `AuthAltTypes=auth/jwt` in both slurm.conf and slurmdbd.conf
- The service is configured with proper runtime directory permissions
- Note: While the Slurm documentation recommends running slurmrestd as a dedicated unprivileged user, munge authentication requires matching UIDs between communicating processes. Since slurmrestd needs to connect to slurmdbd (which runs as `slurm`), both must run as the same user.

**Verify slurmrestd:**

```bash
# Check service status
systemctl status slurmrestd

# Verify it's listening on port 6820
ss -tulpn | grep 6820

# Check Unix socket
ls -la /run/slurmrestd/slurmrestd.socket
```

**Example API usage:**

```bash
# Get a JWT token (requires valid Slurm user)
eval $(scontrol token username=mte24 lifespan=3600)

# Test connection
curl -H "X-SLURM-USER-NAME: mte24" -H "X-SLURM-USER-TOKEN: $SLURM_JWT" \
  http://basil.caelum.ci.dev:6820/slurm/v0.0.40/ping

# List nodes
curl -H "X-SLURM-USER-NAME: mte24" -H "X-SLURM-USER-TOKEN: $SLURM_JWT" \
  http://basil.caelum.ci.dev:6820/slurm/v0.0.40/nodes

# List jobs
curl -H "X-SLURM-USER-NAME: mte24" -H "X-SLURM-USER-TOKEN: $SLURM_JWT" \
  http://basil.caelum.ci.dev:6820/slurm/v0.0.40/jobs

# Query accounts (requires slurmdbd)
curl -H "X-SLURM-USER-NAME: mte24" -H "X-SLURM-USER-TOKEN: $SLURM_JWT" \
  http://basil.caelum.ci.dev:6820/slurmdb/v0.0.40/accounts
```

**Submitting jobs:**

```bash
# Get a JWT token
eval $(scontrol token username=mte24 lifespan=3600)

# Create a job submission file
cat > job.json << 'EOF'
{
  "script": "#!/bin/bash\nhostname",
  "job": {
    "name": "hostname_test",
    "account": "ocaml",
    "nodes": "1",
    "tasks": 1,
    "current_working_directory": "/home/mte24",
    "standard_output": "/home/mte24/slurm-%j.out",
    "environment": ["PATH=/usr/bin:/bin"]
  }
}
EOF

# Submit the job
curl -X POST \
  -H "X-SLURM-USER-NAME: mte24" \
  -H "X-SLURM-USER-TOKEN: $SLURM_JWT" \
  -H "Content-Type: application/json" \
  --data-binary @job.json \
  http://basil.caelum.ci.dev:6820/slurm/v0.0.40/job/submit

# Check job status (replace JOB_ID with the returned job_id)
curl -H "X-SLURM-USER-NAME: mte24" -H "X-SLURM-USER-TOKEN: $SLURM_JWT" \
  http://basil.caelum.ci.dev:6820/slurm/v0.0.40/job/JOB_ID
```

**Note:** The `script` field must be at the top level of the JSON, not nested inside the `job` object. Use `\n` for newlines in the script. Always use `--data-binary` (not `--data`) to avoid corrupting the payload.

For complete API documentation, see: https://slurm.schedmd.com/rest_api.html
