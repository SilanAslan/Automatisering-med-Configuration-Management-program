# Examination 1 - Understanding SSH and public key authentication

Connect to one of the virtual lab machines through SSH, i.e.

    $ ssh -i deploy_key -l deploy webserver

Study the `.ssh` folder in the home directory of the `deploy` user:

    $ ls -ld ~/.ssh

Look at the contents of the `~/.ssh` directory:

    $ ls -la ~/.ssh/

## QUESTION A

What are the permissions of the `~/.ssh` directory?

Why are the permissions set in such a way?

## Svar
- rwx - 700
shilan@shilan-Precision-Tower-3620:~/Desktop/vagrant$ ls -ld ~/.ssh
drwx------ 2 shilan shilan 4096 Oct 16 11:40 /home/shilan/.ssh


- SSH kräver att endast användaren själv har tillgång till katalogen för att skydda privata nycklar från andra användare på systemet.

## QUESTION B

What does the file `~/.ssh/authorized_keys` contain?

## Svar

Innehåller de publika SSH-nycklar som tillåts logga in utan lösenord som användaren deploy.
```
[deploy@webserver ~]$ cat ~/.ssh/authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKARUSJQwSK8h6TgVu+ocDRoa1gaWjXxsN1XFTEsj4zr deploy key
```

## QUESTION C

When logged into one of the VMs, how can you connect to the
other VM without a password?

## Svar

Jag skapade ett nytt SSH-nyckelpar på webservern och lade till den publika nyckeln i filen authorized_keys på dbservern.
```
 [deploy@webserver ~]$ ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
```
Jag kopierade den publika nyckeln: 
```
 [deploy@webserver ~]$ cat ~/.ssh/id_ed25519.pub
  ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOlRKm6Dfa6rXV0QyECpmCe/NNwQnQVcKPtpQYb8gpqu deploy@webserver
```
och klistrade in den på /home/deploy/.ssh/authorized_keys på dbservern via vi.

Efter det kunde jag ansluta från webservern till dbservern utan att behöva ange lösenord:
```
 [deploy@webserver ~]$ ssh deploy@192.168.121.158
 Last login: Mon Oct 27 23:39:04 2025 from 192.168.121.1
 [deploy@dbserver ~]$ 
```

### Hints:

* man ssh-keygen(1)
* ssh-copy-id(1) or use a text editor

## BONUS QUESTION

Can you run a command on a remote host via SSH? How?

## Svar

Ja, man kan köra ett kommando på en fjärrmaskin via SSH utan att starta en interaktiv session genom att skriva kommandot direkt efter SSH-målet, till exempel:
```
[deploy@webserver ~]$ ssh deploy@192.168.121.158 hostname
dbserver

[deploy@webserver ~]$ ssh deploy@192.168.121.158 ls -l /home/deploy
total 0
[deploy@webserver ~]$ 
```