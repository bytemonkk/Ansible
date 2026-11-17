# Ansible Automation Lab Using Docker – Complete Practical Guide

## 1. Project Overview

This project demonstrates how to configure and manage multiple Ubuntu Linux machines using Ansible.

We create one Ansible Controller and three Managed Nodes using Docker containers. The controller communicates with the managed nodes through SSH and executes commands and playbooks remotely.

The practical implementation covers:

- Ansible Controller setup.
- Ubuntu managed-node Docker image creation.
- Multiple client container creation.
- Docker networking and hostname resolution.
- SSH key-based authentication.
- Ansible inventory configuration.
- Ad-hoc command execution.
- Remote system information gathering.
- Nginx installation and verification.
- YAML Playbook creation and execution.
- BusyBox installation using a playbook.
- Idempotency verification.

### 1.1 Architecture

```text
                 WINDOWS HOST
                 E:\Ansible
                      |
              Docker Environment
                      |
             ansible-network
                      |
             ANSIBLE CONTROLLER
              Ubuntu + Ansible
                      |
                SSH (Port 22)
                      |
          +-----------+-----------+
          |           |           |
       Client1     Client2     Client3
       Ubuntu      Ubuntu      Ubuntu
          |           |           |
       Nginx       Database    Application
       Webserver    Group       Group
```

**Controller:** The machine where Ansible is installed and from which automation is executed.

**Managed Nodes:** Remote machines that Ansible controls.

**SSH:** The communication protocol used by Ansible to connect to Linux managed nodes.

---

# 2. Prerequisites

The following tools are required:

| Tool | Purpose |
|---|---|
| Windows 10/11 | Host operating system |
| Docker Desktop | Container management |
| Docker Engine | Running Ubuntu containers |
| Docker Compose | Optional multi-container orchestration |
| Ubuntu 24.04 | Controller and managed-node operating system |
| Python 3 | Required by Ansible on managed Linux nodes |
| Ansible | Configuration management and automation |
| OpenSSH | Remote communication |
| PowerShell | Windows terminal |

### Verify Docker installation

Open Windows PowerShell:

```powershell
docker --version
```

Verify Docker Compose:

```powershell
docker compose version
```

Verify running containers:

```powershell
docker ps
```

**Note:** Other project containers may already be running on Docker. Do not stop or remove unrelated containers while configuring this lab.

---

# 3. Create the Project Directory Structure

Create the main project directory:

```powershell
mkdir E:\Ansible
cd E:\Ansible
```

Create the required directories:

```powershell
mkdir controller
mkdir inventory
mkdir managed-nodes
mkdir playbooks
```

The resulting structure:

```text
E:\Ansible
|
+---controller
|       Dockerfile
|
+---inventory
|       hosts.ini
|
+---managed-nodes
|       Dockerfile
|
+---playbooks
|       install_busybox.yml
|
+---ansible_key
+---ansible_key.pub
```

**Purpose of each directory:**

- `controller`: Contains the Dockerfile used to build the Ansible controller.
- `managed-nodes`: Contains the Dockerfile used to build the Ubuntu client image.
- `inventory`: Contains the Ansible inventory file.
- `playbooks`: Contains YAML automation files.
- `ansible_key`: Private SSH key.
- `ansible_key.pub`: Public SSH key.

---

# 4. Build the Ansible Controller

The controller is the central machine from which we execute Ansible commands.

We install Ansible inside an Ubuntu Docker container instead of installing it directly on Windows.

### Step 1: Create the controller Dockerfile

Navigate to:

```powershell
cd E:\Ansible\controller
code Dockerfile
```

Add:

```dockerfile
FROM ubuntu:24.04

RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    openssh-client \
    sshpass \
    && rm -rf /var/lib/apt/lists/*

RUN pip3 install ansible --break-system-packages

CMD ["sleep", "infinity"]
```

### Explanation

| Instruction | Purpose |
|---|---|
| `FROM ubuntu:24.04` | Uses Ubuntu as the base image |
| `apt-get update` | Updates package information |
| `python3` | Installs Python |
| `python3-pip` | Installs Python package manager |
| `openssh-client` | Provides SSH client |
| `sshpass` | Provides password-based SSH support |
| `pip3 install ansible` | Installs Ansible |
| `CMD ["sleep", "infinity"]` | Keeps the container running |

