# Ansible Playbooks - Intermediate Lab

## The Scenario

Our company is deploying a new internal application stack. The web team needs a load-balanced pair of web servers running Apache with a custom status page, and the database team needs their servers prepared with MariaDB installed, configured, and seeded with initial data.

You will write multiple playbooks that handle:
- Multi-group targeting with different tasks per group
- Variables and facts in tasks
- Handlers for efficient service restarts
- Conditional logic
- File management and templating basics
- A verification play to confirm everything works

This lab builds on the basics by introducing playbook patterns you'll use in every real-world deployment.

### Prerequisites

On the Ansible Controller, create a new lab directory:

```bash
mkdir -p ~/lab-playbook-intermediate
cd ~/lab-playbook-intermediate
```

Create an `inventory` file:

```ini
[web]
node1 ansible_host=<IP of TargetNode-1 from /home/ansible/inventory/inventory.yaml>

[db]
node2 ansible_host=<IP of TargetNode-2 from /home/ansible/inventory/inventory.yaml>

[all:vars]
app_env=staging
```

> **Note:** The `[all:vars]` section defines variables that apply to every host in the inventory. We'll use `app_env` inside our playbooks.

---

## Part 1: Deploy the Web Servers

Create a file named `deploy_web.yml`:

```yaml
---
- name: Deploy web servers
  hosts: web
  become: yes
  vars:
    http_port: 80
    doc_root: /var/www/html

  tasks:
    - name: Install required packages
      yum:
        name:
          - httpd
          - mod_ssl
          - unzip
        state: present

    - name: Start and enable httpd
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Deploy custom index page
      copy:
        content: |
          <!DOCTYPE html>
          <html>
          <head><title>{{ inventory_hostname }} - {{ app_env }}</title></head>
          <body>
            <h1>Welcome to {{ inventory_hostname }}</h1>
            <p>Environment: <strong>{{ app_env }}</strong></p>
            <p>Server IP: {{ ansible_default_ipv4.address }}</p>
            <p>OS: {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
            <p>Managed by Ansible</p>
          </body>
          </html>
        dest: "{{ doc_root }}/index.html"
        owner: apache
        group: apache
        mode: '0644'

    - name: Deploy a health check endpoint
      copy:
        content: '{"status": "ok", "host": "{{ inventory_hostname }}", "env": "{{ app_env }}"}'
        dest: "{{ doc_root }}/health.json"
        owner: apache
        group: apache
        mode: '0644'
```

### Run the web playbook

```bash
ansible-playbook -i inventory deploy_web.yml
```

### Verify it works

```bash
curl http://<node1_IP>/index.html
curl http://<node1_IP>/health.json
```

You should see the HTML page with the hostname, IP, and OS information filled in, and the health check endpoint returning JSON.

> **Key Concepts Introduced:**
> - **`vars` section**: Defines playbook-level variables
> - **Inventory variables**: `app_env` comes from `[all:vars]` in the inventory
> - **Ansible facts**: `ansible_default_ipv4.address`, `ansible_distribution`, etc. are automatically gathered
> - **`copy` with `content`**: Creates files with dynamic content using variables

---

## Part 2: Introduce Handlers

The playbook above starts httpd unconditionally every run. Let's improve it. Handlers only run when triggered by a task that reports a change, making playbooks more efficient.

Create a new file named `deploy_web_v2.yml`:

```yaml
---
- name: Deploy web servers (with handlers)
  hosts: web
  become: yes
  vars:
    http_port: 80
    doc_root: /var/www/html

  handlers:
    - name: restart httpd
      service:
        name: httpd
        state: restarted

    - name: reload httpd
      service:
        name: httpd
        state: reloaded

  tasks:
    - name: Install required packages
      yum:
        name:
          - httpd
          - mod_ssl
          - unzip
        state: present

    - name: Ensure httpd is enabled
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Configure httpd to listen on correct port
      lineinfile:
        path: /etc/httpd/conf/httpd.conf
        regexp: '^Listen '
        line: "Listen {{ http_port }}"
      notify: restart httpd

    - name: Set ServerName to avoid warnings
      lineinfile:
        path: /etc/httpd/conf/httpd.conf
        regexp: '^ServerName'
        line: "ServerName {{ inventory_hostname }}"
        insertafter: '^#ServerName'
      notify: reload httpd

    - name: Deploy custom index page
      copy:
        content: |
          <!DOCTYPE html>
          <html>
          <head><title>{{ inventory_hostname }} - {{ app_env }}</title></head>
          <body>
            <h1>Welcome to {{ inventory_hostname }}</h1>
            <p>Environment: <strong>{{ app_env }}</strong></p>
            <p>Server IP: {{ ansible_default_ipv4.address }}</p>
            <p>OS: {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
            <p>Managed by Ansible</p>
          </body>
          </html>
        dest: "{{ doc_root }}/index.html"
        owner: apache
        group: apache
        mode: '0644'

    - name: Deploy health check endpoint
      copy:
        content: '{"status": "ok", "host": "{{ inventory_hostname }}", "env": "{{ app_env }}"}'
        dest: "{{ doc_root }}/health.json"
        owner: apache
        group: apache
        mode: '0644'
```

