# Ubuntu Studio Software Deployment with SSH and Ansible

This repository documents how to set up SSH access to an Ubuntu Studio machine, configure Ansible from a Mac, allow Ansible to run sudo commands without repeatedly entering a password, and install audio software using Ansible playbooks.

The setup used here assumes:

```text
Control machine: macOS
Target machine: Ubuntu Studio
Ubuntu user: l42i
Ansible inventory group: studio
```

## Repository files

```text
.
├── inventory.ini
├── ansible.cfg
├── install_reaper.yaml
├── install_superCollider.yml
└── install_iem.yml
```

Note: the uploaded config file is named `anisble.cfg`. For Ansible to automatically detect it, rename it to:

```bash
mv anisble.cfg ansible.cfg
```

## 1. Enable SSH access on the Ubuntu system

Ansible connects to the Ubuntu machine over SSH. First, install and enable the SSH server on Ubuntu.

On the Ubuntu Studio machine, run:

```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

Check that SSH is running:

```bash
sudo systemctl status ssh
```

Find the Ubuntu machine's IP address:

```bash
ip addr
```

Look for the local network IP address, for example:

```text
192.168.2.2
```

From the Mac, test that SSH works:

```bash
ssh l42i@192.168.2.2
```

If this opens a terminal session on the Ubuntu machine, SSH is working.

## 2. Set up Ansible on the Mac

Install Ansible on macOS. If using Homebrew:

```bash
brew install ansible
```

Check that Ansible is installed:

```bash
ansible --version
```

Create an inventory file named `inventory.ini`:

```ini
[studio]
192.168.2.2 ansible_user=l42i ansible_python_interpreter=/usr/bin/python3
```

This tells Ansible:

```text
studio = the group name for the Ubuntu machine
192.168.2.2 = the Ubuntu machine's IP address
ansible_user=l42i = connect as the l42i user
ansible_python_interpreter=/usr/bin/python3 = use Python 3 on Ubuntu
```

Test the connection:

```bash
ansible -i inventory.ini studio -m ping
```

Expected result:

```text
192.168.2.2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## 3. Configure Ansible defaults

Create or rename the Ansible config file as `ansible.cfg`:

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
timeout = 30

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = True
```

What this does:

```text
inventory = inventory.ini
```

Makes Ansible use `inventory.ini` by default.

```text
host_key_checking = False
```

Prevents Ansible from stopping on first SSH connection because of host key confirmation prompts.

```text
timeout = 30
```

Gives SSH connections 30 seconds before timing out.

```text
become = True
```

Makes Ansible use privilege escalation by default.

```text
become_method = sudo
```

Uses `sudo` to become root.

```text
become_ask_pass = True
```

Prompts for the sudo password when needed.

After passwordless sudo is configured, this can be changed to:

```ini
become_ask_pass = False
```

## 4. Ensure sudo commands can run from Ansible

Many software installation tasks require root access. In Ansible, this is done with:

```yaml
become: yes
```

Before making sudo passwordless, test sudo manually from the Mac:

```bash
ssh l42i@192.168.2.2
sudo whoami
```

Expected output:

```text
root
```

Then test sudo through Ansible:

```bash
ansible -i inventory.ini studio -m command -a "whoami" -b -K
```

Where:

```text
-b = become root using sudo
-K = ask for the sudo password
```

Expected output:

```text
192.168.2.2 | CHANGED | rc=0 >>
root
```

If this returns `root`, Ansible can use sudo correctly.

## 5. Make sudo passwordless for Ansible

To avoid typing the sudo password every time Ansible runs a playbook, allow the `l42i` user to run sudo without a password.

On the Ubuntu machine, run:

```bash
sudo visudo
```

Add this line at the bottom:

```text
l42i ALL=(ALL) NOPASSWD:ALL
```

Save and exit.

Now test from the Mac:

```bash
ansible -i inventory.ini studio -m command -a "whoami" -b
```

Expected output:

```text
192.168.2.2 | CHANGED | rc=0 >>
root
```

If that works without asking for a password, passwordless sudo is working.

You can now update `ansible.cfg`:

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
timeout = 30

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = False
```

Security note: `NOPASSWD:ALL` is convenient for lab deployment, but it gives the `l42i` user passwordless sudo for all commands. Only use this on trusted lab machines and trusted user accounts.

## 6. Playbook: Install REAPER

File:

```text
install_reaper.yaml
```

Run it with:

```bash
ansible-playbook install_reaper.yaml
```

This playbook installs REAPER from a local tar file on the Mac.

Important variables:

```yaml
reaper_tar_local: /Users/adityarpawar/Desktop/Summer2026/l42i/reaper_install/reaper774_linux_x86_64.tar.xz
reaper_remote_dir: /home/l42i/Desktop/reaper_install
reaper_tar_remote: /home/l42i/Desktop/reaper_install/reaper774_linux_x86_64.tar.xz
reaper_install_dir: /opt
```

What each variable means:

```text
reaper_tar_local
```

Path to the REAPER `.tar.xz` installer on the Mac.

```text
reaper_remote_dir
```

Folder that will be created on the Ubuntu Desktop.

```text
reaper_tar_remote
```

Where the REAPER installer will be copied on Ubuntu.

```text
reaper_install_dir
```

System install location for REAPER. This playbook installs into `/opt`.

Tasks explained:

```yaml
- name: Create REAPER install folder on Ubuntu Desktop
```

Creates `/home/l42i/Desktop/reaper_install` on the Ubuntu machine.

```yaml
- name: Copy REAPER tar file from Mac to Ubuntu Desktop
```

