# Examination 16 - Security Compliance Check

The ever-present IT security team were not content with just having us put firewall rules
on our servers. They also want our servers to pass CIS certifications.

# QUESTION A

Implement _at least_ 10 of the checks in the [CIS Security Benchmark](https://www.cisecurity.org/benchmark/almalinuxos_linux) for AlmaLinux 10 and run them on the virtual machines.

These checks should be run by a playbook called `16-compliance-check.yml`.

*Important*: The playbook should only _check_ or _assert_ the compliance status, not perform any changes.

Use Ansible facts, modules, and "safe" commands. Here is an example:

    ---
    - name: Security Compliance Checks
      hosts: all
      tasks:
        - name: check for telnet-server
          ansible.builtin.command:
            cmd: rpm -q telnet-server
            warn: false
          register: result
          changed_when: result.stdout != "package telnet-server is not installed"
          failed_when: result.changed

Again, the playbook should make *no changes* to the servers, only report.

Often, there are more elegant and practical ways to assert compliance. The example above is
taken more or less verbatim from the CIS Security Benchmark suite, but it is often considered
bad practice to run arbitrary commands through [ansible.builtin.command] or [ansible.builtin.shell]
if you can avoid it.

In this case, you _can_ avoid it, by using the [ansible.builtin.package_facts](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_facts_module.html).

In conjunction with the [ansible.builtin.assert](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html) module you have a toolset to accomplish the same thing, only more efficiently and in an Ansible-best-practice way.

For instance:

    ---
    - name: Security Compliance Checks
      hosts: all
      tasks:
        - name: Gather the package facts
          ansible.builtin.package_facts:

        - name: check for telnet-server
          ansible.builtin.assert:
            fail_msg: telnet-server package is installed
            success_msg: telnet-server package is not installed
            that: "'telnet-server' not in ansible_facts.packages"

It is up to you to implement the solution you feel works best.

## Svar
Jag skapade playbooken 16-compliance-check.yml som börjar med att samla in all nödvändig fakta om både installerade paket (package_facts) och systemtjänster (service_facts). Därefter körs 10 assert-kontroller för att granska serverns säkerhetsstatus.

Kontrollerna är uppdelade i två typer:

- Paketkontroller (8 st): Sju av dessa kontroller verifierar att osäkra eller onödiga paket (som vsftpd, tftp-server, cups, kea, autofs, samba och avahi) inte är installerade. En åttonde kontroll verifierar att det nödvändiga paketet firewalld är installerat.

- Tjänstekontroller (2 st): Den första säkerställer att den nödvändiga tjänsten firewalld.service är igång (running). Den andra säkerställer att den onödiga tjänsten rsyncd.service inte är igång.

Resultat av granskning:

Körningen av 16-compliance-check.yml (se output-logg) visar att servrarna klarade alla 10 säkerhetskontroller:

Godkänd (OK): De osäkra paketen vsftpd, tftp-server, cups, kea, autofs, samba och avahi är inte installerade.

Godkänd (OK): Brandväggspaketet firewalld är installerat och firewalld.service är igång (running).

Godkänd (SKIPPED): Kontrollen för rsyncd.service hoppades över, eftersom when-villkoret identifierade att tjänsten inte ens existerar på systemet, vilket uppfyller säkerhetskravet.

16-compliance-check.yaml:
```
---
- name: Security compliance check
  hosts: all
  become: true
  tasks:
    - name: Gather facts about installed packages
      ansible.builtin.package_facts:

    - name: Gather facts about system services
      ansible.builtin.service_facts:

    - name: Ensure FTP server (vsftpd) is not installed
      ansible.builtin.assert:
        that: "'vsftpd' not in ansible_facts.packages"
        fail_msg: "FAILED: vsftpd is installed"
        success_msg: "OK: vsftpd is not installed"

    - name: Ensure TFTP server is not installed
      ansible.builtin.assert:
        that: "'tftp-server' not in ansible_facts.packages"
        fail_msg: "FAILED: TFTP server is installed"
        success_msg: "OK: TFTP server is not installed"

    - name: Ensure CUPS print server is not installed
      ansible.builtin.assert:
        that: "'cups' not in ansible_facts.packages"
        fail_msg: "FAILED: CUPS print server is installed"
        success_msg: "OK: CUPS print server is not installed"

    - name: Ensure DHCP server (kea) is not in use  # kea, den moderna och numera rekommenderade DHCP-serverprogramvaran från ISC
      ansible.builtin.assert:
        that: "'kea' not in ansible_facts.packages"
        fail_msg: "FAILED: DHCP server (kea) is installed"
        success_msg: "OK: DHCP server (kea) is not installed"

    - name: Ensure autofs (automatic mounting) is not in use
      ansible.builtin.assert:
        that: "'autofs' not in ansible_facts.packages"
        fail_msg: "FAILED: autofs is installed"
        success_msg: "OK: autofs is not installed"

    - name: Ensure Samba is not in use
      ansible.builtin.assert:
        that: "'samba' not in ansible_facts.packages"
        fail_msg: "FAILED: samba is installed"
        success_msg: "OK: samba is not installed"

    - name: Ensure avahi deamon is not in use
      ansible.builtin.assert:
        that: "'avahi' not in ansible_facts.packages"
        fail_msg: "FAILED: avahi is installed"
        success_msg: "OK: avahi is not installed"

    - name: Ensure firewalld is installed
      ansible.builtin.assert:
        that: "'firewalld' in ansible_facts.packages"
        fail_msg: "FAILED: firewalld is not installed"
        success_msg: "OK: firewalld is installed"

    - name: Ensure firewalld is running
      ansible.builtin.assert:
        that: "ansible_facts.services['firewalld.service'].state == 'running'"
        fail_msg: "FAILED: firewalld.service is not 'running'."
        success_msg: "OK: firewalld.service is 'running'."
      # Undviker att Ansible kraschar om tjänsten inte existerar, eftersom that försöker läsa .state på en icke-existerande tjänst annars
      when: "'firewalld.service' in ansible_facts.services"

    - name: Ensure rsync service is not in use
      ansible.builtin.assert:
        that: "ansible_facts.services['rsyncd.service'].state != 'running'"
        fail_msg: "FAILED: rsyncd.service is running"
        success_msg: "OK: rsyncd.service is not running"
      when: "'rsyncd.service' in ansible_facts.services"
```
Output:
```

PLAY [Security compliance check] *******************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************
ok: [webserver]
ok: [dbserver]

TASK [Gather facts about installed packages] *******************************************************************************************
ok: [webserver]
ok: [dbserver]

TASK [Gather facts about system services] **********************************************************************************************
ok: [webserver]
ok: [dbserver]

TASK [Ensure FTP server (vsftpd) is not installed] *************************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: vsftpd is not installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: vsftpd is not installed"
}

TASK [Ensure TFTP server is not installed] *********************************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: TFTP server is not installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: TFTP server is not installed"
}

TASK [Ensure CUPS print server is not installed] ***************************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: CUPS print server is not installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: CUPS print server is not installed"
}

TASK [Ensure DHCP server (kea) is not in use] ******************************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: DHCP server (kea) is not installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: DHCP server (kea) is not installed"
}

TASK [Ensure autofs (automatic mounting) is not in use] ********************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: autofs is not installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: autofs is not installed"
}

TASK [Ensure Samba is not in use] ******************************************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: samba is not installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: samba is not installed"
}

TASK [Ensure avahi deamon is not in use] ***********************************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: avahi is not installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: avahi is not installed"
}

TASK [Ensure firewalld is installed] ***************************************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: firewalld is installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: firewalld is installed"
}

TASK [Ensure firewalld is running] *****************************************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: firewalld.service is 'running'."
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: firewalld.service is 'running'."
}

TASK [Ensure rsync service is not in use] **********************************************************************************************
skipping: [dbserver]
skipping: [webserver]

PLAY RECAP *****************************************************************************************************************************
dbserver                   : ok=12   changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
webserver                  : ok=12   changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0  
```

# BONUS QUESTION

If you implement these tasks within one or more roles, you will gain enlightenment and additional karma.

## Svar
Eftersom jag behövde endast en map under roles så skapade jag rollen med:
```
mkdir -p roles/cis/tasks
```
Jag kopierade 16-compliance-check.yaml filen till main.yml och redigerade den:
```
cp 16-compliance-check.yaml roles/cis/tasks/main.yml
```

shilan@shilan-Precision-Tower-3620:~/ansible/roles/cis/tasks$ cat main.yml:
```
---
# Security Compliance Check

- name: Gather facts about installed packages
  ansible.builtin.package_facts:

- name: Gather facts about system services
  ansible.builtin.service_facts:

- name: Ensure FTP server (vsftpd) is not installed
  ansible.builtin.assert:
    that: "'vsftpd' not in ansible_facts.packages"
    fail_msg: "FAILED: vsftpd is installed"
    success_msg: "OK: vsftpd is not installed"

- name: Ensure TFTP server is not installed
  ansible.builtin.assert:
    that: "'tftp-server' not in ansible_facts.packages"
    fail_msg: "FAILED: TFTP server is installed"
    success_msg: "OK: TFTP server is not installed"

- name: Ensure CUPS print server is not installed
  ansible.builtin.assert:
    that: "'cups' not in ansible_facts.packages"
    fail_msg: "FAILED: CUPS print server is installed"
    success_msg: "OK: CUPS print server is not installed"

- name: Ensure DHCP server (kea) is not in use  # kea, den moderna och numera rekommenderade DHCP-serverprogramvaran från ISC
  ansible.builtin.assert:
    that: "'kea' not in ansible_facts.packages"
    fail_msg: "FAILED: DHCP server (kea) is installed"
    success_msg: "OK: DHCP server (kea) is not installed"

- name: Ensure autofs (automatic mounting) is not in use
  ansible.builtin.assert:
    that: "'autofs' not in ansible_facts.packages"
    fail_msg: "FAILED: autofs is installed"
    success_msg: "OK: autofs is not installed"

- name: Ensure Samba is not in use
  ansible.builtin.assert:
    that: "'samba' not in ansible_facts.packages"
    fail_msg: "FAILED: samba is installed"
    success_msg: "OK: samba is not installed"

- name: Ensure avahi deamon is not in use
  ansible.builtin.assert:
    that: "'avahi' not in ansible_facts.packages"
    fail_msg: "FAILED: avahi is installed"
    success_msg: "OK: avahi is not installed"

- name: Ensure firewalld is installed
  ansible.builtin.assert:
    that: "'firewalld' in ansible_facts.packages"
    fail_msg: "FAILED: firewalld is not installed"
    success_msg: "OK: firewalld is installed"

- name: Ensure firewalld is running
  ansible.builtin.assert:
    that: "ansible_facts.services['firewalld.service'].state == 'running'"
    fail_msg: "FAILED: firewalld.service is not 'running'."
    success_msg: "OK: firewalld.service is 'running'."
  # Undviker att Ansible kraschar om tjänsten inte existerar, eftersom that försöker läsa .state på en icke-existerande tjänst annars
  when: "'firewalld.service' in ansible_facts.services"

- name: Ensure rsync service is not in use
  ansible.builtin.assert:
    that: "ansible_facts.services['rsyncd.service'].state != 'running'"
    fail_msg: "FAILED: rsyncd.service is running"
    success_msg: "OK: rsyncd.service is not running"
  when: "'rsyncd.service' in ansible_facts.services"
```
Jag skapade 16-cis.yml som den anropande playbook i ansible-mappen:
```
shilan@shilan-Precision-Tower-3620:~/ansible$ cat 16-cis.yml 
---
- name: Run CIS Security Compliance Check
  hosts: all
  become: true
  roles:
    - cis

```
Outout:
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible-playbook 16-cis.yml 

PLAY [Run CIS Security Compliance Check] ***********************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************
ok: [webserver]
ok: [dbserver]

TASK [cis : Gather facts about installed packages] *************************************************************************************
ok: [webserver]
ok: [dbserver]

TASK [cis : Gather facts about system services] ****************************************************************************************
ok: [webserver]
ok: [dbserver]

TASK [cis : Ensure FTP server (vsftpd) is not installed] *******************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: vsftpd is not installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: vsftpd is not installed"
}

