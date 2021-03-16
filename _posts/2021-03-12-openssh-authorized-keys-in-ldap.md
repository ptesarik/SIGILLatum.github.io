---
layout: post
title: "OpenSSH: Store authorized keys in LDAP"
date: "2021-03-12"
tags: [guide, shell, ldap, ssh]
description:
  A guide on storing OpenSSH authorized keys with OpenLDAP, complete with
  code.
  
  
  **UPDATE:** Added instructions for SSSD.
---

Get the [schema file](/files/openssh-ldap.schema) and save it as
`/etc/ldap/schema/openssh-ldap.schema`.

Unfortunately, OpenLDAP no longer supports this file format directly, so it
must be translated to LDIF format first. I wrote a tiny configuration file and
saved it as `convert.conf`:

    include /etc/ldap/schema/openssh-ldap.schema

Run `slaptest` and copy the resulting file to the final destination:

    slaptest -f convert.conf -F ldif
    cp 'ldif/cn=config/cn=schema/cn={0}openssh-ldap.ldif' /etc/ldap/schema/openssh-ldap.ldif

Now, add the schema to your LDAP server configuration (server URI and
authentication options may vary):

    ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/openssh-ldap.ldif

Next, create a dedicated system user for querying the SSH attribute:

    useradd --system --user-group --home-dir /nonexistent --no-create-home --shell /bin/false sshd-ldap

Now is a good time to adjust access rights to your LDAP database. Get the UID
and GID of the newly created user (UID 124, GID 129 on my system) and add an
attribute like this to your `olcDatabase=<your_db_name>,cn=config`:

	olcAccess: to * attrs=sshPublicKey 
	 by self write 
	 by dn.exact=gidNumber=129+uidNumber=124,cn=peercred,cn=external,cn=auth read

If you use SSSD on your system, add `ssh` to the `services` option in your
`/etc/sssd/sssd.conf`, restart `sssd` and add these options to your
`/etc/ssh/sshd_config`:

	AuthorizedKeysCommand /usr/bin/sss_ssh_authorizedkeys
	AuthorizedKeysCommandUser nobody

If you don't use SSSD, download a [shell script](/files/ldap-authorized-keys),
customize it to match
your needs (change the base object DN, add a server URI, etc.) and install it
in a suitable directory:

    install -o root -g root ldap-authorized-keys /usr/lib/openssh/

Then add these options to `/etc/ssh/sshd_config` instead:

	AuthorizedKeysCommand /usr/lib/openssh/ldap-authorized-keys
	AuthorizedKeysCommandUser sshd-ldap

Last but not least, reload the SSH daemon configuration:

    systemctl reload sshd

You can now add SSH public keys with an LDIF file like this:

	dn: uid=user,ou=users,dc=example,dc=com
	changetype: modify
	add: objectClass
	objectClass: ldapPublicKey
	-
	add: sshPublicKey
	sshPublicKey: <user's public key>
