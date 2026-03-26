# Ansible Roles - Intermediate Lab

## The Scenario

Your company is standardizing its server fleet. The operations team needs a repeatable, multi-role deployment that configures every server with a security baseline, then layers on role-specific configuration for web servers and database servers. The roles must be flexible enough to handle different environments (staging vs. production) using variable precedence, and the web role must depend on the baseline role so it is always applied first.

You will build three roles from scratch using `ansible-galaxy init`, wire them together with role dependencies and conditional logic, and use defaults, group variables, and handlers to create a production-ready automation pattern.

This lab builds on the basic roles lab by introducing `ansible-galaxy init`, role defaults vs. vars, role dependencies, conditionals inside roles, handlers in roles, and Galaxy collection roles.

### Prerequisites

On the Ansible Controller, create a new lab directory:

```bash
mkdir -p ~/lab-roles-intermediate
cd ~/lab-roles-intermediate
```

Create an `inventory` file:

```ini
[web]
node1 ansible_host=<IP of TargetNode-1 from /home/ansible/inventory/inventory.yaml>

[db]
node2 ansible_host=<IP of TargetNode-2 from /home/ansible/inventory/inventory.yaml>

[web:vars]
server_role=webserver

[db:vars]
server_role=database

[all:vars]
app_env=staging
company_name=Acme Corp
```

> **Note:** We define `server_role` per group and `app_env` globally. The roles will use these variables to customize their behavior per host.

---

## Part 1: Scaffold Roles with `ansible-galaxy init`

In the basic roles lab you created the directory structure by hand. In practice, `ansible-galaxy init` does this for you and creates a complete, best-practice layout.

### Create the roles directory and scaffold three roles

```bash
mkdir -p ~/lab-roles-intermediate/roles
cd ~/lab-roles-intermediate/roles
ansible-galaxy init security_baseline
ansible-galaxy init web_server
ansible-galaxy init db_server
```

### Explore the generated structure

```bash
tree security_baseline
```

If `tree` is not installed, use:

```bash
find security_baseline -type f | sort
```

You should see directories for `defaults`, `vars`, `handlers`, `tasks`, `templates`, `files`, `meta`, and `tests`. This is the standard Ansible role layout.

> **Key Concept: `ansible-galaxy init`** scaffolds a role with all standard directories and placeholder files. This saves time and enforces consistent structure across your team.

---

## Part 2: Build the Security Baseline Role

This role will apply common security configuration to every server: create a service account, deploy an MOTD banner, harden SSH settings, and ensure security packages are installed.

### Set role defaults

Role defaults are variables with the lowest precedence — they act as sensible fallbacks that can be overridden by inventory variables, playbook variables, or the role's own `vars`.

Edit `roles/security_baseline/defaults/main.yml`:

```yaml
---
security_user: svc_ansible
security_user_comment: "Ansible Service Account"
security_packages:
  - audit
  - chrony
motd_template: motd.j2
enable_firewalld: false
```

> **Key Concept: Defaults vs. Vars** — Variables in `defaults/main.yml` have the *lowest* precedence and are meant to be overridden. Variables in `vars/main.yml` have *higher* precedence and are meant for values that should rarely change. Use defaults for anything you expect users to customize.

### Create the MOTD template

Edit `roles/security_baseline/templates/motd.j2`:

```jinja2
*************************************************************
  Hostname : {{ ansible_hostname }}
  IP       : {{ ansible_default_ipv4.address }}
  Role     : {{ server_role | default('unknown') }}
  Env      : {{ app_env | default('unknown') }}
  Company  : {{ company_name | default('UNDEFINED') }}
  Managed by Ansible - Do not edit manually
*************************************************************
```

> **New Concept: `default()` filter** — The `default()` Jinja2 filter provides a fallback value if the variable is undefined. This prevents template failures when a variable is missing and makes your roles more portable.

### Create the tasks

Edit `roles/security_baseline/tasks/main.yml`:

```yaml
---
- name: Create service account
  user:
    name: "{{ security_user }}"
    comment: "{{ security_user_comment }}"
    shell: /bin/bash
    state: present

- name: Create .ssh directory for service account
  file:
    path: "/home/{{ security_user }}/.ssh"
    state: directory
    owner: "{{ security_user }}"
    group: "{{ security_user }}"
    mode: '0700'

- name: Install security packages
  yum:
    name: "{{ security_packages }}"
    state: present

- name: Start and enable chronyd
  service:
    name: chronyd
    state: started
    enabled: yes

- name: Start and enable auditd
  service:
    name: auditd
    state: started
    enabled: yes

- name: Deploy MOTD banner
  template:
    src: "{{ motd_template }}"
    dest: /etc/motd
    mode: '0644'

- name: Harden SSH - disable empty passwords
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitEmptyPasswords'
    line: 'PermitEmptyPasswords no'
  notify: restart sshd

- name: Harden SSH - set max auth tries
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?MaxAuthTries'
    line: 'MaxAuthTries 4'
  notify: restart sshd
```

