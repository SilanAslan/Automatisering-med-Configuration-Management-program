# Examination 10 - Templating

With the installation of the web server earlier in Examination 6, we set up
the `nginx` web server with a static configuration file that listened to all
interfaces on the (virtual) machine.

What we really want is for the webserver to _only_ listen to the external
interface, i.e. the interface with the IP address that we connect to the machine to.

Of course, we can statically enter the IP address into the file and upload it,
but if the IP address of the machine changes, we have to do it again, and if the
playbook is meant to be run against many different web servers, we have to be able
to do this manually.

Make a directory called `templates/` and put the `nginx` configuration file from Examination 6
into that directory, and call it `example.internal.conf.j2`.

If you look at the `nginx` documentation, note that you don't have to enable any IPv6 interfaces
on the web server. Stick to IPv4 for now.

# QUESTION A

Copy the finished playbook from Examination 6 (`06-web.yml`) and call it `10-web-template.yml`.

Make the static configuration file we used earlier into a Jinja template file,
and set the values for the `listen` parameters to include the external IP
address of the virtual machine itself.

Use the `ansible.builtin.template` module to accomplish this task.

## Svar

Jag skapade en ny templates-mapp och en example.internal.conf.j2-mallfil. I mallfilen bytte jag ut de statiska listen-direktiven mot den dynamiska variabeln {{ ansible_default_ipv4.address }} för både port 80 och 443.
``` 
shilan@shilan-Precision-Tower-3620:~/ansible$ cat templates/example.internal.conf.j2
server {
    listen {{ ansible_default_ipv4.address }}:80;
    listen {{ ansible_default_ipv4.address }}:443 ssl;
    root /var/www/example.internal/html;
    index index.html;
    server_name example.internal;

    ssl_certificate "/etc/pki/nginx/server.crt";
    ssl_certificate_key "/etc/pki/nginx/private/server.key";
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers PROFILE=SYSTEM;
    ssl_prefer_server_ciphers on;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
Därefter uppdaterade jag min playbook 10-web-template.yml genom att byta ut ansible.builtin.copy-modulen mot ansible.builtin.template och peka src: till den nya .j2-filen.
 ```
---
- name: Configure webserver for HTTPS
  hosts: web
  become: true
  tasks:
    - name: Ensure HTTPS configuration file is present
      ansible.builtin.copy:
        src: files/https.conf
        dest: /etc/nginx/conf.d/https.conf
      register: https_conf_result

    - name: Ensure the nginx configuration is updated for example.internal
      ansible.builtin.template:
        src: templates/example.internal.conf.j2
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
        src: files/index.html
        dest: /var/www/example.internal/html/index.html
        owner: nginx
        group: nginx
        mode: 0644

    - name: Ensure nginx is restarted
      ansible.builtin.service:
        name: nginx
        state: restarted
      when: https_conf_result.changed or example_internal_conf_result.changed

```
När jag körde playbooken blev resultatet changed=2, vilket var väntat. Andra uppgiften (task) med modulen ansible.builtin.template upptäckte en ändring (den nya listen-adressen) och when-villkoret triggade en omstart av Nginx.
```
shilan@shilan-Precision-Tower-3620:~/ansible$ ansible-playbook 10-web-template.yml 

PLAY [Configure webserver for HTTPS] *************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [webserver]

TASK [Ensure HTTPS configuration file is present] ************************************************************************************************************************
ok: [webserver]

TASK [Ensure the nginx configuration is updated for example.internal] ****************************************************************************************************
changed: [webserver]

TASK [Ensure virtual host directory is created] **************************************************************************************************************************
ok: [webserver]

TASK [Upload index.html to virtual host directory] ***********************************************************************************************************************
ok: [webserver]

TASK [Ensure nginx is restarted] *****************************************************************************************************************************************
changed: [webserver]

PLAY RECAP ***************************************************************************************************************************************************************
webserver                  : ok=6    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 

```
Slutligen verifierade jag att ändringen fungerade. Mina curl-tester (se nedan) bevisar att webservern nu korrekt svarar på example.internal (både HTTP och HTTPS), men helt ignorerar localhost (ger "Connection failed"), vilket var målet med uppgiften.

```
shilan@shilan-Precision-Tower-3620:~/ansible$ curl http://example.internal
<?xml version="1.0"?>
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>Hello Nackademin!</title>
  </head>
  <body>
    <h1>Hello Nackademin!</h1>
    <p>This is a totally awesome web page</p>
    <p>This page has been uploaded with <a href="https://docs.ansible.com/">Ansible</a>!</p>
  </body>
</html>
``` 
``` 
shilan@shilan-Precision-Tower-3620:~/ansible$ curl http://localhost
curl: (7) Failed to connect to localhost port 80 after 0 ms: Couldn't connect to server
``` 
``` 
shilan@shilan-Precision-Tower-3620:~/ansible$ curl --insecure https://example.internal
<?xml version="1.0"?>
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>Hello Nackademin!</title>
  </head>
  <body>
    <h1>Hello Nackademin!</h1>
    <p>This is a totally awesome web page</p>
    <p>This page has been uploaded with <a href="https://docs.ansible.com/">Ansible</a>!</p>
  </body>
</html>
``` 
``` 
shilan@shilan-Precision-Tower-3620:~/ansible$ curl --insecure  https://localhost
curl: (7) Failed to connect to localhost port 443 after 0 ms: Couldn't connect to server
```

# Resources and Documentation

* https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html
* https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html
* https://nginx.org/en/docs/http/ngx_http_core_module.html#listen
