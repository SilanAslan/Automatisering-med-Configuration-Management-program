# Examination 14 - Firewalls (VG)

The IT security team has noticed that we do not have any firewalls enabled on the servers,
and thus ITSEC surmises that the servers are vulnerable to intruders and malware.

As a first step to appeasing them, we will install and enable `firewalld` and
enable the services needed for connecting to the web server(s) and the database server(s).

# QUESTION A

Create a playbook `14-firewall.yml` that utilizes the [ansible.posix.firewalld](https://docs.ansible.com/ansible/latest/collections/ansible/posix/firewalld_module.html) module to enable the following services in firewalld:

* On the webserver(s), `http` and `https`
* On the database servers(s), the `mysql`

You will need to install `firewalld` and `python3-firewall`, and you will need to enable
the `firewalld` service and have it running on all servers.

When the playbook is run, you should be able to do the following on each of the
servers:

## dbserver

    [deploy@dbserver ~]$ sudo cat /etc/firewalld/zones/public.xml
    <?xml version="1.0" encoding="utf-8"?>
    <zone>
      <short>Public</short>
      <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
      <service name="ssh"/>
      <service name="dhcpv6-client"/>
      <service name="cockpit"/>
      <service name="mysql"/>
    <forward/>
    </zone>

## webserver

    [deploy@webserver ~]$ sudo cat /etc/firewalld/zones/public.xml
    <?xml version="1.0" encoding="utf-8"?>
    <zone>
      <short>Public</short>
      <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
      <service name="ssh"/>
      <service name="dhcpv6-client"/>
      <service name="cockpit"/>
      <service name="https"/>
      <service name="http"/>
      <forward/>
    </zone>

# Resources and Documentation

https://firewalld.org/

### Svar
14-firewall.yml:
```
---
# Play 1: Installation and Start of the Firewall on all hosts
- name: Ensure firewalld is installed and running on all hosts
  hosts: all
  become: true
  tasks:
    - name: Install firewalld and dependencies
      ansible.builtin.package:
        name:
          - firewalld
          - python3-firewall
        state: present

    - name: Enable and start firewalld
      ansible.builtin.service:
        name: firewalld
        enabled: true
        state: started

# Play2: Configure HTTP/HTTPS rules for Webservers
- name: Configure HTTP/HTTPS on Webservers
  hosts: web
  become: true
  tasks:
    - name: Open HTTP and HTTPS permanently in the Public Zone
      ansible.posix.firewalld:
        service: "{{ item }}" # Add both services in one loop
        permanent: true
        state: enabled
        immediate: true
      loop: 
        - http
        - https

# Play 3: Configure MySQL Rule for Database Servers
- name: Configure MySQL on Database Servers
  hosts: db
  become: true
  tasks:
    - name: Open MySQL permanently in the Public Zone
      ansible.posix.firewalld:
        service: mysql
        permanent: true
        state: enabled
        zone: public
        immediate: true
```

Output:
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible-playbook 14-firewall.yml
```
```
PLAY [Ensure firewalld is installed and running on all hosts] **************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************
ok: [dbserver]
ok: [webserver]

TASK [Install firewalld and dependencies] **********************************************************************************************
changed: [webserver]
changed: [dbserver]

TASK [Enable and start firewalld] ******************************************************************************************************
changed: [webserver]
changed: [dbserver]

PLAY [Configure HTTP/HTTPS on Webservers] **********************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************
ok: [webserver]

TASK [Open HTTP and HTTPS permanently in the Public Zone] ******************************************************************************
changed: [webserver] => (item=http)
changed: [webserver] => (item=https)

PLAY [Configure MySQL on Database Servers] *********************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************
ok: [dbserver]

TASK [Open MySQL permanently in the Public Zone] ***************************************************************************************
changed: [dbserver]

PLAY RECAP *****************************************************************************************************************************
dbserver                   : ok=5    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
webserver                  : ok=5    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
Kontroller:
- dbserver -> /etc/firewalld/zones/public.xml
```
shilan@shilan-Precision-Tower-3620:~/Desktop/vagrant$ ssh -i deploy_key -l deploy 192.168.121.158
Last login: Tue Nov  4 17:35:30 2025 from 192.168.121.1
[deploy@dbserver ~]$ sudo cat /etc/firewalld/zones/public.xml
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Public</short>
  <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
  <service name="ssh"/>
  <service name="dhcpv6-client"/>
  <service name="cockpit"/>
  <service name="mysql"/>
  <forward/>
</zone>
[deploy@dbserver ~]$ 
```
- webserver -> /etc/firewalld/zones/public.xml

```
shilan@shilan-Precision-Tower-3620:~/Desktop/vagrant$ ssh -i deploy_key -l deploy 192.168.121.78
Last login: Tue Nov  4 17:00:31 2025 from 192.168.121.1
[deploy@webserver ~]$ sudo cat /etc/firewalld/zones/public.xml
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Public</short>
  <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
  <service name="ssh"/>
  <service name="dhcpv6-client"/>
  <service name="cockpit"/>
  <service name="http"/>
  <service name="https"/>
  <forward/>
</zone>
[deploy@webserver ~]$ 
```