> **Note:** The SSH hardening tasks use `notify` to trigger a handler. The handler will only run if the configuration actually changes — this avoids unnecessary SSH restarts.

### Create the handler

Edit `roles/security_baseline/handlers/main.yml`:

```yaml
---
- name: restart sshd
  service:
    name: sshd
    state: restarted
```

> **Key Concept: Handlers in roles** live in `handlers/main.yml` and are automatically available to all tasks within the role. You reference them by name in `notify` directives, just like in a playbook.

---

## Part 3: Build the Web Server Role with Dependencies

The web server role will install and configure Apache. It *depends* on the security baseline role, meaning Ansible will automatically run the baseline role first whenever the web role is applied.

### Define the dependency

Edit `roles/web_server/meta/main.yml` and add the dependency at the bottom:

```yaml
---
galaxy_info:
  author: your name
  description: Web server role with Apache
  license: MIT
  min_ansible_version: "2.9"
  platforms:
    - name: EL
      versions:
        - "9"

dependencies:
  - role: security_baseline
```

> **Key Concept: Role dependencies** in `meta/main.yml` tell Ansible to automatically apply other roles first. This guarantees that the security baseline is always in place before web-specific configuration runs — you never have to remember to include it manually.

### Set role defaults

Edit `roles/web_server/defaults/main.yml`:

```yaml
---
http_port: 80
doc_root: /var/www/html
web_packages:
  - httpd
  - mod_ssl
server_admin: "webmaster@{{ company_name | default('example.com') | lower | replace(' ', '') }}"
```

> **New Trick:** You can chain Jinja2 filters. Here `company_name` gets lowercased and spaces removed to create a valid email domain: `Acme Corp` becomes `webmaster@acmecorp`.

### Create the index page template

Edit `roles/web_server/templates/index.html.j2`:

```html
<!DOCTYPE html>
<html>
<head><title>{{ inventory_hostname }} - {{ app_env }}</title></head>
<body>
  <h1>{{ company_name }} - {{ inventory_hostname }}</h1>
  <table>
    <tr><td>Environment</td><td>{{ app_env }}</td></tr>
    <tr><td>Server IP</td><td>{{ ansible_default_ipv4.address }}</td></tr>
    <tr><td>OS</td><td>{{ ansible_distribution }} {{ ansible_distribution_version }}</td></tr>
    <tr><td>Server Admin</td><td>{{ server_admin }}</td></tr>
    <tr><td>Port</td><td>{{ http_port }}</td></tr>
  </table>
</body>
</html>
```

### Create the virtual host template

Edit `roles/web_server/templates/vhost.conf.j2`:

```apache
# Managed by Ansible - Do not edit
<VirtualHost *:{{ http_port }}>
    ServerName {{ inventory_hostname }}
    ServerAdmin {{ server_admin }}
    DocumentRoot {{ doc_root }}

    <Directory {{ doc_root }}>
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog /var/log/httpd/{{ inventory_hostname }}_error.log
    CustomLog /var/log/httpd/{{ inventory_hostname }}_access.log combined
</VirtualHost>
```

### Create the tasks

Edit `roles/web_server/tasks/main.yml`:

```yaml
---
- name: Install web server packages
  yum:
    name: "{{ web_packages }}"
    state: present

- name: Configure httpd to listen on correct port
  lineinfile:
    path: /etc/httpd/conf/httpd.conf
    regexp: '^Listen '
    line: "Listen {{ http_port }}"
  notify: restart httpd

- name: Deploy virtual host configuration
  template:
    src: vhost.conf.j2
    dest: /etc/httpd/conf.d/{{ inventory_hostname }}.conf
    mode: '0644'
  notify: restart httpd

- name: Deploy index page
  template:
    src: index.html.j2
    dest: "{{ doc_root }}/index.html"
    owner: apache
    group: apache
    mode: '0644'

- name: Deploy health check endpoint
  copy:
    content: '{"status": "ok", "host": "{{ inventory_hostname }}", "role": "{{ server_role }}", "env": "{{ app_env }}"}'
    dest: "{{ doc_root }}/health.json"
    owner: apache
    group: apache
    mode: '0644'

- name: Ensure httpd is started and enabled
  service:
    name: httpd
    state: started
    enabled: yes
```

### Create the handler

Edit `roles/web_server/handlers/main.yml`:

```yaml
---
- name: restart httpd
  service:
    name: httpd
    state: restarted
```

---

## Part 4: Build the Database Server Role

The database role will also depend on the security baseline and will install MariaDB with a pre-configured application database.