### Step 2: Build the Docker image

Return to the project root:

```powershell
cd E:\Ansible
```

Build the image:

```powershell
docker build -t ansible-controller:v1 ./controller
```

### Step 3: Create the controller container

```powershell
docker run -d --name ansible-controller ansible-controller:v1
```

### Step 4: Enter the controller

```powershell
docker exec -it ansible-controller bash
```

Verify Ansible:

```bash
ansible --version
```

If the version information is displayed, the controller is ready.

---

# 5. Build the Managed Node Docker Image

Managed nodes are the remote Ubuntu machines that Ansible will control.

Instead of creating three different images, we build one common Ubuntu image and create three independent containers from it.

### Step 1: Create the managed-node Dockerfile

Open another Windows PowerShell terminal:

```powershell
cd E:\Ansible\managed-nodes
notepad Dockerfile
```

Add:

```dockerfile
FROM ubuntu:24.04

RUN apt-get update && apt-get install -y \
    openssh-server \
    python3 \
    sudo \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /run/sshd

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]
```

### Explanation

| Instruction | Purpose |
|---|---|
| `FROM ubuntu:24.04` | Uses Ubuntu as the base image |
| `openssh-server` | Installs the SSH server |
| `python3` | Required for Ansible modules |
| `sudo` | Provides privilege escalation support |
| `mkdir -p /run/sshd` | Creates the SSH runtime directory |
| `EXPOSE 22` | Documents the SSH port |
| `sshd -D` | Runs the SSH daemon in the foreground |

**Important:** `EXPOSE 22` does not publish the port to the Windows host. Our containers will communicate through their shared Docker network.

### Step 2: Build the managed-node image

```powershell
cd E:\Ansible
docker build -t ansible-managed-node:v1 ./managed-nodes
```

This creates one reusable image.

---

# 6. Create Three Client Containers

Now we create three independent Ubuntu containers using the image we built.

Execute in PowerShell:

```powershell
docker run -d --name client1 ansible-managed-node:v1

docker run -d --name client2 ansible-managed-node:v1

docker run -d --name client3 ansible-managed-node:v1
```

### Verify the containers

```powershell
docker ps
```

Expected containers:

```text
ansible-controller
client1
client2
client3
```

Each container acts as an independent managed Linux machine.

### Why create three containers?

In a real infrastructure, different servers perform different responsibilities.

For our lab:

| Container | Assigned responsibility |
|---|---|
| client1 | Web server |
| client2 | Database server |
| client3 | Application server |

---

# 7. Understand SSH Communication

## Why do we need SSH?

Ansible normally connects to Linux managed nodes through SSH.

It executes commands remotely rather than requiring Ansible to be installed on every client.

The controller requires an SSH client, and the managed nodes require an SSH server.

### SSH components

| Component | Purpose |
|---|---|
| `ssh` | SSH client command |
| `sshd` | SSH server daemon |
| Port 22 | Default SSH port |
| Private key | Used by the client for authentication |
| Public key | Installed on the remote server |
| `authorized_keys` | Stores permitted public keys |

### Important distinction

- SSH client: Runs on the controller.
- SSH server: Runs on each managed node.
- Ansible: Uses SSH to communicate with managed nodes.

---

# 8. Generate SSH Key Pair

We use public-key authentication so that Ansible can connect without manually entering passwords.

Open Windows PowerShell:

```powershell
cd E:\Ansible
```

Generate an Ed25519 key:

```powershell
ssh-keygen -t ed25519 -f .\ansible_key -N '""'
```

This is the command used in our lab.

Alternatively, run:

```powershell
ssh-keygen -t ed25519 -f .\ansible_key
```

Then press Enter twice when asked for an empty passphrase.

### Understand the difference

**Private key:**

```text
ansible_key
```

Only the controller should have access to this file. Never share it publicly.

**Public key:**

```text
ansible_key.pub
```

This is distributed to the managed nodes.

