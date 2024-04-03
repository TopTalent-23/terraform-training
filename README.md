# DevOps Training
This repository serves as a platform for my personal DevOps training, showcasing my skills in scripting and DevOps tools. Its primary objectives are:

1. **Self-Training:** To train myself in DevOps tasks.
2. **Skill Demonstration:** To demonstrate my capabilities in DevOps tools and scripting to potential employers and collaborators.
3. **Personal Future Reference:** This repository will be used by me for future reference and for future projects.

## Issue Description
1. **Servers Updates:** Simplify the update process for two of my personal servers and AWS EC2 server.
2. **Automated AWS EC2 Setup:** Implement automated setup, and destruction of Ubuntu servers on AWS EC2.
3. **Remote Server Monitoring:** Monitor system status from all servers remotely, no direct access needed.
4. **Simple Execution:** Execute all tasks with a simple command-line interface.
5. **Keep it Simple:** I am aiming to make the scripts and codes as simple and readable as possible.

## Solution
Since I try to keep it simple, I aim to maintain the scripts and code with minimal dependencies possible. Additionally, it's important that they remain readable without excessive commenting. I will thoroughly explain the solution on each section.

Throughout this README, I will reference three servers. Please refer to the table below. I will refer to the servers by their names.

| Name| Hostname|os| Comments|
| ----------- | ----------- |---------|-------|
| cloud| cloud.ss.fish|Ubuntu| My personal cloud server|
| ss| ss.fish|OpenBSD|My personal OpenBSD server|
| ec2| | Ubuntu|This server will be created on an AWS EC2 instance|

### Servers Updates
To update all servers, I'll employ both **Ansible** and **Python** scripts. The **Python** script is crucial for converting the CSV file into an INI format for the **Ansible** inventory. Since servers are listed in a CSV file for other automation purposes, it's necessary.

**Ansible Playbooks:**
1. Update and check if reboot is needed on cloud server: [ansible-playbook/update_upgrade_cloud.yml](ansible-playbook/update_upgrade_cloud.yml)
2. Update all the package on ss server: [ansible-playbook/update_upgrade_ss.yml](ansible-playbook/update_upgrade_ss.yml)
3. Update and check if reboot is needed on ec2 server: [ansible-playbook/update_upgrade_ec2.yml](ansible-playbook/update_upgrade_ec2.yml)

On all three servers, the playbook will run as the user named 'ansible,' which I created for security purposes. This user can execute specific commands without a password when using sudo and doas, as I have manually configured.

