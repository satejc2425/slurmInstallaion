# Slurm 20.11.9 Install Guide: 1 Controller + 1 Compute Node

This guide reproduces your working cluster, with the problems found in your histories fixed.

## Before you start

- Two Ubuntu 24.04 machines that can reach each other and the internet.
- The same user `satej` (with `sudo`) on both machines.
- Controller IP `192.168.76.33`, compute IP `192.168.76.108`. Confirm with `ip -br a` and change the IPs below if yours differ.
- Both machines run `slurmd` (the controller is also a compute node), exactly like your current setup.
- Slurm is installed from source into `/slurmdir/slurm-20.11.9`.
- Accounting (`slurmdbd`) is **not** included. It was never verified in your history.

Legend: **[C]** = controller only, **[N]** = compute node only, **[BOTH]** = run on both machines.

Run the steps in order. After each step, stop and fix any error before continuing.

---

## Step 1: Hostnames and `/etc/hosts`

**[C]**
```bash
sudo hostnamectl set-hostname controller
```

**[N]**
```bash
sudo hostnamectl set-hostname compute1
```

**[BOTH]** (open a new terminal afterwards so the prompt shows the new name)
```bash
sudo sed -i '/^127\.0\.1\.1/d' /etc/hosts
echo "192.168.76.33  controller" | sudo tee -a /etc/hosts
echo "192.168.76.108 compute1"   | sudo tee -a /etc/hosts
```

Check:
```bash
ping -c 2 controller
ping -c 2 compute1
```
Both must reply. If not, fix the network before going on.

---

## Step 2: Fix apt sources (only if `apt update` fails because of "cdrom")

**[BOTH]**
```bash
sudo apt update
```
If this shows an error about `cdrom`, comment out that line, then run `apt update` again:
```bash
sudo nano /etc/apt/sources.list
sudo apt update
```

---

## Step 3: Install packages and set up time sync

**[BOTH]**
```bash
sudo apt install -y build-essential munge libmunge-dev libmunge2 \
    libmysqlclient-dev libssl-dev libpam0g-dev libnuma-dev perl wget \
    openssh-server chrony
sudo systemctl enable --now chrony
```

---

## Step 4: Munge key (must be identical on both machines)

**[C]** Send the controller's key to the compute node:
```bash
sudo systemctl enable --now munge
sudo cat /etc/munge/munge.key | ssh satej@compute1 'cat > /tmp/munge.key'
```
Enter the `satej` password for `compute1` when asked.

**[N]** Install the key and start munge:
```bash
sudo mv /tmp/munge.key /etc/munge/munge.key
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
sudo systemctl enable munge
sudo systemctl restart munge
```

**[C]** Test:
```bash
munge -n | ssh satej@compute1 unmunge
```
The output must contain `STATUS:          Success (0)`. If it does not, the keys differ or the clocks are out of sync. Do not continue until it succeeds.

---

## Step 5: Download, build and install Slurm

**[BOTH]**
```bash
sudo mkdir -p /slurmdir
sudo chown satej:satej /slurmdir
cd /slurmdir
wget https://download.schedmd.com/slurm/slurm-20.11.9.tar.bz2
tar -xvjf slurm-20.11.9.tar.bz2
cd slurm-20.11.9
./configure --prefix=/slurmdir/slurm-20.11.9/
make -j"$(nproc)"
make install
```

---

## Step 6: Add Slurm to PATH (permanent)

**[BOTH]**
```bash
cat >> ~/.bashrc << 'EOF'
export PATH="/slurmdir/slurm-20.11.9/bin:/slurmdir/slurm-20.11.9/sbin:$PATH"
export LD_LIBRARY_PATH="/slurmdir/slurm-20.11.9/lib:$LD_LIBRARY_PATH"
export SLURM_CONF="/slurmdir/slurm-20.11.9/etc/slurm.conf"
EOF
source ~/.bashrc
```

Check:
```bash
which sinfo slurmd
```
Both paths must start with `/slurmdir/slurm-20.11.9/`.

---

## Step 7: Check the hardware of each machine

**[BOTH]**
```bash
slurmd -C
```

Your config in Step 9 assumes this output on **both** machines:

- `CPUs=2 Boards=1 SocketsPerBoard=2 CoresPerSocket=1 ThreadsPerCore=1`
- `RealMemory` of at least `3000`

If a machine reports different CPU values, edit the `NodeName` line in Step 9 to match. If `RealMemory` is below 3000 on either machine, use a value below the smaller of the two numbers.

---

## Step 8: Create spool and log directories

**[C]**
```bash
sudo mkdir -p /var/spool/slurm/ctld
sudo chown satej:satej /var/spool/slurm/ctld
sudo touch /var/log/slurmctld.log
sudo chown satej:satej /var/log/slurmctld.log
```

**[BOTH]**
```bash
sudo mkdir -p /var/spool/slurm/d
sudo chown satej:satej /var/spool/slurm/d
sudo chmod 755 /var/spool/slurm /var/spool/slurm/d
sudo touch /var/log/slurmd.log
sudo chown satej:satej /var/log/slurmd.log
sudo chmod 644 /var/log/slurmd.log
```

---

