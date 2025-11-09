# Examination 12 - Roles

So far we have been using separate playbooks and ran them whenever we wanted to make
a specific change.

With Ansible [roles](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html) we
have the capability to organize tasks into sets, which are called roles.

These roles can then be used in a single playbook to perform the right tasks on each host.

Consider a playbook that looks like this:

    ---
    - name: Configure the web server(s) according to specs
      hosts: web
      roles:
        - webserver

    - name: Configure the database server(s) according to specs
      hosts: db
      roles:
        - dbserver

This playbook has two _plays_, each play utilizing a _role_.

This playbook is also included in this directory as [site.yml](site.yml).

Study the Ansible documentation about roles, and then start work on [QUESTION A](#question-a).

# QUESTION A

Considering the playbook above, create a role structure in your Ansible working directory
that implements the previous examinations as two separate roles; one for `webserver`
and one for `dbserver`.

Copy the `site.yml` playbook to be called `12-roles.yml`.

HINT: You can use

    $ ansible-galaxy role init [name]

to create a skeleton for a role. You won't need ALL the directories created by this,
but it gives you a starting point to fill out in case you don't want to start from scratch.

## Svar:
### webserver
main.yml på ~/ansible/roles/webserver/tasks/
```
---
# tasks file for webserver

- name: Ensure nginx installed
  ansible.builtin.package:
    name: nginx
    state: latest

- name: Ensure the /etc/pki/nginx directory exists
  ansible.builtin.file:
    path: /etc/pki/nginx
    state: directory
    mode: "0755"
    owner: root

- name: Ensure we have a /etc/pki/nginx/private directory
  ansible.builtin.file:
    path: /etc/pki/nginx/private
    state: directory
    mode: "0700"
    owner: root

- name: Ensure we have necessary software installed
  ansible.builtin.package:
    name: python3-cryptography
    state: present

- name: Ensure we have a private key for our certificate
  community.crypto.openssl_privatekey:
    path: /etc/pki/nginx/private/server.key

- name: Create a self-signed certificate
  community.crypto.x509_certificate:
    path: /etc/pki/nginx/server.crt
    privatekey_path: /etc/pki/nginx/private/server.key
    provider: selfsigned

- name: Ensure HTTPS configuration file is present
  ansible.builtin.copy:
    src: https.conf
    dest: /etc/nginx/conf.d/https.conf
  register: https_conf_result

- name: Ensure the nginx configuration is updated for example.internal
  ansible.builtin.template:
    src: example.internal.conf.j2
    dest: /etc/nginx/conf.d/example.internal.conf
  register: example_internal_conf_result

- name: Ensure virtual host directory is created
  ansible.builtin.file:
    path: /var/www/example.internal/html/
    state: directory
    owner: nginx
    group: nginx
    mode: 0755
    setype: httpd_sys_content_t

- name: Upload index.html to virtual host directory
  ansible.builtin.copy:
    src: index.html
    dest: /var/www/example.internal/html/index.html
    owner: nginx
    group: nginx
    mode: 0644

- name: Ensure nginx is restarted
  ansible.builtin.service:
    name: nginx
    state: restarted
  when: https_conf_result.changed or example_internal_conf_result.changed

- name: Create IT department users and add to groups
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    state: present
  loop:
    - { name: 'alovelace', groups: 'wheel,video,audio' }
    - { name: 'aturing', groups: 'tape' }
    - { name: 'edijkstra', groups: 'tcpdump' }
    - { name: 'ghopper', groups: 'wheel,audio' }


```
 ~/ansible/roles/webserver/files innehåller:
 - https.conf  
 - index.html

~/ansible/roles/webserver/templates innehåller:
- example.internal.conf.j2


### dbserver
main.yml på ~/ansible/roles/dbserver/tasks/
```
---
# tasks file for dbserver
- name: Ensure MariaDB-server and python3-PYMySQL installed
  ansible.builtin.package:
    name:
      - mariadb-server
      - python3-PyMySQL
    state: present


- name: Ensure MariaDB-server is started
  ansible.builtin.service:
    name: mariadb
    enabled: true
    state: started

- name: Create a new database with name webappdb
  community.mysql.mysql_db:
    name: webappdb
    state: present
    login_unix_socket: /var/lib/mysql/mysql.sock

- name: Create a new user with name webappuser
  community.mysql.mysql_user:
    name: webappuser
    password: '{{ webappuser_password }}'
    priv: 'webappdb.*:ALL'
    state: present
    login_unix_socket: /var/lib/mysql/mysql.sock

- name: Copy all .md files to the deploy's home directory 
  ansible.builtin.copy:
    src: "{{ item }}"
    dest: "/home/deploy"
    owner: "deploy"
    mode: 0644
  with_fileglob: "*.md"


```
~/ansible/roles/dbserver/files innehåller:
- bar.md
- baz.md
- foo.md

### 12-roles.yml
shilan@shilan-Precision-Tower-3620:~/ansible$ cat 12-roles.yml 
```
---
- name: Configure the web server(s) according to specs
  hosts: web
  become: true
  roles:
    - webserver

- name: Configure the database server(s) according to specs
  hosts: db
  become: true
  vars_files:
    - vault.yml
  roles:
    - dbserver
```
Kontroll:
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible-playbook 12-roles.yml --ask-vault-pass
Vault password:
```
Output:
```
PLAY [Configure the web server(s) according to specs] **********************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************
ok: [webserver]

TASK [webserver : Ensure nginx installed] **********************************************************************************************
ok: [webserver]

TASK [webserver : Ensure the /etc/pki/nginx directory exists] **************************************************************************
ok: [webserver]

TASK [webserver : Ensure we have a /etc/pki/nginx/private directory] *******************************************************************
ok: [webserver]

TASK [webserver : Ensure we have necessary software installed] *************************************************************************
ok: [webserver]

TASK [webserver : Ensure we have a private key for our certificate] ********************************************************************
ok: [webserver]

TASK [webserver : Create a self-signed certificate] ************************************************************************************
ok: [webserver]

TASK [webserver : Ensure HTTPS configuration file is present] **************************************************************************
ok: [webserver]

TASK [webserver : Ensure the nginx configuration is updated for example.internal] ******************************************************
ok: [webserver]

TASK [webserver : Ensure virtual host directory is created] ****************************************************************************
ok: [webserver]

TASK [webserver : Upload index.html to virtual host directory] *************************************************************************
ok: [webserver]

TASK [webserver : Ensure nginx is restarted] *******************************************************************************************
skipping: [webserver]

TASK [webserver : Create IT department users and add to groups] ************************************************************************
ok: [webserver] => (item={'name': 'alovelace', 'groups': 'wheel,video,audio'})
ok: [webserver] => (item={'name': 'aturing', 'groups': 'tape'})
ok: [webserver] => (item={'name': 'edijkstra', 'groups': 'tcpdump'})
ok: [webserver] => (item={'name': 'ghopper', 'groups': 'wheel,audio'})

PLAY [Configure the database server(s) according to specs] *****************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************
ok: [dbserver]

TASK [dbserver : Ensure MariaDB-server and python3-PYMySQL installed] ******************************************************************
ok: [dbserver]

TASK [dbserver : Ensure MariaDB-server is started] *************************************************************************************
ok: [dbserver]

TASK [dbserver : Create a new database with name webappdb] *****************************************************************************
ok: [dbserver]

TASK [dbserver : Create a new user with name webappuser] *******************************************************************************
[WARNING]: Option column_case_sensitive is not provided. The default is now false, so the column's name will be uppercased. The default
will be changed to true in community.mysql 4.0.0.
ok: [dbserver]

TASK [dbserver : Copy all .md files to the deploy's home directory] ********************************************************************
ok: [dbserver] => (item=/home/shilan/ansible/roles/dbserver/files/foo.md)
ok: [dbserver] => (item=/home/shilan/ansible/roles/dbserver/files/baz.md)
ok: [dbserver] => (item=/home/shilan/ansible/roles/dbserver/files/bar.md)

PLAY RECAP *****************************************************************************************************************************
dbserver                   : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
webserver                  : ok=12   changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   

shilan@shilan-Precision-Tower-3620:~/ansible$ 
```
