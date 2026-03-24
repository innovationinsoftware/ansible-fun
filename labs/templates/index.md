# Ansible Templates - Intro Lab

## Scenario

A colleague was the unfortunate victim of a scam email, and their network account was compromised. We have been tasked with deploying a hardened `sudoers` file. We need to create an Ansible template of the `sudoers` file.

We also need to create an accompanying playbook named `security.yml` that will deploy this template to all servers using the Ansible CLI.

## Prerequisites

This lab assumes you have completed the previous labs and have:
- SSH access to the Ansible Controller
- An inventory file with your managed nodes
- Ansible installed and working on the controller

## Set Up Lab Directory

On the Ansible Controller, create the working directory:

```bash
mkdir -p ~/lab-templates/templates-lab
cd ~/lab-templates
```

## Create the Inventory

Create an `inventory` file with `web` and `database` groups:

```
[web]
node1 ansible_host=<IP of TargetNode-1 from /home/ansible/inventory/inventory.yaml>

[database]
node2 ansible_host=<IP of TargetNode-2 from /home/ansible/inventory/inventory.yaml>
```

## Create the Template and Playbook

### Create the Jinja2 Template

Inside the `templates-lab` directory, create a file named `hardened.j2` with the following content:

```jinja2
%sysops {{ ansible_default_ipv4.address }} = (ALL) ALL
Host_Alias WEBSERVERS = {{ groups['web']|join(' ') }}
Host_Alias DBSERVERS = {{ groups['database']|join(' ') }}
%httpd WEBSERVERS = /bin/su - webuser
%dba DBSERVERS = /bin/su - dbuser
```

### Create the Playbook

In the `templates-lab` directory, create a playbook named `security.yml` with the following:

```yaml
---
- hosts: all
  become: yes
  tasks:
    - name: Create directory for template deployment
      file:
        path: /home/ansible/lab-templates
        state: directory
        mode: '0755'

    - name: Deploy sudo template
      template:
        src: hardened.j2
        dest: /etc/sudoers.d/hardened
        mode: '0440'
        validate: /sbin/visudo -cf %s

    - name: Display success message
      debug:
        msg: "Hardened sudoers template deployed successfully"
```

## Run the Playbook

Run the playbook from the command line:

```bash
cd ~/lab-templates
ansible-playbook -i inventory templates-lab/security.yml
```

The output should show:
- Directory creation
- Template deployment with validation
- Success message for each host

## Verify the Deployment

### Using Ad-Hoc Commands

Verify the deployed file on all nodes:

```bash
ansible -i inventory all -b -m shell -a "cat /etc/sudoers.d/hardened"
```

You should see the customized sudoers content with:
- The actual IP addresses of your nodes
- Properly populated host aliases for WEBSERVERS and DBSERVERS groups

### Expected Output

The deployed file should contain something like:
```
%sysops 10.0.1.10 = (ALL) ALL
Host_Alias WEBSERVERS = node1
Host_Alias DBSERVERS = node2
%httpd WEBSERVERS = /bin/su - webuser
%dba DBSERVERS = /bin/su - dbuser
```

*(IP addresses will match your actual node IPs)*

## Understanding the Template

Let's break down what the template accomplished:

- **`{{ ansible_default_ipv4.address }}`**: This Ansible fact gets replaced with each host's primary IP address
- **`{{ groups['web']|join(' ') }}`**: This creates a space-separated list of all hosts in the 'web' group
- **`{{ groups['database']|join(' ') }}`**: This creates a space-separated list of all hosts in the 'database' group
- **Template validation**: The `validate` parameter ensures the sudoers syntax is correct before deployment
- **Jinja2 filters**: The `join(' ')` filter converts the host list into a space-separated string

## Test Template Customization

Let's verify that the template is truly dynamic by checking different hosts:

```bash
ansible -i inventory all -m setup -a "filter=ansible_default_ipv4"
```

Compare the IPs shown in the setup module with those in the deployed sudoers file.

Each host should have its own IP address in the sudoers file, demonstrating the power of Ansible templates.

## Conclusion

Congratulations! You have successfully:
- Created a Jinja2 template using Ansible facts and group variables
- Built a playbook that deploys templates with syntax validation
- Used the Ansible CLI to deploy customized configuration files
- Verified that templates generate host-specific content
- Understood how Jinja2 filters work with Ansible group data

This demonstrates how Ansible templates enable you to maintain consistent configuration patterns while allowing for host-specific customization - a crucial skill for managing diverse infrastructure at scale.
