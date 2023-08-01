# ACME Cert Getter

&copy; 2023 Ian Pilcher <<arequipeno@gmail.com>>

A simple ACME (Let's Encrypt) client that manages a single certificate and
reloads Apache (`httpd`) when necessary.

* [**Installation**](#installation)
* [**Apache configuration**](#apache-configuration)

## Installation

Build the SELinux policy module.  (Ignore any warnings about duplicate macro
definitions.)

```
$ make -f /usr/share/selinux/devel/Makefile
...
Compiling targeted acg module
Creating targeted acg.pp policy package
rm tmp/acg.mod tmp/acg.mod.fc
```

Install the policy module.

```
# semodule -i acg.pp
```

Create the `acg` user (and group).

```
# useradd -c 'ACME Cert Getter' -d /var/lib/acg -r -s /usr/sbin/nologin acg
```

Install the executable.

```
# cp acg /usr/local/bin/
# restorecon /usr/local/bin/acg
```

Set up the configuration directory, which contains Let's Encrypt client key(s).

```
# mkdir -p /etc/acg/private
# cp ${CLIENT_KEY} /etc/acg/private/${HOSTNAME}-acme-client.key
# chown -R acg:acg /etc/acg
# chmod 0500 /etc/acg/private
# chmod 0400 /etc/acg/private/*
# restorecon -R /etc/acg
```

Set up the state directory.

```
# mkdir /var/lib/acg
# cp ${HOSTNAME}.csr /var/lib/acg/
# chown -R acg:acg /var/lib/acg
# restorecon -R /var/lib/acg
```

Create the ACME challenge directory.

```
# mkdir /var/www/acme-challenge
# chown acg:acg /var/www/acme-challenge
# restorecon /var/www/acme-challenge
```

Install the `systemd-tmpfiles` configuration.

```
# cp acg.conf /etc/tmpfiles.d
```

Install the `systemd` units.

```
# cp acg-httpd-reload.service acg-httpd-reload.timer acg@.service acg@.timer \
	/etc/systemd/system/
# systemctl daemon-reload
```

Enable (and start) the timers.

```
# systemctl enable acg-httpd-reload.timer acg@${HOSTNAME}.timer --now
Created symlink /etc/systemd/system/timers.target.wants/acg-httpd-reload.timer → /etc/systemd/system/acg-httpd-reload.timer.
Created symlink /etc/systemd/system/timers.target.wants/acg@${HOSTNAME}.timer → /etc/systemd/system/acg@.timer.
```

## Apache configuration

Configure Apache to serve the ACME challenge directory
(`/var/www/acme-challenge`) at `/.well-known/acme-challenge`.  For example:

```
Alias /.well-known/acme-challenge/ /var/www/acme-challenge/
```

Configure Apache to use the ACG-managed certificate.  For example:

```
SSLCertificateFile /var/lib/acg/${HOSTNAME}.crt
```
