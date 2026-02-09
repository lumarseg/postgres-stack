# Ansible para PostgreSQL HA Cluster

## Objetivo

Automatizar el despliegue, configuración y mantenimiento del cluster PostgreSQL HA usando Ansible.

## Casos de uso

| Caso | Descripción | Frecuencia |
|------|-------------|------------|
| Despliegue inicial | Instalar Docker, copiar configs, levantar stack | Una vez |
| Configuración | Gestionar /etc/hosts, docker-compose.yml, configs | Cuando hay cambios |
| Rolling updates | Actualizar contenedores uno a uno sin downtime | Mensual/según necesidad |
| Mantenimiento | Backups, limpieza de logs, health checks | Diario/semanal |
| Recuperación | Reconstruir un nodo desde cero | Cuando falla un nodo |

## Estructura de directorios

```
proxmox/
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/
│   │   └── hosts.yml
│   ├── group_vars/
│   │   └── postgres_cluster.yml
│   ├── roles/
│   │   ├── common/
│   │   │   └── tasks/
│   │   │       └── main.yml
│   │   ├── docker/
│   │   │   └── tasks/
│   │   │       └── main.yml
│   │   ├── postgres-ha/
│   │   │   ├── tasks/
│   │   │   │   └── main.yml
│   │   │   ├── templates/
│   │   │   │   ├── docker-compose.yml.j2
│   │   │   │   └── .env.j2
│   │   │   └── handlers/
│   │   │       └── main.yml
│   │   ├── haproxy/
│   │   │   ├── tasks/
│   │   │   │   └── main.yml
│   │   │   └── templates/
│   │   │       └── haproxy.cfg.j2
│   │   └── keepalived/
│   │       ├── tasks/
│   │       │   └── main.yml
│   │       ├── templates/
│   │       │   ├── Dockerfile.j2
│   │       │   └── keepalived.conf.j2
│   │       └── handlers/
│   │           └── main.yml
│   └── playbooks/
│       ├── deploy.yml
│       ├── update.yml
│       ├── backup.yml
│       ├── recover-node.yml
│       └── health-check.yml
```

---

## Archivos de configuración

### ansible.cfg

```ini
[defaults]
inventory = inventory/hosts.yml
remote_user = debian
private_key_file = ~/.ssh/id_rsa
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
become_user = root
```

### inventory/hosts.yml

```yaml
all:
  children:
    postgres_cluster:
      hosts:
        vm06:
          ansible_host: 192.168.37.16
          node_ip: 192.168.37.16
          keepalived_priority: 100
          keepalived_state: MASTER
          etcd_name: etcd01
          patroni_name: vm06
          failover_priority: 3
        vm07:
          ansible_host: 192.168.37.17
          node_ip: 192.168.37.17
          keepalived_priority: 99
          keepalived_state: BACKUP
          etcd_name: etcd02
          patroni_name: vm07
          failover_priority: 2
        vm08:
          ansible_host: 192.168.37.18
          node_ip: 192.168.37.18
          keepalived_priority: 98
          keepalived_state: BACKUP
          etcd_name: etcd03
          patroni_name: vm08
          failover_priority: 1
```

### group_vars/postgres_cluster.yml

```yaml
---
# Cluster settings
cluster_name: postgres-cluster
vip: 192.168.37.15
haproxy_port: 5000
haproxy_stats_port: 7000

# etcd settings
etcd_version: "v3.5.17"
etcd_cluster: "etcd01=http://192.168.37.16:2380,etcd02=http://192.168.37.17:2380,etcd03=http://192.168.37.18:2380"
etcd_hosts: "192.168.37.16:2379,192.168.37.17:2379,192.168.37.18:2379"

# PostgreSQL settings
postgres_version: "16"
spilo_image: "ghcr.io/zalando/spilo-16:3.3-p1"
postgres_superuser: postgres
postgres_password: "cjtz8CHhVxF70hbTjpBu"
postgres_replicator: replicator
postgres_replicator_password: "replpass8CHhVxF70hbT"
postgres_admin: admin
postgres_admin_password: "cjtz8CHhVxF70hbTjpBu"

# HAProxy settings
haproxy_image: "haproxy:2.9-alpine"
haproxy_maxconn: 1000

# Keepalived settings
keepalived_vrrp_id: 51
keepalived_auth_pass: pgcluster
keepalived_interface: eth0

# Paths
stack_path: /opt/postgres-stack

# All nodes (for templates)
cluster_nodes:
  - { name: "db01", ip: "192.168.37.16" }
  - { name: "db02", ip: "192.168.37.17" }
  - { name: "db03", ip: "192.168.37.18" }
```

