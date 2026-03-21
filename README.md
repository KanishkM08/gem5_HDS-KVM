# Setting up gem5 v23.0.0.1 with KVM CPU Switching on Windows (WSL2) / Linux

This guide explains how to run gem5 v23.0.0.1 with KVM acceleration and CPU switching using docker on Linux or on Windows using WSL.
The setup enables:

* KVM acceleration (X86KvmCPU)
* Switching to timing CPUs during simulation
* Running full-system simulations inside Docker
* Access to hardware performance counters

## 1. Verify Hardware Virtualization (VT-x / VT-d)

Ensure virtualization is enabled in BIOS.
- Then on Windows, (need different steps to check it on Linux)
- Press Ctrl + Shift + Esc to open Task Manager
- Go to Performance → CPU
- In the details section at the bottom right, check Virtualization is enabled


<img width="468" height="190" alt="image" src="https://github.com/user-attachments/assets/05e25e6b-f0f8-44bd-9b3a-1bda7ca7ac49" />


## 2. Install WSL2 (For Linux you can skip this step)

### Install Ubuntu 22.04 on WSL2.
Expected version:
```bash
PRETTY_NAME="Ubuntu 22.04.2 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.2 LTS (Jammy Jellyfish)"
```

### Configure WSL for Nested Virtualization
Shutdown WSL:
```bash
wsl --shutdown
```

Edit: C:\Users\<YourUsername>\.wslconfig
Add:
```bash
[wsl2]
memory=16GB
swap=64GB
processors=16
nestedVirtualization=true
```

Restart WSL.

### Verify KVM Availability inside WSL

Install cpu-checker
```bash
sudo apt install cpu-checker
```

Run kvm-ok

Expected output:
```bash
kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
```

Verify KVM Device
```bash
ls /dev/kvm
```

Enable Performance Counters
```bash
echo "-1" | sudo tee /proc/sys/kernel/perf_event_paranoid
```

Fix KVM Permissions
```bash
sudo chown root:kvm /dev/kvm
sudo chmod 660 /dev/kvm
```

Add User to KVM Group
```bash
sudo usermod -aG kvm $USER
```

Restart WSL:
```bash
wsl --shutdown
wsl
```

Verify groups
```bash
groups $USER
```

## 3. Install Docker

### Only for WSL setup
Enable systemd
```bash
sudo vi /etc/wsl.conf
```
Add:
```bash
[boot]
systemd=true
```
Restart WSL.


### Install Dependencies
```bash
# Move to Linux home first
cd ~

sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
# Add Docker GPG Key
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
 | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg
# If Key Download Fails
cd ~
wget https://download.docker.com/linux/ubuntu/gpg -O docker_key.asc
head -n 5 docker_key.asc

# Expected output:

-----BEGIN PGP PUBLIC KEY BLOCK-----

# If it looks correct, dearmor and save it
cat docker_key.asc | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Repository to Apt sources
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu jammy stable" | sudo tee /etc/apt/sources.list.d/docker.list
```

### Install Docker
```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Start Docker service and give yourself permission to use it
```bash
sudo service docker start
sudo usermod -aG docker $USER
# logout of wsl and login
Wsl --shutdown
wsl
```

## 4. Run Docker with KVM Enabled
```bash
docker run --rm \
  --privileged \
  --cap-add=SYS_ADMIN \
  --cap-add=SYS_PTRACE \
  --device=/dev/kvm \
  -e TERM=xterm \
  -e TZ=$(cat /etc/timezone) \
  -v /home/ananthak/project/gem5:/gem5 \
  -v /home/ananthak/project/parsec-benchmark:/parsec-benchmark \
  -v /home/ananthak/gem5_cache:/root/.cache/gem5 \
  -v /etc/timezone:/etc/timezone:ro \
  -v /etc/localtime:/etc/localtime:ro \
  -w /gem5 \
  -e LANG=C.UTF-8 \
  -e LC_ALL=C.UTF-8 \
  -it ghcr.io/gem5/ubuntu-22.04_all-dependencies:v23-0
