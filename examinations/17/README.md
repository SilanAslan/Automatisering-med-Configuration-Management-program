# Examination 17 - sudo rules

In real life, passwordless sudo rules is a security concern. Most of the time, we want
to protect the switching of user identity through a password.

# QUESTION A

Create an Ansible role or playbook to remove the passwordless `sudo` rule for the `deploy`
user on your machines, but create a `sudo` rule to still be able to use `sudo` for everything,
but be required to enter a password.

On each virtual machine, the `deploy` user got its passwordless `sudo` rule from the Vagrant
bootstrap script, which placed it in `/etc/sudoers.d/deploy`.

Your solution should be able to have `deploy` connect to the host, make the change, and afterwards
be able to `sudo`, only this time with a password.

To be clear; we want to make sure that at no point should the `deploy` user be completely without
the ability to use `sudo`, since then we're pretty much locked out of our machines (save for using
Vagrant to connect to he machine and fix the problem).

*Tip*: Check out _validate_ in [ansible.builtin.lineinfile](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/lineinfile_module.html) to ensure a file can be parsed correctly (such as running `visudo --check`)
before being changed.

No password is set for the `deploy` user, so begin by setting the password to `hunter12`.

HINT: To construct a password hash (such as one for use in [ansible.builtin.user](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html), you can use the following command:

    $ openssl passwd -6 hunter12

This will give you a SHA512 password hash that you can use in the `password:` field.

You can verify this by trying to login to any of the nodes without the SSH key, but using the password
provided instead.

To be able to use the password when running the playbooks later, you must use the `--ask-become-pass`
option to `ansible` and `ansible-playbook` to provide the password. You can also place the password
in a file, like with `ansible-vault`, or have it encrypted via `ansible-vault`.

## Svar

17-sudo.yml
```
---
- name: Update deploy sudo rules
  hosts: all
  become: true
  tasks:
    - name: Set login password for deploy user
      ansible.builtin.user:
        name: deploy
        password: $6$0omElwfmc1OenL/g$0HacTM0O2QNsuq8.8L3zDL4IwhonY9/LRnslLbvX6jKNRqAAe9yGXqqNlqoijsr.W5x3euyB8fFRImI1i3FWi/
        update_password: always

    - name: Replace NOPASSWD rule for deploy
      ansible.builtin.lineinfile:
        path: /etc/sudoers.d/deploy
        regexp: '^deploy ALL=(ALL) NOPASSWD: ALL'
        line: 'deploy ALL=(ALL) ALL'
        validate: 'visudo -cf %s'
```

Playbooken kördes framgångsrikt (changed=2), vilket bevisar att båda ändringarna genomfördes:

```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible-playbook 17-sudo.yml --ask-become-pass
BECOME password: 
```
Output:
```
PLAY [Update deploy sudo rules] ***************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************
ok: [dbserver]
ok: [webserver]

TASK [Set login password for deploy user] *****************************************************************************************
changed: [webserver]
changed: [dbserver]

TASK [Replace NOPASSWD rule for deploy] *******************************************************************************************
changed: [webserver]
changed: [dbserver]

PLAY RECAP ************************************************************************************************************************
dbserver                   : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
webserver                  : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```

### Verifering
Sudo-test (Misslyckat): En ny körning med --ask-become-pass men utan att ange lösenord misslyckades korrekt med FAILED! => {"msg": "Missing sudo password"}. Detta bevisar att NOPASSWD-regeln är borta:
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible-playbook 17-sudo.yml --ask-become-pass
BECOME password: 

PLAY [Update deploy sudo rules] ***************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************
fatal: [webserver]: FAILED! => {"msg": "Missing sudo password"}
fatal: [dbserver]: FAILED! => {"msg": "Missing sudo password"}

PLAY RECAP ************************************************************************************************************************
dbserver                   : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
webserver                  : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   

shilan@shilan-Precision-Tower-3620:~/ansible$ ansible-playbook 17-sudo.yml --ask-become-pass
```

SSH-test (Lyckat): Manuell SSH-inloggning med -o PubkeyAuthentication=no lyckades med lösenordet hunter12, vilket bevisar att ansible.builtin.user-modulen fungerade korrekt:
```
shilan@shilan-Precision-Tower-3620:~/Desktop/vagrant$ ssh -o PubkeyAuthentication=no deploy@192.168.121.158
deploy@192.168.121.158's password: 
Last login: Sat Nov  8 19:11:24 2025 from 192.168.121.1
[deploy@dbserver ~]$ 

```
```
shilan@shilan-Precision-Tower-3620:~/Desktop/vagrant$ ssh -o PubkeyAuthentication=no deploy@192.168.121.78
deploy@192.168.121.78's password: 
Last login: Sat Nov  8 19:15:37 2025 from 192.168.121.1
[deploy@webserver ~]$ 

```