## Step 9: Create `slurm.conf` and copy it to the compute node

**[C]** Write the config (this replaces the example file):
```bash
cat > /slurmdir/slurm-20.11.9/etc/slurm.conf << 'EOF'
ClusterName=linux
ControlMachine=controller
SlurmUser=satej
SlurmctldPort=6817
SlurmdPort=6818
AuthType=auth/munge
StateSaveLocation=/var/spool/slurm/ctld
SlurmdSpoolDir=/var/spool/slurm/d
SwitchType=switch/none
MpiDefault=none
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmdPidFile=/var/run/slurmd.pid
ProctrackType=proctrack/pgid
ReturnToService=2
SlurmctldTimeout=300
SlurmdTimeout=300
InactiveLimit=0
MinJobAge=300
KillWait=30
Waittime=0
SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core
SlurmctldDebug=info
SlurmctldLogFile=/var/log/slurmctld.log
SlurmdDebug=info
SlurmdLogFile=/var/log/slurmd.log
JobCompType=jobcomp/none

NodeName=controller,compute1 CPUs=2 Boards=1 SocketsPerBoard=2 CoresPerSocket=1 ThreadsPerCore=1 RealMemory=3000 State=UNKNOWN
PartitionName=debug Nodes=ALL Default=YES MaxTime=INFINITE State=UP
EOF
```

Copy it to the compute node:
```bash
scp /slurmdir/slurm-20.11.9/etc/slurm.conf satej@compute1:/slurmdir/slurm-20.11.9/etc/slurm.conf
```

**[BOTH]** Verify both files are identical (the two hashes must match):
```bash
md5sum /slurmdir/slurm-20.11.9/etc/slurm.conf
```

Notes:

- `ReturnToService=2` makes a node come back automatically after a reboot. Your old setting (`0`) is why `compute1` stayed `down`.
- `RealMemory=3000` is deliberately a little below the real value, so Slurm never rejects the node for having less memory than configured.

---

## Step 10: Install and start the services

**[C]**
```bash
sudo ln -sf /slurmdir/slurm-20.11.9/etc/slurmctld.service /usr/lib/systemd/system/slurmctld.service
sudo ln -sf /slurmdir/slurm-20.11.9/etc/slurmd.service    /usr/lib/systemd/system/slurmd.service
sudo systemctl daemon-reload
sudo systemctl enable --now slurmctld
sudo systemctl enable --now slurmd
```

**[N]**
```bash
sudo ln -sf /slurmdir/slurm-20.11.9/etc/slurmd.service /usr/lib/systemd/system/slurmd.service
sudo systemctl daemon-reload
sudo systemctl enable --now slurmd
```

**[BOTH]** Open the Slurm ports in the firewall (harmless if `ufw` is inactive):
```bash
sudo ufw allow 6817:6819/tcp
```

Check the services:
```bash
sudo systemctl status slurmd --no-pager
```
**[C]** also:
```bash
sudo systemctl status slurmctld --no-pager
```
Each must show `active (running)`.

---

## Step 11: Verify the cluster

**[C]**
```bash
sinfo
```
Expected: both `controller` and `compute1` in state `idle`. The first minute may show `unk*` or `idle*`; wait 30 seconds and run `sinfo` again.

If `compute1` is `down` or `drain`, run:
```bash
scontrol show node compute1 | grep -i reason
sudo env "PATH=$PATH" scontrol update NodeName=compute1 State=RESUME
```

Test a job on both nodes:
```bash
srun -N2 hostname
```
Expected output: `controller` and `compute1`.

Test a batch job:
```bash
cat > ~/test.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=test
#SBATCH --output=test_%j.out
hostname
sleep 10
EOF
sbatch ~/test.sh
squeue
```

---

## Step 12: Reboot test (optional but recommended)

Reboot `compute1`, wait two minutes, then on the controller run:
```bash
sinfo
```
`compute1` should return to `idle` by itself.

---

## Troubleshooting

| Symptom | What to do |
|---------|-----------|
| A service fails to start | Run it in the foreground to see the real error: `slurmctld -D -vvv` or `slurmd -D -vvv` |
| `Invalid user id` or munge errors | Munge key differs, key has wrong permissions, or clocks are out of sync. Repeat Step 4 |
| `sinfo`: command not found | Run `source ~/.bashrc` (Step 6) |
| `Unable to contact slurm controller` | Check `/etc/hosts`, the firewall, and that `slurmctld` is running |
| Node `drain` or `inval` | Run `scontrol show node <name> \| grep -i reason`. Usually the CPU or memory in `slurm.conf` does not match `slurmd -C` (Step 7) |
| Node `down` after reboot | Resume it (Step 11). With `ReturnToService=2` this should not happen again |
| Config edited | Copy it to the other node, check `md5sum`, then restart `slurmctld` and `slurmd` |

## Not covered

- **Accounting (`slurmdbd` + MariaDB):** not verified in your setup.
- **More compute nodes:** repeat Steps 1 to 8 on each one, add them to `/etc/hosts`, add them to the `NodeName` line, and copy the same `munge.key` and `slurm.conf`.
- **Version:** Slurm 20.11 is old and unsupported. It is fine for a course lab, not for production.