Verify the files:

```powershell
dir
```

You should see:

```text
ansible_key
ansible_key.pub
```

---

# 9. Distribute the Public Key to All Three Clients

First, create the SSH directory inside each client.

Execute in PowerShell:

```powershell
docker exec client1 mkdir -p /root/.ssh
docker exec client2 mkdir -p /root/.ssh
docker exec client3 mkdir -p /root/.ssh
```

Copy the public key:

```powershell
docker cp .\ansible_key.pub client1:/root/.ssh/authorized_keys

docker cp .\ansible_key.pub client2:/root/.ssh/authorized_keys

docker cp .\ansible_key.pub client3:/root/.ssh/authorized_keys
```

### Set correct permissions

Execute:

```powershell
docker exec client1 chmod 700 /root/.ssh
docker exec client1 chmod 600 /root/.ssh/authorized_keys

docker exec client2 chmod 700 /root/.ssh
docker exec client2 chmod 600 /root/.ssh/authorized_keys

docker exec client3 chmod 700 /root/.ssh
docker exec client3 chmod 600 /root/.ssh/authorized_keys
```

### Why are these permissions required?

| Permission | Meaning |
|---|---|
| `700` | Only the owner can access the SSH directory |
| `600` | Only the owner can read and write authorized_keys |

SSH checks permissions for security reasons.

### Verify the authorized key

```powershell
docker exec -it client1 bash
```

Inside client1:

```bash
ls -la /root/.ssh
```

You should see:

```text
authorized_keys
```

Exit:

```bash
exit
```

**Note:** The `.ssh` directory is hidden because its name begins with a dot. Use `ls -la` to display hidden files.

---

# 10. Give the Controller Access to the Private SSH Key

The controller needs the private key to authenticate with the managed nodes.

Run this in Windows PowerShell:

```powershell
cd E:\Ansible

docker cp .\ansible_key ansible-controller:/root/ansible_key
```

Set the private key permissions inside the controller:

```powershell
docker exec ansible-controller chmod 600 /root/ansible_key
```

Verify:

```powershell
docker exec ansible-controller ls -l /root/ansible_key
```

The controller now has access to the private key.

**Security rule:** Never copy the private key into the managed nodes.

---

# 11. Docker Networking Problem and Solution

## Problem encountered

During the first SSH test, we encountered:

```text
ssh: Could not resolve hostname client1:
Name or service not known
```

### Why did this happen?

The controller and clients were created independently using `docker run`.

They were not initially connected to a common user-defined Docker network.

The problem was not the SSH key.

The controller could not resolve the hostname `client1`.

### Understand Docker networking

Docker provides different network types.

The default bridge network does not provide the same automatic container-name DNS resolution as a user-defined bridge network.

A user-defined bridge network allows containers to communicate using their container names.

### Step 1: Create a custom Docker network

Run in PowerShell:

```powershell
docker network create ansible-network
```

### Step 2: Connect the controller

```powershell
docker network connect ansible-network ansible-controller
```

### Step 3: Connect all three clients

```powershell
docker network connect ansible-network client1

docker network connect ansible-network client2

docker network connect ansible-network client3
```

All four Ansible lab containers now share the same network.

### Step 4: Verify network connectivity and hostname resolution

Enter the controller:

```powershell
docker exec -it ansible-controller bash
```

Execute:

```bash
getent hosts client1 client2 client3
```

This verifies that the container names resolve to IP addresses.

We can also test basic ICMP connectivity using:

```bash
ping -c 2 client1
```

However, `ping` tests ICMP connectivity, not SSH. It may not work if the required utility is absent or ICMP traffic is blocked.

`getent hosts` verifies name resolution, while an SSH test verifies SSH connectivity.

### Verify the Docker network

From PowerShell:

```powershell
docker network inspect ansible-network
```

The output should show the controller and three client containers as network members.

---

# 12. Test SSH Connectivity

Now that the Docker networking issue is fixed, test SSH from the controller.

Enter the controller:

```powershell
docker exec -it ansible-controller bash
```

Connect to client1:

```bash
ssh -i /root/ansible_key root@client1
```