---

## Roles

### Role: common

**tasks/main.yml**
```yaml
---
- name: Update /etc/hosts with correct IP
  lineinfile:
    path: /etc/hosts
    regexp: '^127\.0\.1\.1.*{{ inventory_hostname }}'
    line: "{{ node_ip }} {{ inventory_hostname }}.home.arpa {{ inventory_hostname }}"
    state: present

- name: Add cluster hosts to /etc/hosts
  lineinfile:
    path: /etc/hosts
    regexp: ".*{{ item.name }}$"
    line: "{{ item.ip }} {{ item.name }}.home.arpa {{ item.name }}"
    state: present
  loop: "{{ cluster_nodes }}"

- name: Install required packages
  apt:
    name:
      - curl
      - gnupg
      - ca-certificates
    state: present
    update_cache: yes
```

### Role: docker

**tasks/main.yml**
```yaml
---
- name: Check if Docker is installed
  command: docker --version
  register: docker_installed
  ignore_errors: yes
  changed_when: false

- name: Install Docker
  when: docker_installed.rc != 0
  block:
    - name: Add Docker GPG key
      apt_key:
        url: https://download.docker.com/linux/debian/gpg
        state: present

    - name: Add Docker repository
      apt_repository:
        repo: "deb https://download.docker.com/linux/debian {{ ansible_distribution_release }} stable"
        state: present

    - name: Install Docker
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-compose-plugin
        state: present
        update_cache: yes

    - name: Add user to docker group
      user:
        name: "{{ ansible_user }}"
        groups: docker
        append: yes

    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: yes
```

### Role: postgres-ha

**tasks/main.yml**
```yaml
---
- name: Create stack directory
  file:
    path: "{{ stack_path }}"
    state: directory
    mode: '0755'

- name: Deploy docker-compose.yml
  template:
    src: docker-compose.yml.j2
    dest: "{{ stack_path }}/docker-compose.yml"
    mode: '0644'
  notify: Restart postgres stack

- name: Deploy .env file
  template:
    src: .env.j2
    dest: "{{ stack_path }}/.env"
    mode: '0600'
  notify: Restart postgres stack

- name: Pull Docker images
  command: docker compose pull
  args:
    chdir: "{{ stack_path }}"
  changed_when: false

- name: Start etcd and spilo
  command: docker compose up -d etcd spilo
  args:
    chdir: "{{ stack_path }}"
  changed_when: false
```

**templates/docker-compose.yml.j2**
```yaml
services:
  etcd:
    image: quay.io/coreos/etcd:{{ etcd_version }}
    container_name: etcd
    restart: unless-stopped
    network_mode: host
    command:
      - etcd
      - --name=${ETCD_NAME}
      - --initial-advertise-peer-urls=http://${PATRONI_NODE_IP}:2380
      - --listen-peer-urls=http://0.0.0.0:2380
      - --listen-client-urls=http://0.0.0.0:2379
      - --advertise-client-urls=http://${PATRONI_NODE_IP}:2379
      - --initial-cluster={{ etcd_cluster }}
      - --initial-cluster-state=existing
      - --initial-cluster-token={{ cluster_name }}
      - --data-dir=/etcd-data
    volumes:
      - etcd_data:/etcd-data

  spilo:
    image: {{ spilo_image }}
    container_name: spilo
    restart: unless-stopped
    network_mode: host
    depends_on:
      - etcd
    environment:
      SCOPE: {{ cluster_name }}
      PGVERSION: "{{ postgres_version }}"
      ETCD3_HOSTS: {{ etcd_hosts }}
      PATRONI_NAME: ${PATRONI_NAME}
      PATRONI_RESTAPI_CONNECT_ADDRESS: ${PATRONI_NODE_IP}:8008
      PATRONI_RESTAPI_LISTEN: 0.0.0.0:8008
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: ${PATRONI_NODE_IP}:5432
      PATRONI_POSTGRESQL_LISTEN: 0.0.0.0:5432
      PATRONI_POSTGRESQL_DATA_DIR: /home/postgres/pgdata/pgroot/data
      PGUSER_SUPERUSER: ${PGUSER_SUPERUSER}
      PGPASSWORD_SUPERUSER: ${PGPASSWORD_SUPERUSER}
      PGUSER_STANDBY: ${PGUSER_STANDBY}
      PGPASSWORD_STANDBY: ${PGPASSWORD_STANDBY}
      PGUSER_ADMIN: ${PGUSER_ADMIN}
      PGPASSWORD_ADMIN: ${PGPASSWORD_ADMIN}
      PATRONI_TAGS: '{"failover_priority": {{ failover_priority }}}'
    volumes:
      - postgres_data:/home/postgres/pgdata

  haproxy:
    image: {{ haproxy_image }}
    container_name: haproxy
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    depends_on:
      - spilo

  keepalived:
    build: ./keepalived
    container_name: keepalived
    restart: unless-stopped
    network_mode: host
    cap_add:
      - NET_ADMIN
      - NET_BROADCAST
      - NET_RAW
    depends_on:
      - haproxy

volumes:
  etcd_data:
  postgres_data:
```

