# Slurm 20.11.9 Installation Guide (Controller + Compute Node)

Cleaned up from the shell histories of the `controller` and `compute1` machines.

| Role       | Hostname     | IP               |
|------------|--------------|------------------|
| Controller | `controller` | `192.168.76.33`  |
| Compute    | `compute1`   | `192.168.76.108` |

- Slurm version: `20.11.9`
- Install prefix: `/slurmdir/slurm-20.11.9`
- User used for the daemons: `satej`
- Config file location: `/slurmdir/slurm-20.11.9/etc/slurm.conf`

> Legend: **[C]** = controller only, **[N]** = compute node only, **[BOTH]** = run on both machines.

---

## 1. Set hostname and network

**[C]**
```bash
sudo hostnamectl set-hostname controller
```

**[N]**
```bash
sudo hostnamectl set-hostname compute1
```

Check the IP address (set a static IP with `sudo nmtui` if needed):
```bash
ip -br a
```

**[BOTH]** Add both machines to `/etc/hosts`:
```bash
sudo vim /etc/hosts
```
```
192.168.76.33    controller
192.168.76.108   compute1
```

Test:
```bash
ping controller
ping compute1
```

---

## 2. Fix apt sources (only if `apt update` fails due to the CD-ROM)

**[BOTH]**
```bash
sudo grep -rn "cdrom" /etc/apt/
sudo nano /etc/apt/sources.list     # comment out the "deb cdrom:" line
sudo apt update
```

---

## 3. Install dependencies

**[BOTH]**
```bash
sudo apt update
sudo apt install -y build-essential munge libmunge-dev libmunge2 \
    libmysqlclient-dev libssl-dev libpam0g-dev libnuma-dev perl wget
```

> The package is `libnuma-dev` (not `libanuma-dev`, which was a typo in the history).

---

## 4. Time sync (recommended)

Slurm and Munge fail if clocks differ between nodes.

**[BOTH]**
```bash
sudo apt install -y chrony
sudo systemctl enable --now chrony
```

---

## 5. Set up Munge authentication

**[C]** Create a key (or use the one installed with the package):
```bash
sudo /usr/sbin/mungekey --create      # or: sudo create-munge-key
```

Copy the key to the compute node:
```bash
sudo scp /etc/munge/munge.key satej@compute1:/tmp/munge.key
```

**[N]** Install the key:
```bash
sudo mv /tmp/munge.key /etc/munge/munge.key
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
```

**[BOTH]** Start Munge:
```bash
sudo systemctl enable --now munge
sudo systemctl status munge
```

**[C]** Test across nodes:
```bash
munge -n | ssh compute1 unmunge
```
You should see `STATUS: Success (0)`.

> The munge key step was not in your history files, but it is required for a multi-node cluster.

---

## 6. Download, build and install Slurm

**[BOTH]**
```bash
sudo mkdir /slurmdir
sudo chmod 777 /slurmdir
cd /slurmdir
wget https://download.schedmd.com/slurm/slurm-20.11.9.tar.bz2
tar -xvjf slurm-20.11.9.tar.bz2
cd slurm-20.11.9
./configure --prefix=/slurmdir/slurm-20.11.9/
make -j"$(nproc)"
make install
```

> Note: the command is `make install` (the history had a typo `.make install`).

---

## 7. Add Slurm to PATH (make it permanent)

**[BOTH]**
```bash
cat >> ~/.bashrc << 'EOF'
export PATH="/slurmdir/slurm-20.11.9/bin:/slurmdir/slurm-20.11.9/sbin:$PATH"
export LD_LIBRARY_PATH="/slurmdir/slurm-20.11.9/lib:$LD_LIBRARY_PATH"
EOF
source ~/.bashrc
```

> Common mistakes seen in the history: spaces around `=` in `export PATH = ...`, and the variable name `LD_LIBRARY` instead of `LD_LIBRARY_PATH`.

---

## 8. Create spool and log directories

**[C]**
```bash
sudo mkdir -p /var/spool/slurm/ctld
sudo chown satej:satej /var/spool/slurm/ctld
sudo chmod 755 /var/spool/slurm

sudo touch /var/log/slurmctld.log
sudo chown satej:satej /var/log/slurmctld.log
```

**[N]**
```bash
sudo mkdir -p /var/spool/slurmd
sudo chown satej:satej /var/spool/slurmd
sudo chmod 755 /var/spool/slurmd

sudo touch /var/log/slurmd.log
sudo chown satej:satej /var/log/slurmd.log
sudo chmod 644 /var/log/slurmd.log
```

> The permission/ownership errors on spool and log files were what made `slurmd`/`slurmctld` fail to start in the history. Creating them up front avoids that.

---

## 9. Configure `slurm.conf` (on the controller)

**[C]**
```bash
cd /slurmdir/slurm-20.11.9/etc
cp slurm.conf.example slurm.conf
```

Get the node's hardware values (run on the compute node, copy the output line):

**[N]**
```bash
/slurmdir/slurm-20.11.9/sbin/slurmd -C
```

Edit the config on the controller:
```bash
vim /slurmdir/slurm-20.11.9/etc/slurm.conf
```

Minimal example (adjust values to your machines):
```ini
ClusterName=cluster
SlurmctldHost=controller

SlurmUser=satej
SlurmdUser=root

AuthType=auth/munge
StateSaveLocation=/var/spool/slurm/ctld
SlurmdSpoolDir=/var/spool/slurmd
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmdPidFile=/var/run/slurmd.pid
SlurmctldLogFile=/var/log/slurmctld.log
SlurmdLogFile=/var/log/slurmd.log

SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core
ProctrackType=proctrack/linuxproc
TaskPlugin=task/none

# Paste the values from `slurmd -C` here
NodeName=compute1 CPUs=2 RealMemory=1900 State=UNKNOWN
PartitionName=debug Nodes=compute1 Default=YES MaxTime=INFINITE State=UP
```