The first time, SSH may display a host authenticity confirmation.

Enter:

```text
yes
```

After successful login, verify the remote username:

```bash
whoami
```

Expected:

```text
root
```

Verify the remote hostname:

```bash
hostname
```

Exit the remote machine:

```bash
exit
```

### Test the other two clients

```bash
ssh -i /root/ansible_key root@client2

ssh -i /root/ansible_key root@client3
```

If all three connections succeed, SSH key-based authentication is working.

### Important SSH username concept

In:

```bash
ssh -i /root/ansible_key root@client1
```

- `root` is the remote Linux username.
- `client1` is the remote hostname.
- `/root/ansible_key` is the private key path.

`root` is not the home directory name in this command. It is the username. The root user's home directory is `/root`.

---

# 13. What If the SSH Username Is Dobby?

Suppose we create another Linux user:

```bash
useradd -m -s /bin/bash dobby
```

Here:

- `-m` creates the home directory.
- `-s /bin/bash` sets the login shell.

The public key must be authorized for `dobby`, not just root.

For example, on client1:

```bash
mkdir -p /home/dobby/.ssh
```

Copy the authorized public key:

```bash
cp /root/.ssh/authorized_keys \
   /home/dobby/.ssh/authorized_keys
```

Set ownership:

```bash
chown -R dobby:dobby /home/dobby/.ssh
```

Set permissions:

```bash
chmod 700 /home/dobby/.ssh

chmod 600 /home/dobby/.ssh/authorized_keys
```

Connect using:

```bash
ssh -i /root/ansible_key dobby@client1
```

**Important:** The `dobby` account must exist on the remote machine, and its public key must be authorized. The root user's authorized keys do not automatically authorize another account.

---

# 14. Create the Ansible Inventory

An inventory tells Ansible which machines it should manage and how to connect to them.

### Step 1: Create the inventory file

On Windows PowerShell:

```powershell
cd E:\Ansible\inventory

notepad hosts.ini
```

Add:

```ini
[webservers]
client1 ansible_host=client1 ansible_user=root

[databases]
client2 ansible_host=client2 ansible_user=root

[applications]
client3 ansible_host=client3 ansible_user=root

[all:vars]
ansible_ssh_private_key_file=/root/ansible_key
ansible_python_interpreter=/usr/bin/python3
```

### Explanation

| Configuration | Purpose |
|---|---|
| `[webservers]` | Defines web server group |
| `[databases]` | Defines database group |
| `[applications]` | Defines application group |
| `ansible_host` | Specifies the connection hostname |
| `ansible_user` | Specifies the remote SSH username |
| `ansible_ssh_private_key_file` | Specifies the private key |
| `ansible_python_interpreter` | Specifies the Python interpreter on clients |

### Why use groups?

Groups allow us to target specific machines instead of executing commands on every server.

For example:

```bash
ansible webservers ...
```

Targets only client1.

```bash
ansible databases ...
```

Targets only client2.

```bash
ansible all ...
```

Targets all three clients.

### Step 2: Copy inventory into the controller

Currently, `hosts.ini` exists on Windows. The controller container cannot automatically access that file.

Run this in PowerShell:

```powershell
cd E:\Ansible

docker cp .\inventory\hosts.ini ansible-controller:/root/hosts.ini
```

Verify:

```powershell
docker exec ansible-controller cat /root/hosts.ini
```

**Important:** `docker cp` copies a snapshot of the file. If you later modify the Windows inventory, copy it again into the controller.

---

# 15. First Ansible Ad-hoc Command

Ad-hoc commands allow us to execute individual tasks without creating a YAML playbook.

### Step 1: Enter the controller

```powershell
docker exec -it ansible-controller bash
```

### Step 2: List all inventory hosts

```bash
ansible all -i /root/hosts.ini --list-hosts
```

Expected hosts:

```text
client1
client2
client3
```

### Step 3: Test all managed nodes

```bash
ansible all -i /root/hosts.ini -m ping
```

Expected output:

