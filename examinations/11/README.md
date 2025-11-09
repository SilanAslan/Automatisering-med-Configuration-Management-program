# Examination 11 - Loops

Imagine that on the web server(s), the IT department wants a number of users accounts set up:

    alovelace
    aturing
    edijkstra
    ghopper

These requirements are also requests:

* `alovelace` and `ghopper` should be added to the `wheel` group.
* `aturing` should be added to the `tape` group
* `edijkstra` should be added to the `tcpdump` group.
* `alovelace` should be added to the `audio` and `video` groups.
* `ghopper` should be in the `audio` group, but not in the `video` group.

Also, the IT department, for some unknown reason, wants to copy a number of '\*.md' files
to the 'deploy' user's home directory on the `db` machine(s).

I recommend you use two different playbooks for these two tasks. Prefix them both with `11-` to
make it easy to see which examination it belongs to.

# QUESTION A

Write a playbook that uses loops to add these users, and adds them to their respective groups.

When your playbook is run, one should be able to do this on the webserver:

    [deploy@webserver ~]$ groups alovelace
    alovelace : alovelace wheel video audio
    [deploy@webserver ~]$ groups aturing
    aturing : aturing tape
    [deploy@webserver ~]$ groups edijkstra
    edijkstra : edijkstra tcpdump
    [deploy@webserver ~]$ groups ghopper
    ghopper : ghopper wheel audio

There are multiple ways to accomplish this, but keep in mind _idempotency_ and _maintainability_.

## Svar
Här är min playbook som skapar användare och lägger de till de grupper som önskas:
```
---
- name: Create IT department user accounts
  hosts: web
  become: true

  tasks:
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
Output:
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible-playbook 11-users.yml 

PLAY [Create IT department user accounts] ************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************
ok: [webserver]

TASK [Create IT department users and add to groups] **************************************************************************************
changed: [webserver] => (item={'name': 'alovelace', 'groups': 'wheel,video,audio'})
changed: [webserver] => (item={'name': 'aturing', 'groups': 'tape'})
changed: [webserver] => (item={'name': 'edijkstra', 'groups': 'tcpdump'})
changed: [webserver] => (item={'name': 'ghopper', 'groups': 'wheel,audio'})

PLAY RECAP *******************************************************************************************************************************
webserver                  : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```
Kontroller:
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible web -m command -a "groups alovelace"
webserver | CHANGED | rc=0 >>
alovelace : alovelace wheel video audio 
```
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible web -m command -a "groups aturing"
webserver | CHANGED | rc=0 >>
aturing : aturing tape
```
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible web -m command -a "groups edijkstra"
webserver | CHANGED | rc=0 >>
edijkstra : edijkstra tcpdump
```
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible web -m command -a "groups ghopper"
webserver | CHANGED | rc=0 >>
ghopper : ghopper wheel audio
```
# QUESTION B

Write a playbook that uses

    with_fileglob: 'files/*.md5'

to copy all `\*.md` files in the `files/` directory to the `deploy` user's directory on the `db` server(s).

For now you can create empty files in the `files/` directory called anything as long as the suffix is `.md`:

    $ touch files/foo.md files/bar.md files/baz.md

## Svar
11-copy-md-files.yml
```
---
- name: Copy markdown files to DB server
  hosts: db
  tasks:
    - name: Copy all .md files to the deploy's home directory 
      ansible.builtin.copy:
        src: "{{ item }}"
        dest: "/home/deploy"
        owner: "deploy"
        mode: 0644
      with_fileglob: "files/*.md"
```
Output:
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible-playbook 11-copy-md-files.yml

PLAY [Copy markdown files to DB server] *****************************************************************************

TASK [Gathering Facts] **********************************************************************************************
ok: [dbserver]

TASK [Copy all .md files to the deploy's home directory] ************************************************************
changed: [dbserver] => (item=/home/shilan/ansible/files/foo.md)
changed: [dbserver] => (item=/home/shilan/ansible/files/baz.md)
changed: [dbserver] => (item=/home/shilan/ansible/files/bar.md)

PLAY RECAP **********************************************************************************************************
dbserver                   : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

Kontroller:
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible db -m shell -a "ls -l /home/deploy/*.md"
dbserver | CHANGED | rc=0 >>
-rw-r--r--. 1 deploy deploy 0 Nov  2 23:22 /home/deploy/bar.md
-rw-r--r--. 1 deploy deploy 0 Nov  2 23:22 /home/deploy/baz.md
-rw-r--r--. 1 deploy deploy 0 Nov  2 23:22 /home/deploy/foo.md
```
# BONUS QUESTION

Add a password to each user added to the playbook that creates the users. Do not write passwords in plain
text in the playbook, but use the password hash, or encrypt the passwords using `ansible-vault`.

There are various utilities that can output hashed passwords, check the FAQ for some pointers.

Första:
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible all -i localhost, -m debug -a "msg={{ 'password1' | password_hash('sha512', 'salt1') }}"
localhost | SUCCESS => {
    "msg": "$6$rounds=656000$salt1$/r3SnFPrrLbyfArcde7TDJpr2TZ7mKWgbljHbAhugjKWubuOZVVFZWqnjPBQSa8kKq1swFVuGhSigWocgnK4H1"
}
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible all -i localhost, -m debug -a "msg={{ 'password2' | password_hash('sha512', 'salt2') }}"
localhost | SUCCESS => {
    "msg": "$6$rounds=656000$salt2$XVLIvjDOttePUNkZNoML.XNOagC7NDRBlC90I5RLccPKJa9OIk4vuelXbjIA.AcZeS8ePeE9mXpbNCfem74VG0"
}
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible all -i localhost, -m debug -a "msg={{ 'password3' | password_hash('sha512', 'salt3') }}"
localhost | SUCCESS => {
    "msg": "$6$rounds=656000$salt3$SGWYfKonq4nhOTrqnZf2pvNQnPHvMwe7/kNtjwk12TBi/8E0yeP9UGsWZ.hwS6ueMlVnTdvDlKpvziDwFRwnU1"
}
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible all -i localhost, -m debug -a "msg={{ 'password4' | password_hash('sha512', 'salt4') }}"
localhost | SUCCESS => {
    "msg": "$6$rounds=656000$salt4$aPkSl8G6Rmn7uciY0pV8NXOod7qmi0VZm9D6vPPX9SnpVdYPyzSYhU0W4N66bQkUTc/Xcul920GBP5NpFFUDI/"

```
# BONUS BONUS QUESTION

Add the real names of the users we added earlier to the GECOS field of each account. Google is your friend.

# Resources and Documentation

* [loops](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html)
* [ansible.builtin.user](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html)
* [ansible.builtin.fileglob](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/fileglob_lookup.html)
* https://docs.ansible.com/ansible/latest/reference_appendices/faq.html#how-do-i-generate-encrypted-passwords-for-the-user-module