**templates/.env.j2**
```
ETCD_NAME={{ etcd_name }}
PATRONI_NAME={{ patroni_name }}
PATRONI_NODE_IP={{ node_ip }}
PGUSER_SUPERUSER={{ postgres_superuser }}
PGPASSWORD_SUPERUSER={{ postgres_password }}
PGUSER_STANDBY={{ postgres_replicator }}
PGPASSWORD_STANDBY={{ postgres_replicator_password }}
PGUSER_ADMIN={{ postgres_admin }}
PGPASSWORD_ADMIN={{ postgres_admin_password }}
```

**handlers/main.yml**
```yaml
---
- name: Restart postgres stack
  command: docker compose up -d
  args:
    chdir: "{{ stack_path }}"
```

### Role: haproxy

**tasks/main.yml**
```yaml
---
- name: Deploy haproxy.cfg
  template:
    src: haproxy.cfg.j2
    dest: "{{ stack_path }}/haproxy.cfg"
    mode: '0644'
  notify: Restart haproxy

- name: Start HAProxy
  command: docker compose up -d haproxy
  args:
    chdir: "{{ stack_path }}"
  changed_when: false
```

**templates/haproxy.cfg.j2**
```
global
    maxconn {{ haproxy_maxconn }}
    log stdout format raw local0

defaults
    log global
    mode tcp
    retries 3
    timeout connect 10s
    timeout client 30m
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:{{ haproxy_stats_port }}
    stats enable
    stats uri /
    stats refresh 10s

listen postgres
    bind *:{{ haproxy_port }}
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

{% for node in cluster_nodes %}
    server {{ node.name }} {{ node.ip }}:5432 maxconn 100 check port 8008
{% endfor %}
```

**handlers/main.yml**
```yaml
---
- name: Restart haproxy
  command: docker compose restart haproxy
  args:
    chdir: "{{ stack_path }}"
```

### Role: keepalived

**tasks/main.yml**
```yaml
---
- name: Create keepalived directory
  file:
    path: "{{ stack_path }}/keepalived"
    state: directory
    mode: '0755'

- name: Deploy Dockerfile
  template:
    src: Dockerfile.j2
    dest: "{{ stack_path }}/keepalived/Dockerfile"
    mode: '0644'
  notify: Rebuild keepalived

- name: Deploy keepalived.conf
  template:
    src: keepalived.conf.j2
    dest: "{{ stack_path }}/keepalived/keepalived.conf"
    mode: '0644'
  notify: Rebuild keepalived

- name: Build and start Keepalived
  command: docker compose up -d --build keepalived
  args:
    chdir: "{{ stack_path }}"
  changed_when: false
```

**templates/Dockerfile.j2**
```dockerfile
FROM alpine:3.19
RUN apk add --no-cache keepalived curl
COPY keepalived.conf /etc/keepalived/keepalived.conf
CMD ["keepalived", "--dont-fork", "--log-console", "-D"]
```

**templates/keepalived.conf.j2**
```
global_defs {
    router_id PG_HA_{{ inventory_hostname }}
    script_user root
    enable_script_security
}

vrrp_script check_haproxy {
    script "/usr/bin/curl -sf http://127.0.0.1:{{ haproxy_stats_port }}/ > /dev/null"
    interval 2
    weight 2
    fall 3
    rise 2
}

vrrp_instance VI_PG {
    state {{ keepalived_state }}
    interface {{ keepalived_interface }}
    virtual_router_id {{ keepalived_vrrp_id }}
    priority {{ keepalived_priority }}
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass {{ keepalived_auth_pass }}
    }

    virtual_ipaddress {
        {{ vip }}/24
    }

    track_script {
        check_haproxy
    }
}
```