Copies the REAPER installer from the Mac to Ubuntu.

```yaml
- name: Extract REAPER tar file on Ubuntu
```

Extracts the `.tar.xz` file on Ubuntu.

```yaml
- name: Find REAPER install script
```

Searches inside the extracted folder for `install-reaper.sh`.

```yaml
- name: Install REAPER system-wide
```

Runs the REAPER install script with these options:

```text
--install /opt
```

Installs REAPER into `/opt`.

```text
--integrate-desktop
```

Adds desktop integration, such as menu entries.

```text
--usr-local-bin-symlink
```

Creates a command line symlink so `reaper` can be launched from the terminal.

The task uses:

```yaml
creates: /opt/REAPER/reaper
```

This prevents Ansible from reinstalling REAPER if it is already installed.

Verify REAPER after installation:

```bash
ansible -i inventory.ini studio -m command -a "which reaper"
```

Expected output:

```text
/usr/local/bin/reaper
```

## 7. Playbook: Install SuperCollider

File:

```text
install_superCollider.yml
```

Run it with:

```bash
ansible-playbook install_superCollider.yml
```

This playbook installs SuperCollider and related packages through Ubuntu's package manager.

Packages installed:

```yaml
- supercollider
- supercollider-server
- supercollider-ide
- supercollider-language
- supercollider-supernova
- sc3-plugins
```

Tasks explained:

```yaml
- name: Update apt package cache
```

Runs the equivalent of:

```bash
sudo apt update
```

The `cache_valid_time: 3600` setting means Ansible will not refresh the package cache again if it was already updated within the last hour.

```yaml
- name: Install SuperCollider
```

Installs SuperCollider, the server, the IDE, the language tools, Supernova, and SC3 plugins.

```yaml
- name: Verify sclang is installed
```

Runs:

```bash
which sclang
```

This confirms the SuperCollider language executable is installed.

```yaml
- name: Verify scsynth is installed
```

Runs:

```bash
which scsynth
```

This confirms the SuperCollider audio synthesis server is installed.

Expected paths:

```text
/usr/bin/sclang
/usr/bin/scsynth
```

## 8. Playbook: Install IEM Plug-in Suite

File:

```text
install_iem.yml
```

Run it with:

```bash
ansible-playbook install_iem.yml
```

This playbook installs the IEM Plug-in Suite from Ubuntu packages.

Packages installed:

```yaml
- iem-plugin-suite-vst
- iem-plugin-suite-vst3
- iem-plugin-suite-standalone
```

Tasks explained:

```yaml
- name: Update apt package cache
```

Updates the apt package cache.

```yaml
cache_valid_time: 3600
```

Avoids unnecessary package cache updates if the cache was refreshed recently.

```yaml
lock_timeout: 300
```

Tells Ansible to wait up to 300 seconds if apt is locked by another process, such as automatic updates.

```yaml
- name: Install IEM Plug-in Suite
```

Installs VST, VST3, and standalone versions of the IEM Plug-in Suite.

```yaml
- name: Verify IEM packages are installed
```

Runs:

```bash
dpkg -l | grep -i iem-plugin-suite
```

This checks the installed Debian packages and filters for IEM package names.

```yaml
- name: Show installed IEM packages
```

Prints the installed IEM package list in the Ansible output.

## 9. Common commands

Test SSH connection:

```bash
ssh l42i@192.168.2.2
```

Test Ansible ping:

```bash
ansible studio -m ping
```

Test Ansible sudo:

```bash
ansible studio -m command -a "whoami" -b
```

Run REAPER install:

```bash
ansible-playbook install_reaper.yaml
```

Run SuperCollider install:

```bash
ansible-playbook install_superCollider.yml
```

Run IEM install:

```bash
ansible-playbook install_iem.yml
```

Run a playbook with explicit inventory:

```bash
ansible-playbook -i inventory.ini install_superCollider.yml
```

## 10. Troubleshooting

### SSH connection fails

Check that the Ubuntu machine is on the same network as the Mac and that SSH is running:

```bash
sudo systemctl status ssh
```

Check the Ubuntu IP again:

```bash
ip addr
```

If the IP changed, update `inventory.ini`.

### Ansible cannot become root

Run:

```bash
ansible -i inventory.ini studio -m command -a "whoami" -b -K
```

If this fails, check that the `l42i` user can use sudo manually:

```bash
ssh l42i@192.168.2.2
sudo whoami
```

If manual sudo works but Ansible sudo does not, check the `[privilege_escalation]` section in `ansible.cfg`.

### Apt is locked

Sometimes Ubuntu automatic updates hold the apt lock. The IEM playbook already includes:

```yaml
lock_timeout: 300
```

This makes Ansible wait for the lock before failing.

### REAPER tar file not found

Make sure this path exists on the Mac:

```bash
/Users/adityarpawar/Desktop/Summer2026/l42i/reaper_install/reaper774_linux_x86_64.tar.xz
```

If the REAPER installer is in a different location, update this variable in `install_reaper.yaml`:

```yaml
reaper_tar_local: /path/to/reaper774_linux_x86_64.tar.xz
```

## 11. Suggested workflow for a fresh Ubuntu Studio machine

1. Install and start SSH on Ubuntu.
2. Find the Ubuntu IP address.
3. Update `inventory.ini` with the correct IP.
4. Test SSH from the Mac.
5. Test Ansible ping.
6. Test Ansible sudo with `-b -K`.
7. Configure passwordless sudo for `l42i`.
8. Test Ansible sudo without `-K`.
9. Run the software installation playbooks.

