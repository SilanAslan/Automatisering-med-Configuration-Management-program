# Examination 9 - Use Ansible Vault for sensitive information

In the previous examination we set a password for the `webappuser`. To keep this password
in plain text in a playbook, or otherwise, is a huge security hole, especially
if we publish it to a public place like GitHub.

There is a way to keep sensitive information encrypted and unlocked at runtime with the
`ansible-vault` tool that comes with Ansible.

https://docs.ansible.com/ansible/latest/vault_guide/index.html

*IMPORTANT*: Keep a copy of the password for _unlocking_ the vault in plain text, so that
I can run the playbook without having to ask you for the password.

# QUESTION A

Make a copy of the playbook from the previous examination, call it `09-mariadb-password.yml`
and modify it so that the task that sets the password is injected via an Ansible variable,
instead of as a plain text string in the playbook.

## Svar
Här är 09-mariadb-password.yml med password som variabel:
```
---
- name: Install Mariadb
  hosts: db
  become: true
  vars:
    webappuser_password: 'secretpassword'
  tasks:
     - name: Ensure MariaDB-server is installed
       ansible.builtin.package:
         name: mariadb-server
         state: present

     - name: Ensure python3-PYMySQL is present on the db-server
       ansible.builtin.package:
         name: python3-PyMySQL
         state: present

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

     - name: Ensure MariaDB-server is started
       ansible.builtin.service:
         name: mariadb
         enabled: true
         state: started
```
Output:
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible-playbook 09-mariadb-password.yml 

PLAY [Install Mariadb] ***************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [dbserver]

TASK [Ensure MariaDB-server is installed] ********************************************************************************************************************************
ok: [dbserver]

TASK [Ensure python3-PYMySQL is present on the db-server] ****************************************************************************************************************
ok: [dbserver]

TASK [Create a new database with name webappdb] **************************************************************************************************************************
ok: [dbserver]

TASK [Create a new user with name webappuser] ****************************************************************************************************************************
[WARNING]: Option column_case_sensitive is not provided. The default is now false, so the column's name will be uppercased. The default will be changed to true in
community.mysql 4.0.0.
ok: [dbserver]

TASK [Ensure MariaDB-server is started] **********************************************************************************************************************************
ok: [dbserver]

PLAY RECAP ***************************************************************************************************************************************************************
dbserver                   : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

# QUESTION B

When the [QUESTION A](#question-a) is solved, use `ansible-vault` to store the password in encrypted
form, and make it possible to run the playbook as before, but with the password as an
Ansible Vault secret instead.

## Svar

- Jag tog bort hela vars:-blocket från playbooken ```09-mariadb-password.yml```. Playbooken använder nu variabeln 
```{{ webappuser_password }}```, men den är inte längre definierad i samma fil.
- Med vault skapade jag en krypterad fil för att lagra lösenordsvariabeln. Vault password är Linux4Ever:
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible-vault create vault.yml
New Vault password: 
Confirm New Vault password: 
```
Inuti filen lade jag:
```
webappuser_password: 'secretpassword'
```
Jag körde playbooken med flaggorna ```--ask-vault-pass``` för att kunna ange lösenordet och ```-e``` för att ange filen som innehåller variabeln:
```
ansible-playbook 09-mariadb-password.yml --ask-vault-pass -e "@vault.yml"
```
Efter att jag angett lösenordet till valvet kunde Ansible låsa upp filen, läsa variabeln och köra playbooken.
Output:
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible-playbook 09-mariadb-password.yml --ask-vault-pass -e "@vault.yml"
Vault password: 

PLAY [Install Mariadb] ***************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [dbserver]

TASK [Ensure MariaDB-server is installed] ********************************************************************************************************************************
ok: [dbserver]

TASK [Ensure python3-PYMySQL is present on the db-server] ****************************************************************************************************************
ok: [dbserver]

TASK [Create a new database with name webappdb] **************************************************************************************************************************
ok: [dbserver]

TASK [Create a new user with name webappuser] ****************************************************************************************************************************
[WARNING]: Option column_case_sensitive is not provided. The default is now false, so the column's name will be uppercased. The default will be changed to true in
community.mysql 4.0.0.
ok: [dbserver]

TASK [Ensure MariaDB-server is started] **********************************************************************************************************************************
ok: [dbserver]

PLAY RECAP ***************************************************************************************************************************************************************
dbserver                   : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