> `slurm.conf` **must be identical on every node.**

---

## 10. Copy `slurm.conf` to the compute node

**[C]**
```bash
scp /slurmdir/slurm-20.11.9/etc/slurm.conf satej@compute1:/tmp/slurm.conf
```

**[N]**
```bash
sudo mv /tmp/slurm.conf /slurmdir/slurm-20.11.9/etc/slurm.conf
```

> Use `scp` for remote copies. `cp user@host:/path` (used in the history) only works locally and fails.

---

## 11. Install systemd service files

**[C]**
```bash
sudo ln -s /slurmdir/slurm-20.11.9/etc/slurmctld.service /usr/lib/systemd/system/slurmctld.service
sudo ln -s /slurmdir/slurm-20.11.9/etc/slurmd.service    /usr/lib/systemd/system/slurmd.service
sudo systemctl daemon-reload
```

**[N]**
```bash
sudo ln -s /slurmdir/slurm-20.11.9/etc/slurmd.service /usr/lib/systemd/system/slurmd.service
sudo systemctl daemon-reload
```

---

## 12. Start the services

**[C]**
```bash
sudo systemctl enable --now slurmctld
sudo systemctl enable --now slurmd        # only if the controller is also a compute node
sudo systemctl status slurmctld
```

**[N]**
```bash
sudo systemctl enable --now slurmd
sudo systemctl status slurmd
```

> Use `systemctl enable`, not `service <name> enable` (that was invalid in the history).

**If a service fails to start**, run it in the foreground to see the real error:
```bash
/slurmdir/slurm-20.11.9/sbin/slurmctld -D -vvv     # controller
/slurmdir/slurm-20.11.9/sbin/slurmd -D -vvv        # compute node
```

---

## 13. Verify the cluster

**[C]**
```bash
sinfo
scontrol show nodes
srun -N1 hostname
```

If the node shows `down` or `unk`:
```bash
sudo scontrol update NodeName=compute1 State=RESUME
```

Test a batch job:
```bash
cat > test.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=test
#SBATCH --output=test_%j.out
hostname
sleep 10
EOF

sbatch test.sh
squeue
```

---

## 14. (Optional) Accounting with slurmdbd + MariaDB

**[C]** Install the database (the package is `mariadb-server`, not `mysql`):
```bash
sudo apt install -y mariadb-server
sudo systemctl enable --now mariadb
```

Create the database and user:
```bash
sudo mysql -e "CREATE DATABASE slurm_acct_db;"
sudo mysql -e "CREATE USER 'slurm'@'localhost' IDENTIFIED BY 'StrongPassword';"
sudo mysql -e "GRANT ALL ON slurm_acct_db.* TO 'slurm'@'localhost'; FLUSH PRIVILEGES;"
```

Configure slurmdbd:
```bash
cd /slurmdir/slurm-20.11.9/etc
cp slurmdbd.conf.example slurmdbd.conf
vim slurmdbd.conf
```
```ini
DbdHost=controller
SlurmUser=satej
StorageType=accounting_storage/mysql
StorageUser=slurm
StoragePass=StrongPassword
StorageLoc=slurm_acct_db
LogFile=/var/log/slurmdbd.log
PidFile=/var/run/slurmdbd.pid
```

Permissions: **`slurmdbd.conf` must be 600 and owned by `SlurmUser`**
```bash
sudo chown satej:satej slurmdbd.conf
sudo chmod 600 slurmdbd.conf
sudo touch /var/log/slurmdbd.log && sudo chown satej:satej /var/log/slurmdbd.log
```

> In the history the `chmod 600` was applied to `slurmdbd.service` first, which is the wrong file; it belongs on `slurmdbd.conf`.

Add these lines to `slurm.conf`:
```ini
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=controller
```

Start it:
```bash
sudo ln -s /slurmdir/slurm-20.11.9/etc/slurmdbd.service /usr/lib/systemd/system/slurmdbd.service
sudo systemctl daemon-reload
sudo systemctl enable --now slurmdbd
sudo systemctl restart slurmctld
sacctmgr add cluster cluster
sacctmgr show cluster
```

---

## 15. Firewall (if enabled)

| Port | Service     |
|------|-------------|
| 6817 | slurmctld   |
| 6818 | slurmd      |
| 6819 | slurmdbd    |

```bash
sudo ufw allow 6817:6819/tcp
```

---

## Quick troubleshooting

| Symptom | Likely cause / fix |
|---------|--------------------|
| `slurmctld`/`slurmd` fails right after start | Run with `-D -vvv`; usually spool or log directory ownership |
| `sinfo` can't find config | Config is not in default path; run `export SLURM_CONF=/slurmdir/slurm-20.11.9/etc/slurm.conf` or copy it to `/etc/slurm/` |
| `Invalid user id` / munge errors | Munge key differs between nodes, wrong permissions, or clocks out of sync |
| Node stays `down` | Hostname / `NodeName` mismatch, wrong CPU/RAM values, or `slurmd` not running |
| `Unable to contact slurm controller` | Check `/etc/hosts`, firewall ports, and that `slurmctld` is running |
| slurmdbd won't start | `slurmdbd.conf` not mode 600 or not owned by `SlurmUser`; MariaDB not running |

## Notes

- Slurm 20.11 is old and no longer supported. For anything beyond a lab/course setup, use a current release.
- Running daemons as a normal user (`satej`) is fine for a lab. For production, use a dedicated `slurm` user with the same UID/GID on all nodes.