### Run v2 and observe handler behavior

```bash
ansible-playbook -i inventory deploy_web_v2.yml
```

On the first run, the `lineinfile` tasks will report `changed`, which triggers the handlers. Run it again:

```bash
ansible-playbook -i inventory deploy_web_v2.yml
```

On the second run, nothing changes, so the handlers **don't run**. This is idempotency in action.

> **Key Concept: Handlers** are tasks that only execute when notified by another task. They run once at the end of the play, even if notified multiple times. Use them for service restarts to avoid unnecessary disruption.

---

## Part 3: Deploy the Database Servers

Create a file named `deploy_db.yml`:

```yaml
---
- name: Deploy database servers
  hosts: db
  become: yes
  vars:
    db_name: app_inventory
    db_user: app_user
    db_password: SecurePass123

  tasks:
    - name: Install MariaDB packages
      yum:
        name:
          - mariadb-server
          - mariadb
          - python3-PyMySQL
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
        login_unix_socket: /var/lib/mysql/mysql.sock

    - name: Create application user
      mysql_user:
        name: "{{ db_user }}"
        password: "{{ db_password }}"
        priv: "{{ db_name }}.*:ALL"
        host: localhost
        state: present
        login_unix_socket: /var/lib/mysql/mysql.sock

    - name: Create the servers table
      community.mysql.mysql_query:
        db: "{{ db_name }}"
        query: >
          CREATE TABLE IF NOT EXISTS servers (
            id INT AUTO_INCREMENT PRIMARY KEY,
            hostname VARCHAR(255) NOT NULL,
            role VARCHAR(50),
            environment VARCHAR(50),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
          )
        login_unix_socket: /var/lib/mysql/mysql.sock

    - name: Seed the database with initial data
      community.mysql.mysql_query:
        db: "{{ db_name }}"
        query: >
          INSERT INTO servers (hostname, role, environment)
          SELECT '{{ inventory_hostname }}', 'database', '{{ app_env }}'
          WHERE NOT EXISTS (SELECT 1 FROM servers WHERE hostname = '{{ inventory_hostname }}')
        login_unix_socket: /var/lib/mysql/mysql.sock

    - name: Verify database setup
      community.mysql.mysql_query:
        db: "{{ db_name }}"
        query: "SELECT * FROM servers"
        login_unix_socket: /var/lib/mysql/mysql.sock
      register: db_output

    - name: Show database contents
      debug:
        msg: "{{ db_output.query_result }}"
```

### Run the database playbook

```bash
ansible-playbook -i inventory deploy_db.yml
```

You should see the database created, user granted privileges, and the verification showing the seeded data.

> **Note:** The `community.mysql` collection must be installed on the controller. If it's not already available, run:
> ```bash
> ansible-galaxy collection install community.mysql
> ```
> We also install `python3-PyMySQL` on the target hosts — the MySQL modules require a Python MySQL library to connect to the database.

> **Key Concepts Introduced:**
> - **`mysql_db`**: Creates and manages MySQL/MariaDB databases — idempotent and no raw SQL needed
> - **`mysql_user`**: Manages database users and privileges without exposing passwords in process lists
> - **`community.mysql.mysql_query`**: Runs SQL queries and returns structured results
> - **`register`**: Captures the output of a task into a variable
> - **`debug`**: Displays variable contents — invaluable for troubleshooting

---

## Part 4: Multi-Play Playbook with Conditionals

Now let's combine everything into a single playbook with multiple plays and conditional logic.

Create a file named `site.yml`:

