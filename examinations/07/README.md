# Examination 7 - MariaDB installation

To make a dynamic web site, many use an SQL server to store the data for the web site.

[MariaDB](https://mariadb.org/) is an open-source relational SQL database that is good
to use for our purposes.

We can use a similar strategy as with the _nginx_ web server to install this
software onto the correct host(s). Create the playbook `07-mariadb.yml` with this content:

    ---
    - hosts: db
      become: true
      tasks:
        - name: Ensure MariaDB-server is installed.
          ansible.builtin.package:
            name: mariadb-server
            state: present

# QUESTION A

Make similar changes to this playbook that we did for the _nginx_ server, so that
the `mariadb` service starts automatically at boot, and is started when the playbook
is run.

## Svar
```
---
- name: Install Mariadb
  hosts: db
  become: true
  tasks:
     - name: Ensure MariaDB-server is installed
       ansible.builtin.package:
         name: mariadb-server
         state: present

     - name: Ensure MariaDB-server is started
       ansible.builtin.service:
         name: mariadb
         enabled: true
         state: started
```

# QUESTION B

When you have run the playbook above successfully, how can you verify that the `mariadb`
service is started and is running?

## Svar
Nedan kan man se att den är enabled och active (running):
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible db -b -m ansible.builtin.command -a 'systemctl status mariadb'
dbserver | CHANGED | rc=0 >>
● mariadb.service - MariaDB 10.11 database server
     Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; preset: disabled)
     Active: active (running) since Sat 2025-11-01 22:09:51 UTC; 7min ago
 Invocation: 656fd2ec654b4fae9caaa4c58116e2b9
       Docs: man:mariadbd(8)
             https://mariadb.com/kb/en/library/systemd/
    Process: 21604 ExecStartPre=/usr/libexec/mariadb-check-socket (code=exited, status=0/SUCCESS)
    Process: 21627 ExecStartPre=/usr/libexec/mariadb-prepare-db-dir mariadb.service (code=exited, status=0/SUCCESS)
    Process: 21745 ExecStartPost=/usr/libexec/mariadb-check-upgrade (code=exited, status=0/SUCCESS)
   Main PID: 21731 (mariadbd)
     Status: "Taking your SQL requests now..."
      Tasks: 8 (limit: 2543)
     Memory: 89.3M (peak: 110.3M)
        CPU: 1.052s
     CGroup: /system.slice/mariadb.service
             └─21731 /usr/libexec/mariadbd --basedir=/usr

Nov 01 22:09:51 dbserver mariadb-prepare-db-dir[21667]: you need to be the system 'mysql' user to connect.
Nov 01 22:09:51 dbserver mariadb-prepare-db-dir[21667]: After connecting you can set the password, if you would need to be
Nov 01 22:09:51 dbserver mariadb-prepare-db-dir[21667]: able to connect as any of these users with a password and without sudo
Nov 01 22:09:51 dbserver mariadb-prepare-db-dir[21667]: See the MariaDB Knowledgebase at https://mariadb.com/kb
Nov 01 22:09:51 dbserver mariadb-prepare-db-dir[21667]: Please report any problems at https://mariadb.org/jira
Nov 01 22:09:51 dbserver mariadb-prepare-db-dir[21667]: The latest information about MariaDB is available at https://mariadb.org/.
Nov 01 22:09:51 dbserver mariadb-prepare-db-dir[21667]: Consider joining MariaDB's strong and vibrant community:
Nov 01 22:09:51 dbserver mariadb-prepare-db-dir[21667]: https://mariadb.org/get-involved/
Nov 01 22:09:51 dbserver (mariadbd)[21731]: mariadb.service: Referenced but unset environment variable evaluates to an empty string: MYSQLD_OPTS, _WSREP_NEW_CLUSTER
Nov 01 22:09:51 dbserver systemd[1]: Started mariadb.service - MariaDB 10.11 database server.
```

# BONUS QUESTION

How many different ways can use come up with to verify that the `mariadb` service is running?

## Svar

Om man vill ha ett kortare svar kan man använda is-active istället:
``` 
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible db -b -m ansible.builtin.command -a "systemctl is-active mariadb"
dbserver | CHANGED | rc=0 >>
active
```
Med ps aux | grep maridb kan man bland alla processer som körs filtrera mariadb med grep:
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible db -b -m ansible.builtin.shell -a 'ps aux | grep mariadb'
dbserver | CHANGED | rc=0 >>
mysql      21731  0.0 20.7 1080660 95820 ?       Ssl  22:09   0:00 /usr/libexec/mariadbd --basedir=/usr
root       22391  0.0  0.7   6956  3284 pts/1    S+   22:27   0:00 /bin/sh -c ps aux | grep mariadb
root       22393  0.0  0.4   6380  2120 pts/1    S+   22:27   0:00 grep mariadb
```