### Define the dependency

Edit `roles/db_server/meta/main.yml` and add at the bottom:

```yaml
---
galaxy_info:
  author: your name
  description: Database server role with MariaDB
  license: MIT
  min_ansible_version: "2.9"
  platforms:
    - name: EL
      versions:
        - "9"

dependencies:
  - role: security_baseline
```

### Set role defaults

Edit `roles/db_server/defaults/main.yml`:

```yaml
---
db_packages:
  - mariadb-server
  - mariadb
  - python3-PyMySQL
db_name: app_data
db_user: app_user
db_password: S3cureP@ss
db_socket: /var/lib/mysql/mysql.sock
```

### Create the tasks

Edit `roles/db_server/tasks/main.yml`:

```yaml
---
- name: Install database packages
  yum:
    name: "{{ db_packages }}"
    state: present

- name: Start and enable MariaDB
  service:
    name: mariadb
    state: started
    enabled: yes

- name: Create application database
  mysql_db:
    name: "{{ db_name }}"
    state: present
    login_unix_socket: "{{ db_socket }}"

- name: Create application user
  mysql_user:
    name: "{{ db_user }}"
    password: "{{ db_password }}"
    priv: "{{ db_name }}.*:ALL"
    host: localhost
    state: present
    login_unix_socket: "{{ db_socket }}"

- name: Create the inventory table
  community.mysql.mysql_query:
    login_db: "{{ db_name }}"
    query: >
      CREATE TABLE IF NOT EXISTS inventory (
        id INT AUTO_INCREMENT PRIMARY KEY,
        hostname VARCHAR(255) NOT NULL,
        server_role VARCHAR(50),
        environment VARCHAR(50),
        ip_address VARCHAR(45),
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      )
    login_unix_socket: "{{ db_socket }}"

- name: Register this host in the inventory table
  community.mysql.mysql_query:
    login_db: "{{ db_name }}"
    query: >
      INSERT INTO inventory (hostname, server_role, environment, ip_address)
      SELECT '{{ inventory_hostname }}', '{{ server_role }}', '{{ app_env }}', '{{ ansible_default_ipv4.address }}'
      WHERE NOT EXISTS (SELECT 1 FROM inventory WHERE hostname = '{{ inventory_hostname }}')
    login_unix_socket: "{{ db_socket }}"

- name: Verify database contents
  community.mysql.mysql_query:
    login_db: "{{ db_name }}"
    query: "SELECT * FROM inventory"
    login_unix_socket: "{{ db_socket }}"
  register: db_contents

- name: Display database contents
  debug:
    msg: "{{ db_contents.query_result }}"
```

---

## Part 5: Create the Site Playbook

Now bring it all together. The site playbook assigns roles to host groups and demonstrates how role defaults can be overridden at the playbook level.

Create `~/lab-roles-intermediate/site.yml`:

```yaml
---
- name: Configure web servers
  hosts: web
  become: yes
  roles:
    - role: web_server
      vars:
        http_port: 8080

- name: Configure database servers
  hosts: db
  become: yes
  roles:
    - db_server

- name: Verify all servers
  hosts: all
  become: yes
  tasks:
    - name: Gather service facts
      service_facts:

    - name: Verify auditd is running (from security_baseline)
      debug:
        msg: "auditd is {{ ansible_facts.services['auditd.service'].state }}"
      when: "'auditd.service' in ansible_facts.services"

    - name: Verify chronyd is running (from security_baseline)
      debug:
        msg: "chronyd is {{ ansible_facts.services['chronyd.service'].state }}"
      when: "'chronyd.service' in ansible_facts.services"

    - name: Read the MOTD
      slurp:
        src: /etc/motd
      register: motd_content

    - name: Display MOTD
      debug:
        msg: "{{ motd_content.content | b64decode }}"

    - name: Check service account exists
      getent:
        database: passwd
        key: svc_ansible

    - name: Show service account info
      debug:
        msg: "Service account UID: {{ getent_passwd['svc_ansible'][1] }}"
```

> **Key Concept: Overriding defaults at the playbook level** — The web play sets `http_port: 8080` which overrides the role default of `80`. This demonstrates how defaults provide flexibility: the role works out of the box, but consumers can customize it without editing role internals.

### Run the site playbook

```bash
cd ~/lab-roles-intermediate
ansible-playbook -i inventory site.yml
```

Watch the output carefully. You should see:

1. **Web play:** The `security_baseline` role runs first (because of the dependency), then `web_server` tasks run
2. **DB play:** The `security_baseline` role runs first again for the db host, then `db_server` tasks run
3. **Verify play:** Confirms that the baseline role's configuration (auditd, chronyd, MOTD, service account) is in place on *all* hosts

