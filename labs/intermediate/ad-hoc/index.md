# Ad-Hoc Ansible Commands - Intermediate Lab

## The Scenario

A security incident has been reported: an unauthorized user account `rogue_admin` was discovered on several production systems. The security team needs us to act fast. We must investigate all systems in our environment, remove the unauthorized account, lock down SSH access, collect forensic evidence, and harden the systems — all using ad-hoc commands.

You'll also set up a monitoring user account for the NOC team, deploy a Message of the Day (MOTD) warning banner, and verify that critical security services are properly configured.

This lab builds on the basics of ad-hoc commands by introducing more modules, command chaining, and real-world troubleshooting patterns.

### Prerequisites

On the Ansible Controller, create a new lab directory:

```bash
mkdir -p ~/lab-ad-hoc-intermediate
cd ~/lab-ad-hoc-intermediate
```

Inside the `lab-ad-hoc-intermediate` folder, create an `inventory` file with the following:

```ini
[webservers]
node1 ansible_host=<IP of TargetNode-1 from /home/ansible/inventory/inventory.yaml>

[dbservers]
node2 ansible_host=<IP of TargetNode-2 from /home/ansible/inventory/inventory.yaml>

[allservers:children]
webservers
dbservers
```

> **Note:** The `allservers` group uses the `:children` suffix to include other groups as members. This is a powerful inventory pattern you'll use often.

---

## Part 1: Investigate the Systems

Before making changes, good practice is to gather information. Let's use the `setup` module to collect facts about our hosts.

### Gather basic facts

```bash
ansible -i inventory allservers -m setup -a "filter=ansible_hostname"
```

This returns only the hostname fact. The `filter` parameter is useful when you need specific information without the full fact dump.

### Check for the unauthorized user account

Use the `getent` module to look up the `rogue_admin` user in the system's password database:

```bash
ansible -i inventory allservers -b -m getent -a "database=passwd key=rogue_admin"
```

> **What happened?** If the user doesn't exist yet, you'll see a failure. That's expected — the `getent` module returns an error when the key isn't found. We'll create the user first to simulate the incident, then remove it.
>
> **Why `getent` instead of `command`?** The `getent` module queries the system's name service databases directly and returns structured data. It's the Ansible-native way to look up users, groups, hosts, and other NSS databases — no need to shell out to `grep`.

### Simulate the incident — create the rogue account

```bash
ansible -i inventory allservers -b -m user -a "name=rogue_admin shell=/bin/bash groups=wheel state=present"
```

Now verify the account exists:

```bash
ansible -i inventory allservers -b -m getent -a "database=passwd key=rogue_admin"
```

You should see output showing the user's password database entry (UID, GID, home directory, shell) on both nodes.

---

## Part 2: Contain the Threat

### Remove the unauthorized user and their home directory

```bash
ansible -i inventory allservers -b -m user -a "name=rogue_admin state=absent remove=yes"
```

The `remove=yes` parameter deletes the user's home directory and mail spool — important for cleanup.

### Verify the user is gone

```bash
ansible -i inventory allservers -b -m getent -a "database=passwd key=rogue_admin"
```

You should now see a failure on each host — the `getent` module fails when the user is not found, which confirms the removal worked.

### Kill any lingering processes

Use the `shell` module (not `command`) because we need a pipeline:

```bash
ansible -i inventory allservers -b -m shell -a "ps aux | grep rogue_admin | grep -v grep || echo 'No processes found'"
```

