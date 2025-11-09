# Examination 13 - Handlers

In [Examination 5](../05/) we asked the question what the disadvantage is of restarting
a service every time a task is run, whether or not it's actually needed.

In order to minimize the amount of restarts and to enable a complex configuration to run
through all its steps before reloading or restarting anything, we can trigger a _handler_
to be run once when there is a notification of change.

Read up on [Ansible handlers](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html)

In the previous examination ([Examination 12](../12/)), we changed the structure of the project to two separate
roles, `webserver` and `dbserver`.

# QUESTION A

Make the necessary changes to the `webserver` role, so that `nginx` only reloads when it's configuration
has changed in a task, such as when we have changed a `server` stanza.

Also note the difference between `restarted` and `reloaded` in the [ansible.builtin.service](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html) module.

In order for `nginx` to pick up any configuration changes, it's enough to do a `reload` instead of
a full `restart`.

### Svar
För att säkerställa att Nginx endast läser in ny konfiguration när en fil faktiskt har ändrats, och undvika onödiga omstarter, skapades en handler som heter reload nginx i handlers/main.yml. Den använder kommandot reload istället för restart, vilket är enklare och mer resurssnålt.

- ~/ansible/roles/webserver/handlers main.yml
```
---
# handlers file for webserver
- name: reload nginx
  ansible.builtin.service:
    name: nginx
    state: reload
```

Tasks som hanterar nginx-konfigurationsfiler (https.conf och example.internal.conf.j2) fick raden notify: reload nginx för att säga till handlern vid ändring.

Den gamla lösningen med register och when: för att bestämma när nginx ska starta om togs bort, eftersom den inte behövs längre.


- ~/ansible/roles/webserver/tasks main.yml

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
  notify:
    - reload nginx

- name: Ensure the nginx configuration is updated for example.internal
  ansible.builtin.template:
    src: example.internal.conf.j2
    dest: /etc/nginx/conf.d/example.internal.conf
  notify:
    - reload nginx

- name: Ensure virtual host directory is created
  ansible.builtin.file:
    path: /var/www/example.internal/html/
    state: directory
    owner: nginx
    group: nginx
    mode: 0644

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