**handlers/main.yml**
```yaml
---
- name: Rebuild keepalived
  command: docker compose up -d --build keepalived
  args:
    chdir: "{{ stack_path }}"
```

---

## Playbooks

### playbooks/deploy.yml

Despliegue completo del cluster.

```yaml
---
- name: Deploy PostgreSQL HA Cluster
  hosts: postgres_cluster
  become: yes

  roles:
    - common
    - docker
    - postgres-ha
    - haproxy
    - keepalived

  post_tasks:
    - name: Wait for cluster to stabilize
      pause:
        seconds: 30

    - name: Check cluster status
      command: docker exec spilo patronictl list
      register: cluster_status
      changed_when: false

    - name: Show cluster status
      debug:
        var: cluster_status.stdout_lines
```

### playbooks/update.yml

Rolling update sin downtime.

```yaml
---
- name: Rolling Update PostgreSQL HA Cluster
  hosts: postgres_cluster
  become: yes
  serial: 1  # Un nodo a la vez

  tasks:
    - name: Get current leader
      command: docker exec spilo patronictl list -f json
      register: cluster_info
      run_once: true
      delegate_to: "{{ groups['postgres_cluster'][0] }}"
      changed_when: false

    - name: Parse leader
      set_fact:
        current_leader: "{{ (cluster_info.stdout | from_json | selectattr('Role', 'equalto', 'Leader') | first).Member }}"
      run_once: true

    - name: Skip if this is the leader (update last)
      when: inventory_hostname == current_leader
      meta: end_host

    - name: Pull new images
      command: docker compose pull
      args:
        chdir: "{{ stack_path }}"

    - name: Restart containers
      command: docker compose up -d
      args:
        chdir: "{{ stack_path }}"

    - name: Wait for node to rejoin as replica
      command: docker exec spilo patronictl list
      register: status
      until: "inventory_hostname in status.stdout and 'streaming' in status.stdout"
      retries: 30
      delay: 10
      changed_when: false

- name: Update leader node
  hosts: postgres_cluster
  become: yes
  serial: 1

  tasks:
    - name: Get current leader
      command: docker exec spilo patronictl list -f json
      register: cluster_info
      run_once: true
      delegate_to: "{{ groups['postgres_cluster'][0] }}"
      changed_when: false

    - name: Parse leader
      set_fact:
        current_leader: "{{ (cluster_info.stdout | from_json | selectattr('Role', 'equalto', 'Leader') | first).Member }}"
      run_once: true

    - name: Only run on leader
      when: inventory_hostname == current_leader
      block:
        - name: Switchover to another node
          command: docker exec spilo patronictl switchover {{ cluster_name }} --force

        - name: Wait for switchover
          pause:
            seconds: 15

        - name: Pull new images
          command: docker compose pull
          args:
            chdir: "{{ stack_path }}"

        - name: Restart containers
          command: docker compose up -d
          args:
            chdir: "{{ stack_path }}"

        - name: Wait for node to rejoin
          command: docker exec spilo patronictl list
          register: status
          until: "'streaming' in status.stdout or 'Leader' in status.stdout"
          retries: 30
          delay: 10
          changed_when: false
```

### playbooks/backup.yml

Backup del cluster.

```yaml
---
- name: Backup PostgreSQL Cluster
  hosts: postgres_cluster
  become: yes

  vars:
    backup_dir: /var/backups/postgres
    backup_retention_days: 7

  tasks:
    - name: Get cluster status
      command: docker exec spilo patronictl list -f json
      register: cluster_info
      changed_when: false

    - name: Find a replica for backup
      set_fact:
        backup_node: "{{ (cluster_info.stdout | from_json | selectattr('Role', 'equalto', 'Replica') | first).Member }}"
      run_once: true

    - name: Run backup only on selected replica
      when: inventory_hostname == backup_node
      block:
        - name: Create backup directory
          file:
            path: "{{ backup_dir }}"
            state: directory
            mode: '0700'

        - name: Run pg_basebackup
          command: >
            docker exec spilo bash -c '
              pg_basebackup -h localhost -U {{ postgres_replicator }}
              -D /tmp/backup_{{ ansible_date_time.date }}
              -Ft -z -P
            '
          environment:
            PGPASSWORD: "{{ postgres_replicator_password }}"

        - name: Copy backup from container
          command: >
            docker cp spilo:/tmp/backup_{{ ansible_date_time.date }}
            {{ backup_dir }}/backup_{{ ansible_date_time.date }}

        - name: Cleanup old backups
          find:
            paths: "{{ backup_dir }}"
            age: "{{ backup_retention_days }}d"
            recurse: no
          register: old_backups

        - name: Remove old backups
          file:
            path: "{{ item.path }}"
            state: absent
          loop: "{{ old_backups.files }}"
```