```text
client1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

client2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

client3 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### What just happened?

You executed one command from the controller, and Ansible executed it independently on three remote Linux machines.

No manual SSH login was required.

**Important:** Ansible's `ping` module does not send ICMP packets. It checks whether Ansible can connect to the managed node and execute its Python-based module.

---

# 16. Execute Commands on Specific Server Groups

Until now, we have used `all` to target every client.

Now we target individual groups.

### Target the web server

```bash
ansible webservers -i /root/hosts.ini -m command -a "hostname"
```

### Target the database server

```bash
ansible databases -i /root/hosts.ini -m command -a "whoami"
```

### Target the application server

```bash
ansible applications -i /root/hosts.ini -m command -a "hostname"
```

### Gather operating system information

```bash
ansible all -i /root/hosts.ini -m command -a "cat /etc/os-release"
```

### Check disk usage

```bash
ansible all -i /root/hosts.ini -m command -a "df -h"
```

### Check memory information

```bash
ansible all -i /root/hosts.ini -m command -a "free -h"
```

### Understand the command structure

```bash
ansible all -i /root/hosts.ini -m command -a "hostname"
```

| Component | Purpose |
|---|---|
| `ansible` | Executes an ad-hoc command |
| `all` | Targets all inventory hosts |
| `-i` | Specifies the inventory |
| `-m command` | Selects the command module |
| `-a` | Passes arguments to the module |
| `hostname` | Command executed remotely |

---

# 17. Install Nginx on client1

Now we move from checking server information to actual configuration management.

We install Nginx only on the web server group.

Execute inside the controller:

```bash
ansible webservers -i /root/hosts.ini -m apt -a "name=nginx state=present update_cache=yes"
```

### Understand the command

| Component | Purpose |
|---|---|
| `webservers` | Targets client1 only |
| `-m apt` | Uses Ansible's package-management module |
| `name=nginx` | Specifies the Nginx package |
| `state=present` | Ensures Nginx is installed |
| `update_cache=yes` | Updates package information first |

Expected result:

```text
client1 | CHANGED
```

### Important problem encountered: Nginx did not start automatically

During installation, the output included:

```text
invoke-rc.d: could not determine current runlevel
policy-rc.d denied execution of start.
```

### Why did this happen?

Our managed nodes are Docker containers, not full virtual machines or physical Ubuntu systems.

They do not run systemd by default.

The package installation succeeded, but the container environment prevented the package installer from automatically starting Nginx through its normal service-management mechanism.

Therefore, we manually started Nginx for this lab.

### Verify Nginx installation

```bash
ansible webservers -i /root/hosts.ini -m command -a "nginx -v"
```

Expected:

```text
nginx version: nginx/1.24.0 (Ubuntu)
```

### Start Nginx

```bash
ansible webservers -i /root/hosts.ini -m command -a "nginx"
```

A successful result returns:

```text
client1 | CHANGED | rc=0
```

### Verify running Nginx processes

```bash
ansible webservers -i /root/hosts.ini -m command -a "pgrep -a nginx"
```

Expected output includes:

```text
nginx: master process nginx
nginx: worker process
```

The master process manages Nginx, while worker processes handle requests.

### Install curl

Initially, we attempted to test the Nginx server using curl, but encountered:

```text
Error executing command:
[Errno 2] No such file or directory: b'curl'
```

### Why did this happen?

The minimal Ubuntu managed-node image did not include curl.

Ansible connected successfully, but the remote machine could not find the curl executable.

### Solution: Install curl

```bash
ansible webservers -i /root/hosts.ini -m apt -a "name=curl state=present update_cache=yes"
```

### Test the Nginx web server

```bash
ansible webservers -i /root/hosts.ini -m command -a "curl -I http://localhost"
```

Expected response:

```text
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Content-Type: text/html
```

**Result:** Nginx was successfully installed, started and tested on client1.

---

# 18. Create the First Ansible Playbook

A Playbook is a YAML file in which we define the desired state of our infrastructure and the tasks required to achieve it.

Instead of manually executing multiple commands, we define the configuration once and execute the playbook.

### Ad-hoc commands vs Playbooks

| Ad-hoc commands | Playbooks |
|---|---|
| Execute individual tasks | Execute multiple tasks |
| Usually command-line based | YAML-based |
| Useful for quick operations | Useful for repeatable automation |
| Less structured | Structured and reusable |

### Example

We want client1 to have:

- BusyBox installed.
- The BusyBox executable available.
- Installation verified.

We define these requirements in a YAML file.

---

# 19. Create the BusyBox Playbook

### Step 1: Navigate to the playbooks directory

On Windows PowerShell:

```powershell
cd E:\Ansible\playbooks
```

Create the file:

```powershell
notepad install_busybox.yml
```

### Step 2: Add the YAML configuration

```yaml
---
- name: Install BusyBox on Client 1
  hosts: webservers
  become: true

  tasks:
    - name: Install BusyBox
      ansible.builtin.apt:
        name: busybox
        state: present
        update_cache: true

    - name: Verify BusyBox installation
      ansible.builtin.command: busybox
      register: busybox_result
      changed_when: false

    - name: Display BusyBox verification
      ansible.builtin.debug:
        var: busybox_result.stdout