```yaml
---
# Play 1: Common tasks for all servers
- name: Common configuration for all servers
  hosts: all
  become: yes
  tasks:
    - name: Ensure chrony (NTP) is installed
      yum:
        name: chrony
        state: present

    - name: Ensure chrony is running
      service:
        name: chronyd
        state: started
        enabled: yes

    - name: Set timezone
      timezone:
        name: UTC

    - name: Create application directories
      file:
        path: "/opt/{{ app_env }}"
        state: directory
        mode: '0755'

    - name: Deploy environment marker file
      copy:
        content: |
          ENVIRONMENT={{ app_env }}
          HOSTNAME={{ inventory_hostname }}
          ROLE={{ group_names | first }}
          DEPLOYED={{ ansible_date_time.iso8601 }}
        dest: "/opt/{{ app_env }}/env.conf"
        mode: '0644'

# Play 2: Web-specific tasks
- name: Configure web servers
  hosts: web
  become: yes
  tasks:
    - name: Verify httpd is installed
      yum:
        name: httpd
        state: present

    - name: Verify httpd is running
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Ensure SSL module is enabled
      apache2_module:
        name: ssl
        state: present
      register: ssl_check
      ignore_errors: yes

    - name: Report SSL status
      debug:
        msg: >
          SSL module is {{ 'enabled' if ssl_check is succeeded else 'NOT available' }}
          on {{ inventory_hostname }}

# Play 3: Database-specific tasks
- name: Configure database servers
  hosts: db
  become: yes
  tasks:
    - name: Verify MariaDB is running
      service:
        name: mariadb
        state: started
        enabled: yes

    - name: Check database status
      community.mysql.mysql_info:
        filter:
          - version
          - databases
        login_unix_socket: /var/lib/mysql/mysql.sock
      register: db_status

    - name: Report database status
      debug:
        msg: "Database version on {{ inventory_hostname }}: {{ db_status.version.full }}"

# Play 4: Verification across all hosts
- name: Final verification
  hosts: all
  become: yes
  tasks:
    - name: Gather service facts
      service_facts:

    - name: Verify chronyd is running on all hosts
      debug:
        msg: "chronyd is {{ ansible_facts.services['chronyd.service'].state }}"
      when: "'chronyd.service' in ansible_facts.services"

    - name: Show environment config
      slurp:
        src: "/opt/{{ app_env }}/env.conf"
      register: env_conf

    - name: Display environment configuration
      debug:
        msg: "{{ env_conf.content | b64decode }}"

    - name: Final summary
      debug:
        msg: >
          Host {{ inventory_hostname }} ({{ ansible_default_ipv4.address }})
          is a member of groups: {{ group_names | join(', ') }}.
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}.
          All services configured successfully.
```

### Run the full site playbook

```bash
ansible-playbook -i inventory site.yml
```

Watch the output carefully. You'll see:
1. Common tasks run on **all** hosts
2. Web tasks run on **only web** hosts
3. Database tasks run on **only db** hosts
4. Verification runs on **all** hosts again

> **Key Concepts Introduced:**
> - **Multiple plays**: A single playbook file can contain multiple plays targeting different host groups
> - **`timezone`**: Sets the system timezone — idempotent and no shell command needed
> - **`apache2_module`**: Manages Apache modules natively instead of parsing shell output
> - **`community.mysql.mysql_info`**: Gathers database information as structured data
> - **`slurp`**: Reads remote file contents — use with the `b64decode` Jinja2 filter to display
> - **`when` conditionals**: Execute tasks only when a condition is true
> - **`ignore_errors`**: Continue playbook execution even if a task fails
> - **`service_facts`**: A module that gathers information about all services on a host
> - **`group_names`**: A magic variable containing the list of groups the current host belongs to
> - **Jinja2 filters**: `| first`, `| join(', ')`, `| b64decode` transform variable values inline

---

## Part 5: Cleanup Playbook

Good automation includes cleanup. Create `cleanup.yml`:

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

    - name: Remove web content
      file:
        path: /var/www/html/index.html
        state: absent

    - name: Remove health check
      file:
        path: /var/www/html/health.json
        state: absent

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

    - name: Remove MariaDB packages
      yum:
        name:
          - mariadb-server
          - mariadb
        state: absent

- name: Cleanup common resources
  hosts: all
  become: yes
  tasks:
    - name: Remove application directory
      file:
        path: "/opt/staging"
        state: absent

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
| `vars` | Define variables in the playbook | Reusable values (ports, paths, names) |
| `handlers` | Tasks that run only when notified | Service restarts after config changes |
| `register` | Capture task output into a variable | When you need to use a result later |
| `debug` | Print messages or variables | Troubleshooting and verification |
| `when` | Conditional task execution | Skip tasks based on facts or results |
| `ignore_errors` | Don't stop on failure | Non-critical checks, cleanup tasks |
| `notify` | Trigger a handler | Efficient service management |
| `mysql_db` / `mysql_user` | Manage databases and users natively | Database setup (instead of raw SQL commands) |
| `community.mysql.mysql_query` | Run SQL queries with structured output | Database seeding and verification |
| `timezone` | Set system timezone | Replaces `timedatectl` command |
| `slurp` | Read remote file contents | Replaces `cat` command in playbooks |
| `apache2_module` | Manage Apache modules | Check/enable modules without shell parsing |
| Multiple plays | Different tasks for different groups | Complex deployments, site-wide playbooks |
| Inventory vars | Variables shared across hosts | Environment names, shared settings |

## Conclusion

In this lab you built a realistic multi-tier deployment from scratch. You started with a simple web deployment, improved it with handlers for efficiency, added a database tier using native Ansible modules (`mysql_db`, `mysql_user`, `community.mysql.mysql_query`) instead of raw shell commands, then combined everything into a site-wide playbook with conditionals and verification. You also learned to use purpose-built modules like `timezone`, `apache2_module`, and `slurp` instead of shelling out to system commands. These patterns — handlers, registered variables, native modules, multi-play playbooks, and conditional logic — are the building blocks of production Ansible automation. The cleanup playbook demonstrates that good automation accounts for teardown, not just setup.