```

### Install Python Libraries (Inside Docker)
```bash
apt-get update
apt-get install -y libpython3.11
```

## 5. Build & Run Gem5
### Build Gem5
```bash
scons build/X86_MESI_TWO_LEVEL/gem5.opt \
 --default=X86 \
 PROTOCOL=MESI_Two_Level \
 -j$(nproc)
```

While building GEM5 check for the following output. If you dont see this line while building (appears within first 100 lines of build) then KVM is not getting used.
```bash
Checking for C header file linux/kvm.h... Yes
```

### Run Gem5
```bash
./build/X86_OLD_SIM_MESI_TWO_LEVEL/gem5.opt \
configs/deprecated/example/fskvm.py \
--cpu-type=X86KvmCPU \
--num-cpus=4 \
--ruby \
--network=garnet \
--topology=Mesh_XY \
--mesh-rows=2 \
--num-dirs=4 \
--num-l2caches=4 \
--kernel=./vmlinux-5.4.49 \
--disk-image=./parsec.img \
--script=./helloworld.sh
```

or 

```bash
./build/X86_OLD_SIM_MESI_TWO_LEVEL/gem5.opt \
configs/deprecated/example/fskvm.py \
--cpu-type=X86KvmCPU \
--num-cpus=4 \
--ruby \
--network=garnet \
--topology=Mesh_XY \
--mesh-rows=2 \
--num-dirs=4 \
--num-l2caches=4 \
--kernel=./vmlinux-5.4.49 \
--disk-image=./parsec.img \
--script=./run_parsec.sh
```

Check the output and see that we can connect to the simulated system at port 3456. See next session on how to connect to the simulated system to check the serial logs
```bash
src/base/statistics.hh:279: warn: One of the stats is a legacy stat. Legacy stat is a stat that does not belong to any statistics::Group. Legacy stat is deprecated.
      0: system.pc.south_bridge.cmos.rtc: Real-time clock set to Sun Jan  1 00:00:00 2012
system.pc.com_1.device: Listening for connections on port 3456
src/base/statistics.hh:279: warn: One of the stats is a legacy stat. Legacy stat is a stat that does not belong to any statistics::Group. Legacy stat is deprecated.
```

### Check serial output
On another terminal check for running docker instance
```bash
docker ps
CONTAINER ID   IMAGE                                              COMMAND       CREATED              STATUS              PORTS     NAMES
ef9cddb904e7   ghcr.io/gem5/ubuntu-22.04_all-dependencies:v23-0   "/bin/bash"   About a minute ago   Up About a minute             zen_booth

# Connect to the existing docker container
docker exec -it zen_booth /bin/bash
```

Go to util/term and build m5term
build m5term application to check serial logs
```bash
cd util/term
make
```

Check the serial output

```bash
m5term localhost 3456

# Expected ouput
[  OK  ] Reached target Remote File Systems.
random: systemd: uninitialized urandom read (16 bytes read)
systemd[1]: Created slice User and Session Slice.
[  OK  ] Created slice User and Session Slice.
random: systemd: uninitialized urandom read (16 bytes read)
systemd[1]: Started Forward Password Requests to Wall Directory Watch.
[  OK  ] Started Forward Password Requests to Wall Directory Watch.
systemd[1]: Reached target User and Group Name Lookups.
[  OK  ] Reached target User and Group Name Lookups.

...

Ubuntu 18.04.2 LTS gem5-host ttyS0

gem5-host login: root (automatic login)

Welcome to Ubuntu 18.04.2 LTS (GNU/Linux 5.4.49 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

New release '20.04.2 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

Boot done
in_4.txt
Starting ROI
PARSEC Benchmark Suite Version 3.0-beta-20150206
[HOOKS] PARSEC Hooks Version 1.2
Num of Options: 4
Num of Runs: 100
Size of data: 160
[HOOKS] Entering ROI
[HOOKS] Leaving ROI
[HOOKS] Total time spent in ROI: 0.001s
[HOOKS] Terminating
Ending ROI
root@ef9cddb904e7:/gem5/util/term#
```