```

### Explanation of the Playbook

**`---`**

Indicates the beginning of a YAML document.

**`name`**

Provides a descriptive name for the play.

**`hosts: webservers`**

Specifies the inventory group on which the playbook will run.

In our lab, this group contains client1.

**`become: true`**

Enables privilege escalation so that tasks can run with administrative privileges.

**`tasks`**

Contains the operations Ansible must perform.

**`ansible.builtin.apt`**

Uses Ansible's built-in APT module to manage Ubuntu packages.

**`name: busybox`**

Specifies the package to install.

**`state: present`**

Ensures the package is installed.

**`update_cache: true`**

Updates the APT package cache.

**`register: busybox_result`**

Stores the output of the BusyBox verification command in a variable.

**`changed_when: false`**

Indicates that the verification command should not be reported as a system change.

**`ansible.builtin.debug`**

Displays the stored output.

---

# 20. Copy the Playbook into the Controller

Our playbook exists on Windows, so we must transfer it to the controller container.

Open Windows PowerShell:

```powershell
cd E:\Ansible\playbooks
```

Copy the file:

```powershell
docker cp .\install_busybox.yml ansible-controller:/root/install_busybox.yml
```

Verify:

```powershell
docker exec ansible-controller ls -l /root/install_busybox.yml
```

Expected:

```text
/root/install_busybox.yml
```

---

# 21. Execute the Ansible Playbook

Enter the controller:

```powershell
docker exec -it ansible-controller bash
```

Execute:

```bash
ansible-playbook -i /root/hosts.ini /root/install_busybox.yml
```

### What happens during execution?

1. Ansible reads the YAML playbook.
2. Connects to client1 using SSH.
3. Gathers system facts.
4. Installs BusyBox through APT.
5. Executes the BusyBox verification command.
6. Stores its output.
7. Displays the verification result.

### Actual successful execution

```text
PLAY [Install BusyBox on Client 1]

TASK [Gathering Facts]
ok: [client1]

TASK [Install BusyBox]
changed: [client1]

TASK [Verify BusyBox installation]
ok: [client1]

TASK [Display BusyBox verification]
ok: [client1]