> **Important:** Ansible skips duplicate dependency execution *within the same play*. Since the web and db plays target different hosts, the baseline runs once per host — not redundantly.

### Verify the web server

```bash
curl http://<node1_IP>:8080/
curl http://<node1_IP>:8080/health.json
```

The index page should show the company name, environment, and port 8080. The health endpoint should return JSON with the server role.

### Verify the database

```bash
ansible -i inventory db -b -m community.mysql.mysql_query -a "login_db=app_data query='SELECT * FROM inventory' login_unix_socket=/var/lib/mysql/mysql.sock"
```

---

## Part 6: Override Defaults with Group Variables

Now let's see variable precedence in action. We'll override the security baseline's default service account name for database servers only — without touching the role code.

### Create group_vars

```bash
mkdir -p ~/lab-roles-intermediate/group_vars
```

Create `~/lab-roles-intermediate/group_vars/db.yml`:

```yaml
---
security_user: svc_database
security_user_comment: "Database Service Account"
```

### Run only the database play

```bash
cd ~/lab-roles-intermediate
ansible-playbook -i inventory site.yml --limit db
```

### Verify the override worked

```bash
ansible -i inventory db -b -m getent -a "database=passwd key=svc_database"
```

You should see the `svc_database` user on node2. The web server (node1) still has `svc_ansible` because the group variable only applies to the `db` group.

> **Key Concept: Variable precedence in action** — The role default set `security_user: svc_ansible`. The group variable `group_vars/db.yml` overrides this for the `db` group only. This is how you customize shared roles for different environments without forking the role code. The precedence order (from lowest to highest): role defaults < inventory group_vars < inventory host_vars < playbook vars.

---

## Part 7: Cleanup

Create `~/lab-roles-intermediate/cleanup.yml`:

```yaml
---
- name: Cleanup web servers
  hosts: web
  become: yes
  tasks:
    - name: Stop and disable httpd
      service:
        name: httpd
        state: stopped
        enabled: no
      ignore_errors: yes

    - name: Remove web packages
      yum:
        name:
          - httpd
          - mod_ssl
        state: absent

    - name: Remove virtual host config
      file:
        path: "/etc/httpd/conf.d/{{ inventory_hostname }}.conf"
        state: absent

    - name: Remove web content
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - /var/www/html/index.html
        - /var/www/html/health.json

- name: Cleanup database servers
  hosts: db
  become: yes
  tasks:
    - name: Stop and disable MariaDB
      service:
        name: mariadb
        state: stopped
        enabled: no
      ignore_errors: yes

    - name: Remove database packages
      yum:
        name:
          - mariadb-server
          - mariadb
        state: absent

- name: Cleanup common resources
  hosts: all
  become: yes
  tasks:
    - name: Remove service accounts
      user:
        name: "{{ item }}"
        state: absent
        remove: yes
      loop:
        - svc_ansible
        - svc_database
      ignore_errors: yes

    - name: Confirm cleanup
      debug:
        msg: "Cleanup complete on {{ inventory_hostname }}"
```

> Only run this if you want to tear everything down:
> ```bash
> ansible-playbook -i inventory cleanup.yml
> ```

---

## Concept Summary

| Concept | What It Does | When to Use |
|---|---|---|
| `ansible-galaxy init` | Scaffolds a complete role directory structure | Starting any new role |
| `defaults/main.yml` | Lowest-precedence variables (easy to override) | Configurable values users should customize |
| `vars/main.yml` | Higher-precedence variables (hard to override) | Internal values that should rarely change |
| `meta/main.yml` dependencies | Auto-apply other roles first | Shared prerequisites (security, monitoring) |
| `handlers/main.yml` | Role-scoped handlers triggered by `notify` | Service restarts after config changes |
| `group_vars/` | Override role defaults per inventory group | Environment or group-specific customization |
| `default()` filter | Fallback value for undefined variables | Making templates portable |
| Chained filters | Transform values inline (`lower \| replace`) | Data formatting in templates |
| `--limit` | Run playbook against a subset of hosts | Testing changes on one group |
| `loop` | Iterate over a list in a task | Repeated operations on multiple items |

## Conclusion

In this lab you built a multi-role deployment using production patterns. You used `ansible-galaxy init` to scaffold roles instead of creating directories by hand. You learned the difference between role defaults (easily overridden) and role vars (locked down), and demonstrated variable precedence by customizing the security baseline for database servers using group variables — without modifying the role itself. Role dependencies in `meta/main.yml` ensured the security baseline was automatically applied before any application role. Handlers inside roles kept service restarts efficient, and Jinja2 filters like `default()` and chained `lower | replace` made templates portable and flexible. These patterns — dependency chains, layered variable precedence, and Galaxy-scaffolded structure — are how roles are built and shared in real Ansible teams.
