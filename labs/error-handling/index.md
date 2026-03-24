# Ansible Error Handling

## Scenario

We have to set up an automation to pull a data file from a notoriously unreliable third-party system for integration. Create a playbook that attempts to download https://bit.ly/3dtJtR7 and save it as `transaction_list` to `localhost`. The playbook should gracefully handle the site being down by outputting the message "Site appears to be down. Try again later." to stdout. If the task succeeds, the playbook should write "File downloaded." to stdout. Regardless of whether the playbook errors, it should always output "Attempt completed." to stdout.

If the report is collected, the playbook should write and edit the file to replace all occurrences of `#BLANKLINE` with a line break '\n'.

### Prerequisites

On the Ansible Controller, create a new lab directory:

```bash
mkdir -p ~/lab-error-handling
cd ~/lab-error-handling
```

Create an `inventory` file:

```
[all]
node1 ansible_host=<IP of TargetNode-1 from /home/ansible/inventory/inventory.yaml>
node2 ansible_host=<IP of TargetNode-2 from /home/ansible/inventory/inventory.yaml>
```

## Set up maintenance script

The maintenance playbook simulates a service outage for testing. It is already available at:

```
/home/ansible/automation-dev/labs/error-handling/maint/break_stuff.yml
```

To simulate a down service (adds `127.0.0.1 bit.ly` to `/etc/hosts`):

```bash
ansible-playbook -i inventory /home/ansible/automation-dev/labs/error-handling/maint/break_stuff.yml --tags service_down
```

To restore the service:

```bash
ansible-playbook -i inventory /home/ansible/automation-dev/labs/error-handling/maint/break_stuff.yml --tags service_up
```

## Create the playbook

Create a playbook named `report.yml`.

First, we'll specify our **host** and **tasks** (**name**, and **debug** message):

```yaml
---
- hosts: all
  tasks:
    - name: download transaction_list
      block:
        - file:
            path: /home/ansible/lab-error-handling
            state: directory
            mode: '0755'
        - get_url:
            url: https://bit.ly/3dtJtR7
            dest: /home/ansible/lab-error-handling/transaction_list
        - debug: msg="File downloaded"
```

### Add connection failure logic

We need to reconfigure a bit here, adding a **block** keyword and a **rescue**, in case the URL we're reaching out to is down:

```yaml
---
- hosts: all
  tasks:
    - name: download transaction_list
      block:
        - file:
            path: /home/ansible/lab-error-handling
            state: directory
            mode: '0755'
        - get_url:
            url: https://bit.ly/3dtJtR7
            dest: /home/ansible/lab-error-handling/transaction_list
        - debug: msg="File downloaded"
      rescue:
        - debug: msg="Site appears to be down.  Try again later."
```

### Add an always message

An **always** block here will let us know that the playbook at least made an attempt to download the file:

```yaml
---
- hosts: all
  tasks:
    - name: download transaction_list
      block:
        - file:
            path: /home/ansible/lab-error-handling
            state: directory
            mode: '0755'
        - get_url:
            url: https://bit.ly/3dtJtR7
            dest: /home/ansible/lab-error-handling/transaction_list
        - debug: msg="File downloaded"
      rescue:
        - debug: msg="Site appears to be down.  Try again later."
      always:
        - debug: msg="Attempt completed."
```

### Replace '#BLANKLINE' with '\n'

We can use the **replace** module for this task, and we'll sneak it in between the **get_url** and first **debug** tasks.

```yaml
---
- hosts: all
  tasks:
    - name: download transaction_list
      block:
        - file:
            path: /home/ansible/lab-error-handling
            state: directory
            mode: '0755'
        - get_url:
            url: https://bit.ly/3dtJtR7
            dest: /home/ansible/lab-error-handling/transaction_list
        - replace:
            path: /home/ansible/lab-error-handling/transaction_list
            regexp: "#BLANKLINE"
            replace: '\n'
        - debug: msg="File downloaded"
      rescue:
        - debug: msg="Site appears to be down.  Try again later."
      always:
        - debug: msg="Attempt completed."
```

## Run the playbook

Run the playbook from the command line:

```bash
cd ~/lab-error-handling
ansible-playbook -i inventory report.yml
```

You should see successful output showing "File downloaded" and "Attempt completed."

### Simulate a failure

Now, let's simulate a down service:

```bash
ansible-playbook -i inventory /home/ansible/automation-dev/labs/error-handling/maint/break_stuff.yml --tags service_down
```

After it completes, run the error handling playbook again:

```bash
ansible-playbook -i inventory report.yml
```

You'll see the output shows the rescue block triggered with "Site appears to be down. Try again later."

Finally, restore the service and verify it works again:

```bash
ansible-playbook -i inventory /home/ansible/automation-dev/labs/error-handling/maint/break_stuff.yml --tags service_up
ansible-playbook -i inventory report.yml
```

### Bonus

Use ad-hoc to confirm the `/home/ansible/lab-error-handling/transaction_list` file was created and is formatted correctly:

```bash
ansible -i inventory all -b -m shell -a "cat /home/ansible/lab-error-handling/transaction_list"
```

## Congrats!