PLAY RECAP
client1 : ok=4 changed=1 unreachable=0 failed=0
```

The output confirms that BusyBox was installed successfully.

---

# 22. Verify Idempotency

Idempotency is one of the most important concepts in Ansible.

It means that running the same automation multiple times should produce the same desired system state without unnecessary changes.

### First execution

```bash
ansible-playbook -i /root/hosts.ini /root/install_busybox.yml
```

Result:

```text
client1 : ok=4 changed=1 unreachable=0 failed=0
```

Explanation:

- BusyBox was not previously installed.
- Ansible installed the package.
- One task changed the managed system.

### Second execution

Run the same command again:

```bash
ansible-playbook -i /root/hosts.ini /root/install_busybox.yml
```

Actual result:

```text
client1 : ok=4 changed=0 unreachable=0 failed=0
```

Explanation:

- BusyBox was already installed.
- Ansible checked the current system state.
- No unnecessary installation occurred.
- No system changes were required.

### Understanding the PLAY RECAP

| Field | Meaning |
|---|---|
| `ok` | Number of successful tasks |
| `changed` | Number of tasks that changed the system |
| `unreachable` | Number of hosts Ansible could not reach |
| `failed` | Number of failed tasks |
| `skipped` | Number of skipped tasks |
| `rescued` | Number of tasks recovered through rescue blocks |
| `ignored` | Number of failures ignored |

**Why was `ok=4` when our playbook contains only three tasks?**

Because Ansible also performed the automatic `Gathering Facts` task.

Therefore:

- 1 fact-gathering task.
- 3 explicitly defined tasks.
- Total: 4 successful tasks.

The difference between the two executions demonstrates idempotency.

---

# 23. Problems Encountered and Their Solutions

This section records the actual issues encountered during implementation.

| Problem | Cause | Solution |
|---|---|---|
| `Could not resolve hostname client1` | Containers were not on a shared user-defined network | Created `ansible-network` and connected all containers |
| SSH hostname resolution failure | Docker DNS could not resolve the client name | Verified using `getent hosts client1` |
| SSH authentication failure risk | Public key must be authorized on the remote user account | Copied public key into `authorized_keys` |
| SSH permission issues | Incorrect `.ssh` or `authorized_keys` permissions can prevent authentication | Used `chmod 700` and `chmod 600` |
| Inventory unavailable inside controller | Windows and controller have separate filesystems | Used `docker cp` |
| Nginx did not automatically start | Container did not run systemd and package startup was restricted | Started Nginx directly using `nginx` |
| `curl: No such file or directory` | curl was not installed on client1 | Installed using the Ansible APT module |
| Repeated playbook execution | Needed to confirm that automation avoids unnecessary changes | Executed the playbook twice and checked `changed=0` |

---

# 24. Important Ansible Concepts Learned

### Controller and Managed Nodes

The controller is the machine where Ansible runs. Managed nodes are the remote systems controlled by the controller.

### Inventory

The inventory contains the list of managed machines, their groups and connection details.

### Ad-hoc Commands

Used for quick, individual operations.

Example:

```bash
ansible all -i /root/hosts.ini -m ping
```

### Playbooks

YAML files that define multiple automation tasks.

Example:

```bash
ansible-playbook -i /root/hosts.ini /root/install_busybox.yml
```

### Modules

Modules perform specific operations on managed nodes.

Examples:

- `apt`: Package management.
- `command`: Executes commands.
- `copy`: Copies files.
- `file`: Manages files and directories.
- `service`: Manages services on systems with supported service managers.
- `debug`: Displays information.

### Idempotency

Ensures repeated automation runs do not make unnecessary changes when the desired state is already achieved.

### SSH Key Authentication

Allows the controller to authenticate with remote machines using a private/public key pair.

---

# 25. Scaling the Lab to 70 Physical Systems

The same controller-managed-node architecture can be extended to a larger lab.

For a 70-machine environment:

1. Install or configure Ubuntu on the managed systems.
2. Enable and configure the SSH server.
3. Create a dedicated Ansible user.
4. Configure SSH public-key authentication.
5. Ensure the controller can reach all managed machines over the LAN.
6. Configure the inventory with their IP addresses or DNS names.
7. Test connectivity with the Ansible ping module.
8. Execute playbooks against all systems or selected groups.

### Example inventory for a physical lab

```ini
[webservers]
pc01 ansible_host=192.168.1.101 ansible_user=ansible
pc02 ansible_host=192.168.1.102 ansible_user=ansible

[databases]
pc03 ansible_host=192.168.1.103 ansible_user=ansible