TASK [cis : Ensure TFTP server is not installed] ***************************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: TFTP server is not installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: TFTP server is not installed"
}

TASK [cis : Ensure CUPS print server is not installed] *********************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: CUPS print server is not installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: CUPS print server is not installed"
}

TASK [cis : Ensure DHCP server (kea) is not in use] ************************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: DHCP server (kea) is not installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: DHCP server (kea) is not installed"
}

TASK [cis : Ensure autofs (automatic mounting) is not in use] **************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: autofs is not installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: autofs is not installed"
}

TASK [cis : Ensure Samba is not in use] ************************************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: samba is not installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: samba is not installed"
}

TASK [cis : Ensure avahi deamon is not in use] *****************************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: avahi is not installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: avahi is not installed"
}

TASK [cis : Ensure firewalld is installed] *********************************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: firewalld is installed"
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: firewalld is installed"
}

TASK [cis : Ensure firewalld is running] ***********************************************************************************************
ok: [dbserver] => {
    "changed": false,
    "msg": "OK: firewalld.service is 'running'."
}
ok: [webserver] => {
    "changed": false,
    "msg": "OK: firewalld.service is 'running'."
}

TASK [cis : Ensure rsync service is not in use] ****************************************************************************************
skipping: [dbserver]
skipping: [webserver]

PLAY RECAP *****************************************************************************************************************************
dbserver                   : ok=12   changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
webserver                  : ok=12   changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
```

# Resources

For inspiration and as an example of an advanced project using Ansible for this, see for instance
https://github.com/ansible-lockdown/RHEL10-CIS. Do *NOT*, however, try to run this compliance check
on your virtual (or physical) machines. It will likely have unintended consequences, and may render
your operating system and/or virtual machine unreachable.