Please note that on both the cloud and ec2 server playbooks, I do not use the apt module. Instead, I manually run the command in the Ansible playbook due to [this GitHub issue #51663](https://github.com/ansible/ansible/issues/51663). Additionally, in both server playbooks, it will check if a reboot is needed after every update.

The inventory is located here: [ansible-playbook/hosts.csv](ansible-playbook/hosts.csv.example) and will be converted by the script [csv_to_ini.py](csv_to_ini.py) to [ansible-playbook/hosts](ansible-playbook/hosts). This script will run automatically when using the `run.sh` command and can also be executed manually. Further details on the `run.sh` command will be explained on further section.


### Automated AWS EC2 Setup:
To create an EC2 instance in **AWS**, I utilized **Terraform** and **aws-cli**. **aws-cli** is only needed for authentication and connection to my **AWS** account. To make this work, I created three files: `provider.tf`, `main.tf`, and `output.tf`. Let me explain what each of these files does:

1. **[terraform/provider.tf](terraform/provider.tf)**
    - Specifies `hashicorp/aws` as the provider so Terraform will work with AWS.
    - Sets the region to Sydney (`ap-southeast-2`).
    - Sets up the VPC, Internet Gateway, and creates the Security Group to allow SSH connection.
      - All of this is needed for it to connect to the internet and for SSH to work.

2. **[terraform/main.tf](terraform/main.tf)**
    - Sets the EC2 AMI to `ami-09c8d5d747253fb7a` (Ubuntu 22.04 LTS, x86), sets the instance type to `t2.micro`, and names it "DevOps-Training".
    - Sets the security group 'SSHAcess', which is created in the `provider.tf` file.
    - Creates the user 'ansible', adds my public key, edits the sudoers file, and installs the Python `psutil` module.
      - My public key is visible in the file, but since it's a public key, it's safe to share publicly.
      - I configured it so that `apt` and `apt-get` commands can run passwordless with sudo so Ansible will work.
      - Installs Python `psutil` so that the stats monitoring script will work.

3. **[terraform/output.tf](terraform/output.tf)**
    - Provides the instance IP as output after it successfully creates the instance. I do not plan to use a fixed IP to save costs, so this is necessary for other automation scripts.


### Remote Server Monitoring:
To remotely monitor system statistics on all servers, I'll utilize the psutil **Python** library for reading system statistics and paramiko for establishing SSH connections remotely.

Originally, I planned to create the system reader from scratch using available Unix tools and parsing, but this approach would complicate things, especially since I intend to use this script on three different platforms (Linux, OpenBSD, and Darwin [MacOS]). Instead, it's simpler to use the psutil module since it already works across all these platforms.

I prefer to start most of my scripts with a shebang to env, like below:
```
#!/usr/bin/env python3
```
This way, I can run the script without specifying python3 or python in front of it. Using env also ensures that the OS will pick the correct Python version/program, and since Python isn't always located in the same path on every platform, this approach is more robust.

1. **[sys_reading.py](sys_reading.py)**

This script was created with the assistance of the [psutil documentation](https://psutil.readthedocs.io/en/latest/).  It displays various system metrics, including Hostname, Platform, Uptime, CPU usage, Memory usage, Network Card IPs, Public IP, and Root disk usage.

**Using Psutil**
- **Cpu Reading:** Utilizes `psutil.cpu_percent()` to obtain CPU usage in percentage with half-second readings for quick updates.
- **Memory usage:** Retrieves memory usage information using `psutil.virtual_memory()`.
- **Root disk usage:** Obtains disk usage for the root partition using `psutil.disk_usage()`.

**Using Unix tools**
- **Uptime:** Parses system uptime using the typical Unix uptime command, processed with awk and sed. Although I couldn't find a straightforward way to use `psutil.boot_time()`, I may revisit this approach in the future.
- **Network Card IPs:** Extracts IP addresses from network interfaces using the typical Unix ifconfig command and parses the output with awk.
- **Public IP:** Retrieves the public IP address by querying the ifconfig.me website using curl.
- **Hostname:** just to keep this consistant I'll be using the standard Unix `hostname` command.

**Using Platform Library**
- **Platform:** Utilizes the standard Python platform library to obtain platform information. This could prove essential for future updates to the script, especially if different commands are required for specific platforms.

This script is currently incomplete and somewhat messy. In the future, I plan to improve it by adding the following readings and features:

- Disk usage statistics for all storage partitions.
- Last system update information.
- Detection of security vulnerabilities in the kernel and installed packages.
- Implementation of email alerts for high resource usage.

However, if I end up adding too many features, it might be more efficient to use other specialized tools that can handle these tasks more effectively than a custom script. Nevertheless, this script serves as a valuable practice exercise for me to improve my skills and expanding my knowledge in the realm of DevOps.

2. **[sys_reading_remote.py](sys_reading_remote.py)**

The script [sys_reading.py](sys_reading.py) lacks the capability to SSH into servers and parse CSV files. I do not intend to add this functionality as it would complicate the script further as its already look messy. Additionally, this script is separated because it will execute sys_reading.py on the hosts themselves. In the future, I may consider combining both scripts.

This script parses [ansible-playbook/hosts.csv](ansible-playbook/hosts.csv.example) and SSHs into each host listed in it to execute the `sys_reading.py` script locally.

Initially, I intended to use the pandas library for parsing the CSV file. However, I discovered that Python's built-in csv module functions well for small datasets and offers cleaner code. As a result, I opted to switch to the csv library instead of pandas.

### Simple Execution:

To streamline operations, all the functionalities described above are connected using a single shell script [run.sh](run.sh).

This shell script also utilizes a shebang to env, similar to all the Python scripts above:

```
#!/usr/bin/env sh
```

The script accepts four arguments and performs five actions: 'create', 'destroy', 'stats', 'update', and runs everything when no argument is given. When using the argument, it's possible to specify a character like 'c', 'd', 's', or 'u'.

**create|c**
- Executes `terraform apply` with the `-auto-approve` argument.
- Adds instance IPs to [hosts.csv](ansible-playbook/hosts.csv.example).

**destroy|d**
- Executes `terraform destroy` with the `-auto-approve` argument.
- Removes instance IPs from [hosts.csv](ansible-playbook/hosts.csv.example).

**stats|s**
- Executes `sys_reading_remote.py`, which runs `sys_reading.py` locally on the server.

**update|u**
- Runs `csv_to_ini.py` to convert [hosts.csv](ansible-playbook/hosts.csv.example) to [hosts](ansible-playbook/hosts.example).
- Reads [hosts.csv](ansible-playbook/hosts.csv.example) and runs ansible-playbook on each host to update it.
- Ansible-playbook commands will use the [hosts](ansible-playbook/hosts.example) file as inventory and run with the argument `ANSIBLE_HOST_KEY_CHECKING=False` to disable fingerprint checks on the first SSH connection.

**No Arguments**
When running [run.sh](run.sh) with no argument, it will execute the script in the following order:
- create
- update
- stats
- destroy

Details on usage for each section are provided below.

## Usage
In this section I will show how I use this this repository, and the command [run.sh](run.sh) to automate and simplify this tasks.


### Updates With Ansible

On the directory [ansible-playbook](ansible-playbook), it has all three of the servers playbook and the inventory file [hosts](hosts).



To upgrade cloud I will just run the following command on the [ansible-playbook](ansible-playbook)
```
(myenv) [ansible-playbook] % :ansible-playbook -i hosts update_upgrade_cloud.yml

PLAY [Update and upgrade cloud server] **************************************************************************

...

<details>
  <summary>Click me</summary>

```
(myenv) [devops-training] % :./run.sh u
-------------------------------------------------------
Updating all of my servers including the new EC2 server
-------------------------------------------------------
Convert hosts.csv file to hosts (ini)...
Update hosts...
Updating cloud ...

PLAY [Update and upgrade cloud server] **************************************************************************

TASK [Gathering Facts] ******************************************************************************************
ok: [cloud.ss.fish]

TASK [Update all package] ***************************************************************************************
changed: [cloud.ss.fish]

TASK [Display apt update] ***************************************************************************************
ok: [cloud.ss.fish] => {
    "apt_output_update.stdout_lines": [
        "Get:1 http://security.ubuntu.com/ubuntu jammy-security InRelease [110 kB]",
        "Hit:2 http://archive.ubuntu.com/ubuntu jammy InRelease",
        "Get:3 http://archive.ubuntu.com/ubuntu jammy-updates InRelease [119 kB]",
        "Get:4 http://security.ubuntu.com/ubuntu jammy-security/main amd64 Packages [1,303 kB]",
        "Hit:5 http://archive.ubuntu.com/ubuntu jammy-backports InRelease",
        "Get:6 http://security.ubuntu.com/ubuntu jammy-security/main i386 Packages [436 kB]",
        "Get:7 http://security.ubuntu.com/ubuntu jammy-security/universe amd64 Packages [852 kB]",
        "Get:8 http://security.ubuntu.com/ubuntu jammy-security/universe i386 Packages [599 kB]",
        "Get:9 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 Packages [1,519 kB]",
        "Get:10 http://archive.ubuntu.com/ubuntu jammy-updates/main i386 Packages [602 kB]",
        "Get:11 http://archive.ubuntu.com/ubuntu jammy-updates/restricted amd64 Packages [1,644 kB]",
        "Get:12 http://archive.ubuntu.com/ubuntu jammy-updates/restricted i386 Packages [35.3 kB]",
        "Get:13 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 Packages [1,060 kB]",
        "Get:14 http://archive.ubuntu.com/ubuntu jammy-updates/universe i386 Packages [698 kB]",
        "Fetched 8,977 kB in 4s (2,408 kB/s)",
        "Reading package lists..."
    ]
}

TASK [Upgrade all package] **************************************************************************************
changed: [cloud.ss.fish]

TASK [Display apt upgrade] **************************************************************************************
ok: [cloud.ss.fish] => {
    "apt_output_upgrade.stdout_lines": [
        "Reading package lists...",
        "Building dependency tree...",
        "Reading state information...",
        "Calculating upgrade...",
        "The following packages have been kept back:",
        "  ethtool firmware-sof-signed",
        "0 to upgrade, 0 to newly install, 0 to remove and 2 not to upgrade."
    ]
}

TASK [Check if reboot is needed] ********************************************************************************
fatal: [cloud.ss.fish]: FAILED! => {"changed": false, "cmd": "needs-restart", "msg": "[Errno 2] No such file or directory: b'needs-restart'", "rc": 2, "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}
...ignoring

TASK [Display reboot required status] ***************************************************************************
ok: [cloud.ss.fish] => {
    "msg": "Reboot is needed: no"
}

PLAY RECAP ******************************************************************************************************
cloud.ss.fish              : ok=7    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=1

Updating ss ...

PLAY [Update OpenBSD packages] **********************************************************************************

TASK [Gathering Facts] ******************************************************************************************
[WARNING]: Platform openbsd on host ss.fish is using the discovered Python interpreter at
/usr/local/bin/python3.10, but future installation of another Python interpreter could change the meaning of
that path. See https://docs.ansible.com/ansible-core/2.16/reference_appendices/interpreter_discovery.html for
more information.
ok: [ss.fish]

TASK [Update packages] ******************************************************************************************
changed: [ss.fish]

PLAY RECAP ******************************************************************************************************
ss.fish                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

Updating ec2 ...

PLAY [Update and upgrade cloud server] **************************************************************************

TASK [Gathering Facts] ******************************************************************************************
ok: [13.211.209.95]

TASK [Update all package] ***************************************************************************************
changed: [13.211.209.95]

TASK [Display apt update] ***************************************************************************************
ok: [13.211.209.95] => {
    "apt_output_update.stdout_lines": [
        "Hit:1 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy InRelease",
        "Get:2 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates InRelease [119 kB]",
        "Get:3 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-backports InRelease [109 kB]",
        "Get:4 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy/universe amd64 Packages [14.1 MB]",
        "Get:5 http://security.ubuntu.com/ubuntu jammy-security InRelease [110 kB]",
        "Get:6 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy/universe Translation-en [5652 kB]",
        "Get:7 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy/universe amd64 c-n-f Metadata [286 kB]",
        "Get:8 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy/multiverse amd64 Packages [217 kB]",
        "Get:9 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy/multiverse Translation-en [112 kB]",
        "Get:10 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy/multiverse amd64 c-n-f Metadata [8372 B]",
        "Get:11 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 Packages [1519 kB]",
        "Get:12 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main Translation-en [293 kB]",
        "Get:13 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/restricted amd64 Packages [1644 kB]",
        "Get:14 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/restricted Translation-en [274 kB]",
        "Get:15 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/universe amd64 Packages [1060 kB]",
        "Get:16 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/universe Translation-en [241 kB]",
        "Get:17 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/universe amd64 c-n-f Metadata [22.1 kB]",
        "Get:18 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/multiverse amd64 Packages [49.6 kB]",
        "Get:19 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/multiverse Translation-en [12.0 kB]",
        "Get:20 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/multiverse amd64 c-n-f Metadata [472 B]",
        "Get:21 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-backports/main amd64 Packages [67.1 kB]",
        "Get:22 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-backports/main Translation-en [11.0 kB]",
        "Get:23 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-backports/main amd64 c-n-f Metadata [388 B]",
        "Get:24 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-backports/restricted amd64 c-n-f Metadata [116 B]",
        "Get:25 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-backports/universe amd64 Packages [28.4 kB]",
        "Get:26 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-backports/universe Translation-en [16.2 kB]",
        "Get:27 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-backports/universe amd64 c-n-f Metadata [644 B]",
        "Get:28 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-backports/multiverse amd64 c-n-f Metadata [116 B]",
        "Get:29 http://security.ubuntu.com/ubuntu jammy-security/main amd64 Packages [1303 kB]",
        "Get:30 http://security.ubuntu.com/ubuntu jammy-security/main Translation-en [233 kB]",
        "Get:31 http://security.ubuntu.com/ubuntu jammy-security/restricted amd64 Packages [1616 kB]",
        "Get:32 http://security.ubuntu.com/ubuntu jammy-security/restricted Translation-en [271 kB]",
        "Get:33 http://security.ubuntu.com/ubuntu jammy-security/universe amd64 Packages [852 kB]",
        "Get:34 http://security.ubuntu.com/ubuntu jammy-security/universe Translation-en [163 kB]",
        "Get:35 http://security.ubuntu.com/ubuntu jammy-security/universe amd64 c-n-f Metadata [16.8 kB]",
        "Get:36 http://security.ubuntu.com/ubuntu jammy-security/multiverse amd64 Packages [37.1 kB]",
        "Get:37 http://security.ubuntu.com/ubuntu jammy-security/multiverse Translation-en [7476 B]",
        "Get:38 http://security.ubuntu.com/ubuntu jammy-security/multiverse amd64 c-n-f Metadata [260 B]",
        "Fetched 30.4 MB in 6s (5309 kB/s)",
        "Reading package lists..."
    ]
}

TASK [Upgrade all package] **************************************************************************************
changed: [13.211.209.95]

TASK [Display apt upgrade] **************************************************************************************
ok: [13.211.209.95] => {
    "apt_output_upgrade.stdout_lines": [
        "Reading package lists...",
        "Building dependency tree...",
        "Reading state information...",
        "Calculating upgrade...",
        "The following packages have been kept back:",
        "  linux-aws linux-headers-aws linux-image-aws ubuntu-advantage-tools",
        "  ubuntu-pro-client-l10n",
        "The following packages will be upgraded:",
        "  apt apt-utils bash bsdextrautils bsdutils coreutils curl dpkg eject ethtool",
        "  fdisk libapt-pkg6.0 libblkid1 libcurl3-gnutls libcurl4 libexpat1 libfdisk1",
        "  libgpgme11 libldap-2.5-0 libldap-common libmount1 libsmartcols1 libuuid1",
        "  mount python3-cryptography python3-update-manager snapd update-manager-core",
        "  update-notifier-common util-linux uuid-runtime vim vim-common vim-runtime",
        "  vim-tiny xxd",
        "36 upgraded, 0 newly installed, 0 to remove and 5 not upgraded.",
        "Need to get 43.9 MB of archives.",
        "After this operation, 1584 kB disk space will be freed.",
        "Get:1 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 bash amd64 5.1-6ubuntu1.1 [769 kB]",
        "Get:2 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 bsdutils amd64 1:2.37.2-4ubuntu3.3 [81.5 kB]",
        "Get:3 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 coreutils amd64 8.32-4.1ubuntu1.2 [1437 kB]",
        "Get:4 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 libapt-pkg6.0 amd64 2.4.12 [912 kB]",
        "Get:5 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 dpkg amd64 1.21.1ubuntu2.3 [1239 kB]",
        "Get:6 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 util-linux amd64 2.37.2-4ubuntu3.3 [1063 kB]",
        "Get:7 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 apt amd64 2.4.12 [1363 kB]",
        "Get:8 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 apt-utils amd64 2.4.12 [211 kB]",
        "Get:9 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 mount amd64 2.37.2-4ubuntu3.3 [114 kB]",
        "Get:10 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 libsmartcols1 amd64 2.37.2-4ubuntu3.3 [51.5 kB]",
        "Get:11 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 libuuid1 amd64 2.37.2-4ubuntu3.3 [24.4 kB]",
        "Get:12 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 uuid-runtime amd64 2.37.2-4ubuntu3.3 [32.0 kB]",
        "Get:13 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 update-manager-core all 1:22.04.19 [11.6 kB]",
        "Get:14 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 python3-update-manager all 1:22.04.19 [39.1 kB]",
        "Get:15 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 update-notifier-common all 3.192.54.8 [185 kB]",
        "Get:16 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 libblkid1 amd64 2.37.2-4ubuntu3.3 [104 kB]",
        "Get:17 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 libmount1 amd64 2.37.2-4ubuntu3.3 [122 kB]",
        "Get:18 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 eject amd64 2.37.2-4ubuntu3.3 [26.8 kB]",
        "Get:19 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 libexpat1 amd64 2.4.7-1ubuntu0.3 [91.0 kB]",
        "Get:20 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 vim amd64 2:8.2.3995-1ubuntu2.16 [1736 kB]",
        "Get:21 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 vim-tiny amd64 2:8.2.3995-1ubuntu2.16 [710 kB]",
        "Get:22 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 vim-runtime all 2:8.2.3995-1ubuntu2.16 [6835 kB]",
        "Get:23 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 xxd amd64 2:8.2.3995-1ubuntu2.16 [55.0 kB]",
        "Get:24 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 vim-common all 2:8.2.3995-1ubuntu2.16 [81.5 kB]",
        "Get:25 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 bsdextrautils amd64 2.37.2-4ubuntu3.3 [71.4 kB]",
        "Get:26 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 libldap-2.5-0 amd64 2.5.17+dfsg-0ubuntu0.22.04.1 [183 kB]",
        "Get:27 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 curl amd64 7.81.0-1ubuntu1.16 [194 kB]",
        "Get:28 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 libcurl4 amd64 7.81.0-1ubuntu1.16 [290 kB]",
        "Get:29 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 ethtool amd64 1:5.16-1ubuntu0.1 [207 kB]",
        "Get:30 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 libfdisk1 amd64 2.37.2-4ubuntu3.3 [140 kB]",
        "Get:31 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 fdisk amd64 2.37.2-4ubuntu3.3 [122 kB]",
        "Get:32 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 libcurl3-gnutls amd64 7.81.0-1ubuntu1.16 [284 kB]",
        "Get:33 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 libgpgme11 amd64 1.16.0-1.2ubuntu4.2 [136 kB]",
        "Get:34 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 libldap-common all 2.5.17+dfsg-0ubuntu0.22.04.1 [15.8 kB]",
        "Get:35 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 python3-cryptography amd64 3.4.8-1ubuntu2.2 [236 kB]",
        "Get:36 http://ap-southeast-2.ec2.archive.ubuntu.com/ubuntu jammy-updates/main amd64 snapd amd64 2.61.3+22.04 [24.7 MB]",
        "Fetched 43.9 MB in 1s (54.8 MB/s)",
        "(Reading database ... ",
        "(Reading database ... 5%",
        "(Reading database ... 10%",
        "(Reading database ... 15%",
        "(Reading database ... 20%",
        "(Reading database ... 25%",
        "(Reading database ... 30%",
        "(Reading database ... 35%",
        "(Reading database ... 40%",
        "(Reading database ... 45%",
        "(Reading database ... 50%",
        "(Reading database ... 55%",
        "(Reading database ... 60%",
        "(Reading database ... 65%",
        "(Reading database ... 70%",
        "(Reading database ... 75%",
        "(Reading database ... 80%",
        "(Reading database ... 85%",
        "(Reading database ... 90%",
        "(Reading database ... 95%",
        "(Reading database ... 100%",
        "(Reading database ... 65314 files and directories currently installed.)",
        "Preparing to unpack .../bash_5.1-6ubuntu1.1_amd64.deb ...",
        "Unpacking bash (5.1-6ubuntu1.1) over (5.1-6ubuntu1) ...",
        "Setting up bash (5.1-6ubuntu1.1) ...",
        "update-alternatives: using /usr/share/man/man7/bash-builtins.7.gz to provide /usr/share/man/man7/builtins.7.gz (builtins.7.gz) in auto mode",
        "(Reading database ... ",
        "(Reading database ... 5%",
        "(Reading database ... 10%",
        "(Reading database ... 15%",
        "(Reading database ... 20%",
        "(Reading database ... 25%",
        "(Reading database ... 30%",
        "(Reading database ... 35%",
        "(Reading database ... 40%",
        "(Reading database ... 45%",
        "(Reading database ... 50%",
        "(Reading database ... 55%",
        "(Reading database ... 60%",
        "(Reading database ... 65%",
        "(Reading database ... 70%",
        "(Reading database ... 75%",
        "(Reading database ... 80%",
        "(Reading database ... 85%",
        "(Reading database ... 90%",
        "(Reading database ... 95%",
        "(Reading database ... 100%",
        "(Reading database ... 65314 files and directories currently installed.)",
        "Preparing to unpack .../bsdutils_1%3a2.37.2-4ubuntu3.3_amd64.deb ...",
        "Unpacking bsdutils (1:2.37.2-4ubuntu3.3) over (1:2.37.2-4ubuntu3) ...",
        "Setting up bsdutils (1:2.37.2-4ubuntu3.3) ...",
        "(Reading database ... ",
        "(Reading database ... 5%",
        "(Reading database ... 10%",
        "(Reading database ... 15%",
        "(Reading database ... 20%",
        "(Reading database ... 25%",
        "(Reading database ... 30%",
        "(Reading database ... 35%",
        "(Reading database ... 40%",
        "(Reading database ... 45%",
        "(Reading database ... 50%",
        "(Reading database ... 55%",
        "(Reading database ... 60%",
        "(Reading database ... 65%",
        "(Reading database ... 70%",
        "(Reading database ... 75%",
        "(Reading database ... 80%",
        "(Reading database ... 85%",
        "(Reading database ... 90%",
        "(Reading database ... 95%",
        "(Reading database ... 100%",
        "(Reading database ... 65314 files and directories currently installed.)",
        "Preparing to unpack .../coreutils_8.32-4.1ubuntu1.2_amd64.deb ...",
        "Unpacking coreutils (8.32-4.1ubuntu1.2) over (8.32-4.1ubuntu1.1) ...",
        "Setting up coreutils (8.32-4.1ubuntu1.2) ...",
        "(Reading database ... ",
        "(Reading database ... 5%",
        "(Reading database ... 10%",
        "(Reading database ... 15%",
        "(Reading database ... 20%",
        "(Reading database ... 25%",
        "(Reading database ... 30%",
        "(Reading database ... 35%",
        "(Reading database ... 40%",
        "(Reading database ... 45%",
        "(Reading database ... 50%",
        "(Reading database ... 55%",
        "(Reading database ... 60%",
        "(Reading database ... 65%",
        "(Reading database ... 70%",
        "(Reading database ... 75%",
        "(Reading database ... 80%",
        "(Reading database ... 85%",
        "(Reading database ... 90%",
        "(Reading database ... 95%",
        "(Reading database ... 100%",
        "(Reading database ... 65314 files and directories currently installed.)",
        "Preparing to unpack .../libapt-pkg6.0_2.4.12_amd64.deb ...",
        "Unpacking libapt-pkg6.0:amd64 (2.4.12) over (2.4.11) ...",
        "Setting up libapt-pkg6.0:amd64 (2.4.12) ...",
        "(Reading database ... ",
        "(Reading database ... 5%",
        "(Reading database ... 10%",
        "(Reading database ... 15%",
        "(Reading database ... 20%",
        "(Reading database ... 25%",
        "(Reading database ... 30%",
        "(Reading database ... 35%",
        "(Reading database ... 40%",
        "(Reading database ... 45%",
        "(Reading database ... 50%",
        "(Reading database ... 55%",
        "(Reading database ... 60%",
        "(Reading database ... 65%",
        "(Reading database ... 70%",
        "(Reading database ... 75%",
        "(Reading database ... 80%",
        "(Reading database ... 85%",
        "(Reading database ... 90%",
        "(Reading database ... 95%",
        "(Reading database ... 100%",
        "(Reading database ... 65314 files and directories currently installed.)",
        "Preparing to unpack .../dpkg_1.21.1ubuntu2.3_amd64.deb ...",
        "Unpacking dpkg (1.21.1ubuntu2.3) over (1.21.1ubuntu2.2) ...",
        "Setting up dpkg (1.21.1ubuntu2.3) ...",
        "dpkg-db-backup.service is a disabled or a static unit not running, not starting it.",
        "(Reading database ... ",
        "(Reading database ... 5%",
        "(Reading database ... 10%",
        "(Reading database ... 15%",
        "(Reading database ... 20%",
        "(Reading database ... 25%",
        "(Reading database ... 30%",
        "(Reading database ... 35%",
        "(Reading database ... 40%",
        "(Reading database ... 45%",
        "(Reading database ... 50%",
        "(Reading database ... 55%",
        "(Reading database ... 60%",
        "(Reading database ... 65%",
        "(Reading database ... 70%",
        "(Reading database ... 75%",
        "(Reading database ... 80%",
        "(Reading database ... 85%",
        "(Reading database ... 90%",
        "(Reading database ... 95%",
        "(Reading database ... 100%",
        "(Reading database ... 65314 files and directories currently installed.)",
        "Preparing to unpack .../util-linux_2.37.2-4ubuntu3.3_amd64.deb ...",
        "Unpacking util-linux (2.37.2-4ubuntu3.3) over (2.37.2-4ubuntu3) ...",
        "Setting up util-linux (2.37.2-4ubuntu3.3) ...",
        "fstrim.service is a disabled or a static unit not running, not starting it.",
        "(Reading database ... ",
        "(Reading database ... 5%",
        "(Reading database ... 10%",
        "(Reading database ... 15%",
        "(Reading database ... 20%",
        "(Reading database ... 25%",
        "(Reading database ... 30%",
        "(Reading database ... 35%",
        "(Reading database ... 40%",
        "(Reading database ... 45%",
        "(Reading database ... 50%",
        "(Reading database ... 55%",
        "(Reading database ... 60%",
        "(Reading database ... 65%",
        "(Reading database ... 70%",
        "(Reading database ... 75%",
        "(Reading database ... 80%",
        "(Reading database ... 85%",
        "(Reading database ... 90%",
        "(Reading database ... 95%",
        "(Reading database ... 100%",
        "(Reading database ... 65314 files and directories currently installed.)",
        "Preparing to unpack .../archives/apt_2.4.12_amd64.deb ...",
        "Unpacking apt (2.4.12) over (2.4.11) ...",
        "Setting up apt (2.4.12) ...",
        "(Reading database ... ",
        "(Reading database ... 5%",
        "(Reading database ... 10%",
        "(Reading database ... 15%",
        "(Reading database ... 20%",
        "(Reading database ... 25%",
        "(Reading database ... 30%",
        "(Reading database ... 35%",
        "(Reading database ... 40%",
        "(Reading database ... 45%",
        "(Reading database ... 50%",
        "(Reading database ... 55%",
        "(Reading database ... 60%",
        "(Reading database ... 65%",
        "(Reading database ... 70%",
        "(Reading database ... 75%",
        "(Reading database ... 80%",
        "(Reading database ... 85%",
        "(Reading database ... 90%",
        "(Reading database ... 95%",
        "(Reading database ... 100%",
        "(Reading database ... 65314 files and directories currently installed.)",
        "Preparing to unpack .../apt-utils_2.4.12_amd64.deb ...",
        "Unpacking apt-utils (2.4.12) over (2.4.11) ...",
        "Preparing to unpack .../mount_2.37.2-4ubuntu3.3_amd64.deb ...",
        "Unpacking mount (2.37.2-4ubuntu3.3) over (2.37.2-4ubuntu3) ...",
        "Preparing to unpack .../libsmartcols1_2.37.2-4ubuntu3.3_amd64.deb ...",
        "Unpacking libsmartcols1:amd64 (2.37.2-4ubuntu3.3) over (2.37.2-4ubuntu3) ...",
        "Setting up libsmartcols1:amd64 (2.37.2-4ubuntu3.3) ...",
        "(Reading database ... ",
        "(Reading database ... 5%",
        "(Reading database ... 10%",
        "(Reading database ... 15%",
        "(Reading database ... 20%",
        "(Reading database ... 25%",
        "(Reading database ... 30%",
        "(Reading database ... 35%",
        "(Reading database ... 40%",
        "(Reading database ... 45%",
        "(Reading database ... 50%",
        "(Reading database ... 55%",
        "(Reading database ... 60%",
        "(Reading database ... 65%",
        "(Reading database ... 70%",
        "(Reading database ... 75%",
        "(Reading database ... 80%",
        "(Reading database ... 85%",
        "(Reading database ... 90%",
        "(Reading database ... 95%",
        "(Reading database ... 100%",
        "(Reading database ... 65314 files and directories currently installed.)",
        "Preparing to unpack .../libuuid1_2.37.2-4ubuntu3.3_amd64.deb ...",
        "Unpacking libuuid1:amd64 (2.37.2-4ubuntu3.3) over (2.37.2-4ubuntu3) ...",
        "Setting up libuuid1:amd64 (2.37.2-4ubuntu3.3) ...",
        "(Reading database ... ",
        "(Reading database ... 5%",
        "(Reading database ... 10%",
        "(Reading database ... 15%",
        "(Reading database ... 20%",
        "(Reading database ... 25%",
        "(Reading database ... 30%",
        "(Reading database ... 35%",
        "(Reading database ... 40%",
        "(Reading database ... 45%",
        "(Reading database ... 50%",
        "(Reading database ... 55%",
        "(Reading database ... 60%",
        "(Reading database ... 65%",
        "(Reading database ... 70%",
        "(Reading database ... 75%",
        "(Reading database ... 80%",
        "(Reading database ... 85%",
        "(Reading database ... 90%",
        "(Reading database ... 95%",
        "(Reading database ... 100%",
        "(Reading database ... 65314 files and directories currently installed.)",
        "Preparing to unpack .../uuid-runtime_2.37.2-4ubuntu3.3_amd64.deb ...",
        "Unpacking uuid-runtime (2.37.2-4ubuntu3.3) over (2.37.2-4ubuntu3) ...",
        "Preparing to unpack .../update-manager-core_1%3a22.04.19_all.deb ...",
        "Unpacking update-manager-core (1:22.04.19) over (1:22.04.18) ...",
        "Preparing to unpack .../python3-update-manager_1%3a22.04.19_all.deb ...",
        "Unpacking python3-update-manager (1:22.04.19) over (1:22.04.18) ...",
        "Preparing to unpack .../update-notifier-common_3.192.54.8_all.deb ...",
        "Unpacking update-notifier-common (3.192.54.8) over (3.192.54.6) ...",
        "Preparing to unpack .../libblkid1_2.37.2-4ubuntu3.3_amd64.deb ...",
        "Unpacking libblkid1:amd64 (2.37.2-4ubuntu3.3) over (2.37.2-4ubuntu3) ...",
        "Setting up libblkid1:amd64 (2.37.2-4ubuntu3.3) ...",
        "(Reading database ... ",
        "(Reading database ... 5%",
        "(Reading database ... 10%",
        "(Reading database ... 15%",
        "(Reading database ... 20%",
        "(Reading database ... 25%",
        "(Reading database ... 30%",
        "(Reading database ... 35%",
        "(Reading database ... 40%",
        "(Reading database ... 45%",
        "(Reading database ... 50%",
        "(Reading database ... 55%",
        "(Reading database ... 60%",
        "(Reading database ... 65%",
        "(Reading database ... 70%",
        "(Reading database ... 75%",
        "(Reading database ... 80%",
        "(Reading database ... 85%",
        "(Reading database ... 90%",
        "(Reading database ... 95%",
        "(Reading database ... 100%",
        "(Reading database ... 65314 files and directories currently installed.)",
        "Preparing to unpack .../libmount1_2.37.2-4ubuntu3.3_amd64.deb ...",
        "Unpacking libmount1:amd64 (2.37.2-4ubuntu3.3) over (2.37.2-4ubuntu3) ...",
        "Setting up libmount1:amd64 (2.37.2-4ubuntu3.3) ...",
        "(Reading database ... ",
        "(Reading database ... 5%",
        "(Reading database ... 10%",
        "(Reading database ... 15%",
        "(Reading database ... 20%",
        "(Reading database ... 25%",
        "(Reading database ... 30%",
        "(Reading database ... 35%",
        "(Reading database ... 40%",
        "(Reading database ... 45%",
        "(Reading database ... 50%",
        "(Reading database ... 55%",
        "(Reading database ... 60%",
        "(Reading database ... 65%",
        "(Reading database ... 70%",
        "(Reading database ... 75%",
        "(Reading database ... 80%",
        "(Reading database ... 85%",
        "(Reading database ... 90%",
        "(Reading database ... 95%",
        "(Reading database ... 100%",
        "(Reading database ... 65314 files and directories currently installed.)",
        "Preparing to unpack .../00-eject_2.37.2-4ubuntu3.3_amd64.deb ...",
        "Unpacking eject (2.37.2-4ubuntu3.3) over (2.37.2-4ubuntu3) ...",
        "Preparing to unpack .../01-libexpat1_2.4.7-1ubuntu0.3_amd64.deb ...",
        "Unpacking libexpat1:amd64 (2.4.7-1ubuntu0.3) over (2.4.7-1ubuntu0.2) ...",
        "Preparing to unpack .../02-vim_2%3a8.2.3995-1ubuntu2.16_amd64.deb ...",
        "Unpacking vim (2:8.2.3995-1ubuntu2.16) over (2:8.2.3995-1ubuntu2.15) ...",
        "Preparing to unpack .../03-vim-tiny_2%3a8.2.3995-1ubuntu2.16_amd64.deb ...",
        "Unpacking vim-tiny (2:8.2.3995-1ubuntu2.16) over (2:8.2.3995-1ubuntu2.15) ...",
        "Preparing to unpack .../04-vim-runtime_2%3a8.2.3995-1ubuntu2.16_all.deb ...",
        "Unpacking vim-runtime (2:8.2.3995-1ubuntu2.16) over (2:8.2.3995-1ubuntu2.15) ...",
        "Preparing to unpack .../05-xxd_2%3a8.2.3995-1ubuntu2.16_amd64.deb ...",
        "Unpacking xxd (2:8.2.3995-1ubuntu2.16) over (2:8.2.3995-1ubuntu2.15) ...",
        "Preparing to unpack .../06-vim-common_2%3a8.2.3995-1ubuntu2.16_all.deb ...",
        "Unpacking vim-common (2:8.2.3995-1ubuntu2.16) over (2:8.2.3995-1ubuntu2.15) ...",
        "Preparing to unpack .../07-bsdextrautils_2.37.2-4ubuntu3.3_amd64.deb ...",
        "Unpacking bsdextrautils (2.37.2-4ubuntu3.3) over (2.37.2-4ubuntu3) ...",
        "Preparing to unpack .../08-libldap-2.5-0_2.5.17+dfsg-0ubuntu0.22.04.1_amd64.deb ...",
        "Unpacking libldap-2.5-0:amd64 (2.5.17+dfsg-0ubuntu0.22.04.1) over (2.5.16+dfsg-0ubuntu0.22.04.2) ...",
        "Preparing to unpack .../09-curl_7.81.0-1ubuntu1.16_amd64.deb ...",
        "Unpacking curl (7.81.0-1ubuntu1.16) over (7.81.0-1ubuntu1.15) ...",
        "Preparing to unpack .../10-libcurl4_7.81.0-1ubuntu1.16_amd64.deb ...",
        "Unpacking libcurl4:amd64 (7.81.0-1ubuntu1.16) over (7.81.0-1ubuntu1.15) ...",
        "Preparing to unpack .../11-ethtool_1%3a5.16-1ubuntu0.1_amd64.deb ...",
        "Unpacking ethtool (1:5.16-1ubuntu0.1) over (1:5.16-1) ...",
        "Preparing to unpack .../12-libfdisk1_2.37.2-4ubuntu3.3_amd64.deb ...",
        "Unpacking libfdisk1:amd64 (2.37.2-4ubuntu3.3) over (2.37.2-4ubuntu3) ...",
        "Preparing to unpack .../13-fdisk_2.37.2-4ubuntu3.3_amd64.deb ...",
        "Unpacking fdisk (2.37.2-4ubuntu3.3) over (2.37.2-4ubuntu3) ...",
        "Preparing to unpack .../14-libcurl3-gnutls_7.81.0-1ubuntu1.16_amd64.deb ...",
        "Unpacking libcurl3-gnutls:amd64 (7.81.0-1ubuntu1.16) over (7.81.0-1ubuntu1.15) ...",
        "Preparing to unpack .../15-libgpgme11_1.16.0-1.2ubuntu4.2_amd64.deb ...",
        "Unpacking libgpgme11:amd64 (1.16.0-1.2ubuntu4.2) over (1.16.0-1.2ubuntu4.1) ...",
        "Preparing to unpack .../16-libldap-common_2.5.17+dfsg-0ubuntu0.22.04.1_all.deb ...",
        "Unpacking libldap-common (2.5.17+dfsg-0ubuntu0.22.04.1) over (2.5.16+dfsg-0ubuntu0.22.04.2) ...",
        "Preparing to unpack .../17-python3-cryptography_3.4.8-1ubuntu2.2_amd64.deb ...",
        "Unpacking python3-cryptography (3.4.8-1ubuntu2.2) over (3.4.8-1ubuntu2.1) ...",
        "Preparing to unpack .../18-snapd_2.61.3+22.04_amd64.deb ...",
        "Unpacking snapd (2.61.3+22.04) over (2.58+22.04.1) ...",
        "Setting up libexpat1:amd64 (2.4.7-1ubuntu0.3) ...",
        "Setting up snapd (2.61.3+22.04) ...",
        "Installing new version of config file /etc/apparmor.d/usr.lib.snapd.snap-confine.real ...",
        "snapd.failure.service is a disabled or a static unit not running, not starting it.",
        "snapd.snap-repair.service is a disabled or a static unit not running, not starting it.",
        "Failed to restart snapd.mounts-pre.target: Operation refused, unit snapd.mounts-pre.target may be requested by dependency only (it is configured to refuse manual start/stop).",
        "See system logs and 'systemctl status snapd.mounts-pre.target' for details.",
        "Could not execute systemctl:  at /usr/bin/deb-systemd-invoke line 142.",
        "Setting up apt-utils (2.4.12) ...",
        "Setting up bsdextrautils (2.37.2-4ubuntu3.3) ...",
        "Setting up libldap-common (2.5.17+dfsg-0ubuntu0.22.04.1) ...",
        "Setting up libgpgme11:amd64 (1.16.0-1.2ubuntu4.2) ...",
        "Setting up libldap-2.5-0:amd64 (2.5.17+dfsg-0ubuntu0.22.04.1) ...",
        "Setting up xxd (2:8.2.3995-1ubuntu2.16) ...",
        "Setting up eject (2.37.2-4ubuntu3.3) ...",
        "Setting up vim-common (2:8.2.3995-1ubuntu2.16) ...",
        "Setting up python3-cryptography (3.4.8-1ubuntu2.2) ...",
        "Setting up libfdisk1:amd64 (2.37.2-4ubuntu3.3) ...",
        "Setting up mount (2.37.2-4ubuntu3.3) ...",
        "Setting up uuid-runtime (2.37.2-4ubuntu3.3) ...",
        "uuidd.service is a disabled or a static unit not running, not starting it.",
        "Setting up python3-update-manager (1:22.04.19) ...",
        "Setting up libcurl4:amd64 (7.81.0-1ubuntu1.16) ...",
        "Setting up curl (7.81.0-1ubuntu1.16) ...",
        "Setting up vim-runtime (2:8.2.3995-1ubuntu2.16) ...",
        "Setting up ethtool (1:5.16-1ubuntu0.1) ...",
        "Setting up vim (2:8.2.3995-1ubuntu2.16) ...",
        "Setting up libcurl3-gnutls:amd64 (7.81.0-1ubuntu1.16) ...",
        "Setting up vim-tiny (2:8.2.3995-1ubuntu2.16) ...",
        "Setting up fdisk (2.37.2-4ubuntu3.3) ...",
        "Setting up update-manager-core (1:22.04.19) ...",
        "Setting up update-notifier-common (3.192.54.8) ...",
        "update-notifier-download.service is a disabled or a static unit not running, not starting it.",
        "update-notifier-motd.service is a disabled or a static unit not running, not starting it.",
        "Processing triggers for man-db (2.10.2-1) ...",
        "Processing triggers for dbus (1.12.20-2ubuntu4.1) ...",
        "Processing triggers for install-info (6.8-4build1) ...",
        "Processing triggers for libc-bin (2.35-0ubuntu3.6) ..."
    ]
}

TASK [Check if reboot is needed] ********************************************************************************
fatal: [13.211.209.95]: FAILED! => {"changed": false, "cmd": "needs-restart", "msg": "[Errno 2] No such file or directory: b'needs-restart'", "rc": 2, "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}
...ignoring

TASK [Display reboot required status] ***************************************************************************
ok: [13.211.209.95] => {
    "msg": "Reboot is needed: no"
}

PLAY RECAP ******************************************************************************************************
13.211.209.95              : ok=7    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=1

(myenv) [devops-training] % :

```

</details>





### Creating and Destroying EC2 Instance with Terraform



```
(myenv) [devops-training] % :./run.sh c
------------------------------------
Creating EC2 Instance with terraform
------------------------------------
Run terraform apply...

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated
with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_instance.DevOps-Training will be created
  + resource "aws_instance" "DevOps-Training" {
      + ami                                  = "ami-09c8d5d747253fb7a"
      + arn                                  = (known after apply)
      + associate_public_ip_address          = (known after apply)
      + availability_zone                    = (known after apply)
      + cpu_core_count                       = (known after apply)
      + cpu_threads_per_core                 = (known after apply)
      + disable_api_stop                     = (known after apply)
      + disable_api_termination              = (known after apply)
      + ebs_optimized                        = (known after apply)
      + get_password_data                    = false
      + host_id                              = (known after apply)
      + host_resource_group_arn              = (known after apply)
      + iam_instance_profile                 = (known after apply)
      + id                                   = (known after apply)
      + instance_initiated_shutdown_behavior = (known after apply)
      + instance_lifecycle                   = (known after apply)
      + instance_state                       = (known after apply)
      + instance_type                        = "t2.micro"
      + ipv6_address_count                   = (known after apply)
      + ipv6_addresses                       = (known after apply)
      + key_name                             = "m3air"
      + monitoring                           = (known after apply)
      + outpost_arn                          = (known after apply)
      + password_data                        = (known after apply)
      + placement_group                      = (known after apply)
      + placement_partition_number           = (known after apply)
      + primary_network_interface_id         = (known after apply)
      + private_dns                          = (known after apply)
      + private_ip                           = (known after apply)
      + public_dns                           = (known after apply)
      + public_ip                            = (known after apply)
      + secondary_private_ips                = (known after apply)
      + security_groups                      = [
          + "SSHAccess",
        ]
      + source_dest_check                    = true
      + spot_instance_request_id             = (known after apply)
      + subnet_id                            = (known after apply)
      + tags                                 = {
          + "Name" = "DevOps-Training"
        }
      + tags_all                             = {
          + "Name" = "DevOps-Training"
        }
      + tenancy                              = (known after apply)
      + user_data                            = "93f55b060f203b20dd188cc0805534bf4b5e7ec1"
      + user_data_base64                     = (known after apply)
      + user_data_replace_on_change          = false
      + vpc_security_group_ids               = (known after apply)
    }

  # aws_route_table.terraform will be created
  + resource "aws_route_table" "terraform" {
      + arn              = (known after apply)
      + id               = (known after apply)
      + owner_id         = (known after apply)
      + propagating_vgws = (known after apply)
      + route            = [
          + {
              + carrier_gateway_id         = ""
              + cidr_block                 = "0.0.0.0/0"
              + core_network_arn           = ""
              + destination_prefix_list_id = ""
              + egress_only_gateway_id     = ""
              + gateway_id                 = "igw-053b79f1e698caf46"
              + ipv6_cidr_block            = ""
              + local_gateway_id           = ""
              + nat_gateway_id             = ""
              + network_interface_id       = ""
              + transit_gateway_id         = ""
              + vpc_endpoint_id            = ""
              + vpc_peering_connection_id  = ""
            },
        ]
      + tags_all         = (known after apply)
      + vpc_id           = "vpc-025e30832fbec6fea"
    }

  # aws_security_group.terraform will be created
  + resource "aws_security_group" "terraform" {
      + arn                    = (known after apply)
      + description            = "Allow SSH access"
      + egress                 = [
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + description      = ""
              + from_port        = 0
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "-1"
              + security_groups  = []
              + self             = false
              + to_port          = 0
            },
        ]
      + id                     = (known after apply)
      + ingress                = [
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + description      = ""
              + from_port        = 22
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "tcp"
              + security_groups  = []
              + self             = false
              + to_port          = 22
            },
        ]
      + name                   = "SSHAccess"
      + name_prefix            = (known after apply)
      + owner_id               = (known after apply)
      + revoke_rules_on_delete = false
      + tags_all               = (known after apply)
      + vpc_id                 = (known after apply)
    }

Plan: 3 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + instance_ips = (known after apply)
aws_route_table.terraform: Creating...
aws_security_group.terraform: Creating...
aws_instance.DevOps-Training: Creating...
aws_route_table.terraform: Creation complete after 6s [id=rtb-012085708ee0ff001]
aws_security_group.terraform: Creation complete after 7s [id=sg-0d633d812814fd566]
aws_instance.DevOps-Training: Still creating... [10s elapsed]
aws_instance.DevOps-Training: Still creating... [20s elapsed]
aws_instance.DevOps-Training: Still creating... [30s elapsed]
aws_instance.DevOps-Training: Creation complete after 37s [id=i-0074243c0d7b8b9e2]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

instance_ips = "13.211.209.95"
Adding instance ip to hosts.csv...

```


### Running sys_reading.py remotely with sys_reading_remote.py


```
(myenv) [devops-training] % :./run.sh s
-------------------------
Status of all the servers
-------------------------
Running the sys_reading.py remotely...
--------------------------------------------------
System usage for  cloud :
Hostname:  cloud.ss.fish
Platform: Linux
Uptime: 3days
CPU (%): 0.0
Mem Used (GB): 0.3 / 0.9
Network Card IPs: 45.76.152.170
Public IP: 45.76.152.170
Disk Usage '/' (GB): 22.7 / 29.9

--------------------------------------------------
System usage for  ss :
Hostname:  ss.fish
Platform: OpenBSD
Uptime: 122days
CPU (%): 2.0
Mem Used (GB): 0.2 / 1.0
Network Card IPs: 8.9.6.20, 10.1.96.4
Public IP: 8.9.6.20
Disk Usage '/' (GB): 2.4 / 21.3

--------------------------------------------------
System usage for  ec2 :
Hostname:  ip-172-31-1-232
Platform: Linux
Uptime: 7min
CPU (%): 0.0
Mem Used (GB): 0.2 / 0.9
Network Card IPs:
Public IP: 13.211.209.95
Disk Usage '/' (GB): 1.8 / 7.6

```



### Connect it all together
The background color is `#ffffff` for light mode and `#000000` for dark mode.


[Contribution guidelines for this project](ansible-playbook/update_upgrade_cloud.yml)


![Screenshot of a comment on a GitHub issue showing an image, added in the Markdown, of an Octocat smiling and raising a tentacle.](https://myoctocat.com/assets/images/base-octocat.svg)


![Alt Text](https://media.giphy.com/media/vFKqnCdLPNOKc/giphy.gif)