[applications]
pc04 ansible_host=192.168.1.104 ansible_user=ansible
```

Continue adding the remaining systems.

### Using Windows and WSL2

If the 70 machines must retain Windows:

1. Install WSL2.
2. Install Ubuntu inside WSL2.
3. Configure OpenSSH Server inside Ubuntu.
4. Configure networking so the controller can reach the WSL SSH service.
5. Configure Windows Firewall.
6. Test SSH from the controller.
7. Add the WSL machines to the Ansible inventory.

**Important:** WSL2's default NAT networking can complicate inbound SSH connections. Windows 11 mirrored networking can simplify connectivity on supported configurations. Test connectivity with two machines before scaling to all 70.

For a dedicated Linux lab, native Ubuntu installations are generally simpler to manage.

---

# 26. Final Project Result

Successfully built an Ansible automation environment using Docker.

The implementation demonstrated how a centralized Ansible controller can communicate with multiple Ubuntu managed nodes through SSH, execute ad-hoc commands, install software, gather system information and automate configuration through YAML playbooks.

Nginx was successfully installed and tested on the web server container using HTTP verification. A BusyBox playbook was created and executed successfully, and idempotency was demonstrated through repeated execution.

### Final achievements

- Created one Ansible controller.
- Created three Ubuntu managed nodes.
- Configured a shared Docker network.
- Established SSH key-based authentication.
- Created a grouped Ansible inventory.
- Executed commands across multiple nodes.
- Installed and tested Nginx.
- Installed BusyBox using a YAML playbook.
- Verified successful playbook execution.
- Demonstrated idempotency.

**Final conclusion:**

This practical lab provided hands-on experience with Ansible architecture, SSH connectivity, Docker networking, inventory management, ad-hoc commands, modules, playbooks and idempotent infrastructure automation.

---

# 27. Quick Command Reference

## Docker commands

```powershell
docker --version

docker ps

docker build -t ansible-controller:v1 ./controller

docker build -t ansible-managed-node:v1 ./managed-nodes

docker run -d --name ansible-controller ansible-controller:v1

docker run -d --name client1 ansible-managed-node:v1

docker run -d --name client2 ansible-managed-node:v1

docker run -d --name client3 ansible-managed-node:v1
```

## Docker networking

```powershell
docker network create ansible-network

docker network connect ansible-network ansible-controller

docker network connect ansible-network client1

docker network connect ansible-network client2

docker network connect ansible-network client3

docker network inspect ansible-network
```

## SSH commands

```powershell
ssh-keygen -t ed25519 -f .\ansible_key -N '""'

docker cp .\ansible_key ansible-controller:/root/ansible_key

docker cp .\ansible_key.pub client1:/root/.ssh/authorized_keys
```

Inside the controller:

```bash
ssh -i /root/ansible_key root@client1

ssh -i /root/ansible_key root@client2

ssh -i /root/ansible_key root@client3
```

## Inventory commands

```bash
ansible all -i /root/hosts.ini --list-hosts

ansible webservers -i /root/hosts.ini --list-hosts

ansible databases -i /root/hosts.ini --list-hosts

ansible applications -i /root/hosts.ini --list-hosts
```

## Ad-hoc commands

```bash
ansible all -i /root/hosts.ini -m ping

ansible all -i /root/hosts.ini -m command -a "hostname"

ansible all -i /root/hosts.ini -m command -a "whoami"

ansible all -i /root/hosts.ini -m command -a "df -h"

ansible all -i /root/hosts.ini -m command -a "free -h"

ansible all -i /root/hosts.ini -m command -a "cat /etc/os-release"
```

## Nginx commands

```bash
ansible webservers -i /root/hosts.ini -m apt -a "name=nginx state=present update_cache=yes"

ansible webservers -i /root/hosts.ini -m command -a "nginx -v"

ansible webservers -i /root/hosts.ini -m command -a "nginx"

ansible webservers -i /root/hosts.ini -m command -a "pgrep -a nginx"

ansible webservers -i /root/hosts.ini -m apt -a "name=curl state=present update_cache=yes"

ansible webservers -i /root/hosts.ini -m command -a "curl -I http://localhost"
```

## Playbook commands

```powershell
docker cp .\install_busybox.yml ansible-controller:/root/install_busybox.yml
```

Inside the controller:

```bash
ansible-playbook -i /root/hosts.ini /root/install_busybox.yml
```

Run the same command again to verify idempotency.

---

**End of Documentation**

Project: Ansible Automation Lab Using Docker

Environment: Windows + Docker Desktop + Ubuntu 24.04 + Ansible

Managed Nodes: client1, client2, client3