> **Key Concept:** The `shell` module supports pipes (`|`), redirects (`>`), and other shell features. The `command` module does not. Use `command` when possible (it's safer), and `shell` only when you need shell features.
>
> **Alternative:** In a playbook, you could use the `community.general.pids` module to find processes by pattern, which returns structured data without shell pipe gymnastics. For quick ad-hoc investigation, `shell` is acceptable here.

---

## Part 3: Collect Forensic Evidence

### Check recent login activity

```bash
ansible -i inventory allservers -b -m command -a "last -10"
```

### Save the auth log to a local evidence directory

First, create a local directory for evidence:

```bash
mkdir -p ~/lab-ad-hoc-intermediate/evidence
```

Use the `fetch` module to pull log files from remote hosts:

```bash
ansible -i inventory allservers -b -m fetch -a "src=/var/log/secure dest=~/lab-ad-hoc-intermediate/evidence/ flat=no"
```

> **New Module: `fetch`** — The opposite of `copy`. It pulls files *from* remote hosts *to* the controller. With `flat=no`, files are organized into subdirectories by hostname.

Verify the files were retrieved:

```bash
ls -R ~/lab-ad-hoc-intermediate/evidence/
```

You should see a directory per host, each containing the `secure` log.

---

## Part 4: Harden the Systems

### Create the NOC monitoring user

```bash
ansible -i inventory allservers -b -m user -a "name=noc_monitor comment='NOC Monitoring Account' shell=/bin/bash state=present"
```

### Set up SSH key authentication for the NOC user

Generate a key pair locally:

```bash
mkdir -p ~/lab-ad-hoc-intermediate/keys
ssh-keygen -f ~/lab-ad-hoc-intermediate/keys/noc_monitor_rsa -N "" -C "noc_monitor@company.com"
```

Create the `.ssh` directory on all hosts:

```bash
ansible -i inventory allservers -b -m file -a "path=/home/noc_monitor/.ssh state=directory owner=noc_monitor group=noc_monitor mode=0700"
```

Deploy the public key:

```bash
ansible -i inventory allservers -b -m copy -a "src=~/lab-ad-hoc-intermediate/keys/noc_monitor_rsa.pub dest=/home/noc_monitor/.ssh/authorized_keys owner=noc_monitor group=noc_monitor mode=0600"
```

> **Note:** We set the `.ssh` directory to `0700` and the `authorized_keys` file to `0600`. SSH is strict about these permissions — if they're too open, SSH silently ignores the keys.

### Deploy a warning banner

Use the `copy` module with the `content` parameter to create a file without needing a source:

```bash
ansible -i inventory allservers -b -m copy -a "content='WARNING: Authorized users only. All activity is monitored and logged.\n' dest=/etc/motd"
```

> **New Trick:** The `content` parameter lets you write inline text to a file without needing a local source file. Very handy for quick configuration.

### Ensure security services are running

Verify `sshd` is running on all hosts:

```bash
ansible -i inventory allservers -b -m service -a "name=sshd state=started enabled=yes"
```

Install and start `auditd` on all hosts:

```bash
ansible -i inventory allservers -b -m yum -a "name=audit state=present"
ansible -i inventory allservers -b -m service -a "name=auditd state=started enabled=yes"
```

---

## Part 5: Verify and Report

### Check that security configuration is correct

Verify the `noc_monitor` user exists on all hosts:

```bash
ansible -i inventory allservers -b -m getent -a "database=passwd key=noc_monitor"
```

Verify the MOTD is in place:

```bash
ansible -i inventory allservers -m slurp -a "src=/etc/motd"
```

> **Note:** The `slurp` module returns file contents as base64-encoded data. In ad-hoc mode the output won't be human-readable — this is a trade-off for using the Ansible-native file reading module. In a playbook, you'd decode it with the `b64decode` filter.

Verify running services using the `systemd` module:

```bash
ansible -i inventory allservers -b -m systemd -a "name=sshd"
ansible -i inventory allservers -b -m systemd -a "name=auditd"
```

> **Why two commands?** Each `systemd` call checks one service and returns structured data (state, enabled status, etc.) — no shell operators needed. This is more reliable than parsing shell output.

### Check disk usage across all systems

Use the `setup` module to gather mount/disk information:

```bash
ansible -i inventory allservers -m setup -a "filter=ansible_mounts"
```

> **Why `setup` instead of `df`?** The `setup` module returns structured data about all mounts, including size, used space, and available space. This makes it easy to process programmatically in playbooks.

### Target only web servers

Use the `systemd` module against just the `webservers` group to see how group targeting works:

```bash
ansible -i inventory webservers -b -m systemd -a "name=httpd"
```

> **Note:** This may show httpd is not installed yet — that's fine. The key takeaway is that you can target specific groups for specific tasks.

---

## Bonus Challenges

Try these on your own using ad-hoc commands:

1. **Create a cron job** on all servers that runs a disk space check every hour:
   ```bash
   ansible -i inventory allservers -b -m cron -a "name='disk check' minute=0 job='df -h / >> /var/log/disk_usage.log'"
   ```

2. **Set a sysctl parameter** to disable ICMP redirects (a common hardening step):
   ```bash
   ansible -i inventory allservers -b -m sysctl -a "name=net.ipv4.conf.all.accept_redirects value=0 state=present reload=yes"
   ```

3. **Remove the cron job** you created:
   ```bash
   ansible -i inventory allservers -b -m cron -a "name='disk check' state=absent"
   ```

---

## Module Reference

Here's a summary of all the modules used in this lab:

| Module    | Purpose                                      |
|-----------|----------------------------------------------|
| `setup`   | Gather system facts (hostname, mounts, etc.) |
| `getent`  | Query NSS databases (users, groups, hosts)   |
| `slurp`   | Read file contents from remote hosts         |
| `systemd` | Manage and query systemd services            |
| `shell`   | Run commands (with pipes, redirects, etc.)   |
| `command` | Run commands (no shell features)             |
| `user`    | Manage user accounts                         |
| `file`    | Manage files and directories                 |
| `copy`    | Copy files to remote hosts or write content  |
| `fetch`   | Pull files from remote hosts to controller   |
| `service` | Manage system services                       |
| `yum`     | Manage packages                              |
| `cron`    | Manage cron jobs                             |
| `sysctl`  | Manage kernel parameters                     |

## Conclusion

In this lab you responded to a simulated security incident entirely with ad-hoc commands. You investigated systems, removed a rogue user, collected evidence, created and secured a monitoring account, deployed a warning banner, and hardened services. Along the way you learned to prefer Ansible-native modules over shelling out — using `getent` to look up users, `systemd` to check services, and `setup` to gather disk information. You also used `fetch` to pull remote files, `slurp` to read file contents, and `copy` with inline content. These are the same patterns you'll use in real incident response — the difference is that in production, you'll want to capture these steps in a playbook for repeatability.
