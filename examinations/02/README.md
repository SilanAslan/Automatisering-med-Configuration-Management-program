# Examination 2 - Configure Ansible

To make things as unsurprising and easy as possible, we will all set up
the exact same Ansible configuration in a dedicated folder for this purpose.

Do not use the directory with the Vagrantfile (the `lab_environment` directory
if you are using this `git` repository).

Create a separate directory for this and the following examinations, and keep
the configuration there.

## Create ansible.cfg

Create a directory for Ansible where you will do your work. You can call it
anything, as long as you remember what it is. Also, it should be created
somewhere within your home directory.

In our examples below, we will call it `ansible`.

In our dedicated Ansible directory, create a file called `ansible.cfg`
(there is an example `ansible.cfg` in this same directory that you
 may copy).

Put the following content (only two lines) into this file:

    [defaults]
    host_key_checking = False
    inventory = hosts

Each row in this file either contains a section name within brackets.
`[defaults]` contain default values that are used if nothing more specific
is given.

`host_key_checking` is a mechanism by which `ssh(1)` checks that a hostname
has a given set of encryption keys given as an identoty. This is a security
mechanism used to avoid spoofing and man-in-the-middle attacks out in the
real world. Here, we disable it, otherwise we'll go crazy every time we
destroy or create the Vagrant hosts with new keys.

The `inventory` is where Ansible finds its inventory if you don't say
anything else, and `hosts` is its value. In this context, `hosts` is a file
that need to contain something that can be parsed as an Ansible inventory.
The `hosts` file in the same directory should contain:

    [all:vars]
    ansible_user=deploy
    ansible_ssh_private_key_file=<path_to_the_deploy_key_created_previously>

    [db]
    <IP or name from the Vagrant database host>

    [web]
    <IP or name from the Vagrant webserver host>

Anything within square brackets ('[' and ']') are section names. _all_ is a section
for things that apply to all VMs in this inventory. The section name ending in
'_:vars_' contain variable key/values. Almost every name except _all_ is a valid
name for any section name.

When you have created this file (with the proper values within `<` and `>` are
substituted for the actual values, you should be able to do

    $ ansible --list-hosts all
      hosts (2):
        dbserver
        webserver
    $ ansible -m ping all
    webserver | SUCCESS => {
        "ansible_facts": {
            "discovered_interpreter_python": "/usr/bin/python3.12"
        },
        "changed": false,
        "ping": "pong"
    }
    dbserver | SUCCESS => {
        "ansible_facts": {
            "discovered_interpreter_python": "/usr/bin/python3.12"
        },
        "changed": false,
        "ping": "pong"
    }

The actual values above may be different in your exact setup, but the `SUCCESS`
values should give an indication that Ansible is able to login to each host
via SSH.

### Troubleshooting

If you get a warning from Ansible that looks something like this

    [WARNING]: Platform linux on host dbserver is using the discovered Python interpreter at /usr/bin/python3.12, but
    future installation of another Python interpreter could change the meaning of that path. See
    https://docs.ansible.com/ansible-core/2.18/reference_appendices/interpreter_discovery.html for more information.

To disable the warning, you may follow the suggestion on the web page above, i.e. in the `ansible.cfg`:

    [defaults]
    ...
    interpreter_python = python3
    ...

This will set the interpreter statically to `python3` which should be available on most modern operating systems.

You can also set it to

    interpreter_python = auto_silent

to make ansible figure out which Python interpreter to use, but stop giving warnings about potential future
incompatibilities.

## QUESTION A

What happens if you run `ansible-inventory --list` in the directory you created above?

## Svar

Med detta kommando får jag hela inventariet, inklusive SSH-nycklar samt användarnamn, i JSON-format. 

## QUESTION B

What happens if you run `ansible-inventory --graph` in the directory you created above?

## Svar

Till skillnad från --list, visar detta kommando inventariet som ett kortfattat träddiagram, för en snabb överblick över grupper och hosts.

## QUESTION C

In the `hosts` file, add a new host with the same hostname as the machine you're running
ansible on:

    [controller]
    <hostname_of_your_machine> ansible_connection=local

Now run:

    $ ansible -m ping all

Study the output of this command.

What does the `ansible_connection=local` part mean?

## Svar

Standardmetoden för Ansible är att ansluta via SSH men ansible_connection=local talar om för Ansible att köra ping lokalt via bash. Inte försöka att SSH:a till maskinen där den faktiskt är.

## BONUS QUESTION

The command `ansible-config` can be used for creating, viewing, validating, and listing
Ansible configuration.

Try running

    $ ansible-config --help

Make an initial configuration file with the help of this command, and write it into a file
called `ansible.cfg.init`. HINT: Redirections in the terminal can be done with '>' or 'tee(1)'.

Open this file and look at the various options you can configure in Ansible.

In your Ansible working directory where the `ansible.cfg' is, run

    $ ansible-config dump

You should get a pager displaying all available configuration values. How does it differ
from when you run the same command in your usual home directory?

## Svar

I ansible-katalogen:
CONFIG_FILE() = **/home/shilan/ansible/ansible.cfg**

DEFAULT_HOST_LIST(/home/shilan/ansible/ansible.cfg) = ['/home/shilan/ansible/hosts']

HOST_KEY_CHECKING(/home/shilan/ansible/ansible.cfg) = **False**

I min hemkatalog:
CONFIG_FILE() = **None**

DEFAULT_HOST_LIST(default) = ['/etc/ansible/hosts']

HOST_KEY_CHECKING(default) = **True**


#### Skillnaden:

Ansible letar efter en ansible.cfg-fil i aktuell katalog, där man kör kommandot ifrån först. Om den hittar en, får den filen högst prioritet. Dess inställningar läses in och skriver över standardinställningarna. Annars läser den in de inbyggda standardvärden.
Eftersom jag hade en ansible.cfg i min ansible mapp, skrev de över de standardinställningarna medan i min hemkatalog, där jag saknade en ansible.cgf-fil, var det standardärden som lästes in.
Tack vare detta kan man ha olika inställningar för olika projekt.