### playbooks/recover-node.yml

Recuperar un nodo que falló.

```yaml
---
- name: Recover Failed Node
  hosts: "{{ target_node }}"
  become: yes

  vars_prompt:
    - name: target_node
      prompt: "Enter the node to recover (vm06, vm07, vm08)"
      private: no

    - name: confirm
      prompt: "This will DELETE all data on {{ target_node }}. Type 'yes' to confirm"
      private: no

  tasks:
    - name: Validate confirmation
      fail:
        msg: "Recovery cancelled"
      when: confirm != 'yes'

    - name: Stop all containers
      command: docker compose down -v
      args:
        chdir: "{{ stack_path }}"
      ignore_errors: yes

    - name: Remove data volumes
      command: docker volume rm postgres-stack_etcd_data postgres-stack_postgres_data
      ignore_errors: yes

    - name: Redeploy stack
      command: docker compose up -d
      args:
        chdir: "{{ stack_path }}"

    - name: Wait for node to sync
      command: docker exec spilo patronictl list
      register: status
      until: "inventory_hostname in status.stdout and 'streaming' in status.stdout"
      retries: 60
      delay: 10
      changed_when: false

    - name: Show final status
      debug:
        var: status.stdout_lines
```

### playbooks/health-check.yml

Verificar salud del cluster.

```yaml
---
- name: Health Check PostgreSQL HA Cluster
  hosts: postgres_cluster
  become: yes
  gather_facts: no

  tasks:
    - name: Check Docker is running
      command: docker ps
      changed_when: false

    - name: Check all containers are running
      command: docker compose ps --format json
      args:
        chdir: "{{ stack_path }}"
      register: containers
      changed_when: false

    - name: Check etcd health
      command: >
        docker exec etcd etcdctl endpoint health
        --endpoints=http://{{ node_ip }}:2379
      register: etcd_health
      changed_when: false

    - name: Check Patroni API
      uri:
        url: "http://{{ node_ip }}:8008/patroni"
        return_content: yes
      register: patroni_status

    - name: Check HAProxy stats
      uri:
        url: "http://{{ node_ip }}:{{ haproxy_stats_port }}/"
        return_content: yes
      register: haproxy_status

    - name: Get cluster status (run once)
      command: docker exec spilo patronictl list
      register: cluster_status
      run_once: true
      changed_when: false

    - name: Display cluster status
      debug:
        var: cluster_status.stdout_lines
      run_once: true

    - name: Check VIP location
      command: ip a show {{ keepalived_interface }}
      register: vip_check
      changed_when: false

    - name: Report VIP status
      debug:
        msg: "VIP {{ vip }} is {% if vip in vip_check.stdout %}ACTIVE{% else %}not active{% endif %} on {{ inventory_hostname }}"
```

---

## Uso

### Despliegue inicial

```bash
cd proxmox/ansible
ansible-playbook playbooks/deploy.yml
```

### Rolling update

```bash
ansible-playbook playbooks/update.yml
```

### Backup

```bash
ansible-playbook playbooks/backup.yml
```

### Recuperar un nodo

```bash
ansible-playbook playbooks/recover-node.yml -e target_node=vm07
```

### Health check

```bash
ansible-playbook playbooks/health-check.yml
```

---

## Comandos útiles

```bash
# Verificar conectividad
ansible postgres_cluster -m ping

# Ejecutar comando en todos los nodos
ansible postgres_cluster -a "docker exec spilo patronictl list"

# Ver estado de contenedores
ansible postgres_cluster -a "docker compose ps" --become

# Reiniciar HAProxy en todos los nodos
ansible postgres_cluster -a "docker compose restart haproxy" --become
```
