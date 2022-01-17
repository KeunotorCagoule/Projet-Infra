# Projet - Infra

## Setup users + groups
```sh
# Create a group
$ sudo groupadd admin
# Check if it's ok
$ grep admin /etc/group
admin:x:1001:

# Create user with his home directory (-m), a password (-p), and don't create a group with the same name as the user (-N)
$ sudo useradd -m -p <password> -N <username>
# Check if it's ok
$ grep <username> /etc/passwd
<username>:x:1003:100::/home/<username>:/bin/sh

# Set the group "admin" as the primary group of the user
$ sudo usermod -g admin <username>

# Add user in the group sudo to use sudo commands
$ usermod -aG sudo <username>
# Check if it's ok
$ grep sudo /etc/group
sudo:x:27:debian,gabriel,alexis,roxanne

```
