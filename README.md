## openLDAP

Openldap is a client/server protocol used to access and manage directory information and store username and password .

LDAP, or lightweight directory access protocol

1. ***install openldap***

		$ sudo pacman -S openldap

2. ***Configuration server***

	2.1 The server configuration file is located at  `/etc/openldap/slapd.conf`.
	
	Edit the suffix and rootdn. The suffix typically is your domain name but it 	does not have to be. It depends on how you use your directory. We will use _example_ for the domain name, and _com_ for the tld. The rootdn is your LDAP administrator's name (we will use _Manager_ here).

		suffix     "dc=example,dc=local"
		rootdn     "cn=Manager,dc=example,dc=local"

	2.2 Now we delete the default root password and create a strong one:

		sed -i "/rootpw/ d" /etc/openldap/slapd.conf #find the line with rootpw and delete it
		echo "rootpw    $(slappasswd)" >> /etc/openldap/slapd.conf  #add a line which includes the hashed password output from slappasswd


	2.3 Add some typically used [schemas](http://www.openldap.org/doc/admin24/schema.html) to the top of `slapd.conf`:
	
		include         /etc/openldap/schema/cosine.schema
		include         /etc/openldap/schema/inetorgperson.schema
		include         /etc/openldap/schema/nis.schema
		#include         /etc/openldap/schema/samba.schema

	2.4 Add some typically used [indexes](http://www.openldap.org/doc/admin24/tuning.html#Indexes) to the bottom of `slapd.conf`:

		index   uid             pres,eq
		index   mail            pres,sub,eq
		index   cn              pres,sub,eq
		index   sn              pres,sub,eq
		index   dc              eq
3. ***Create User***
		
		$ sudo useradd -mg users -G wheel,power,storage -s /bin/bash your_user
		
5. ***Configure LDAP Server***

	4.1 Create initial entry

		$ sudo nano base.ldif
		dn: dc=example,dc=local  
		dc: example  
		o: example Organization  
		objectClass: dcObject  
		objectClass: organization  
		  
		dn: cn=Manager,dc=example,dc=local  
		cn: Manager  
		description: LDAP administrator  
		objectClass: organizationalRole  
		objectClass: top  
		roleOccupant: dc=example,dc=local  
		  
		dn: ou=Account,dc=example,dc=local  
		ou: Account  
		objectClass: top  
		objectClass: organizationalUnit  
		  
		dn: ou=Marketing,dc=example,dc=local  
		ou: Marketing  
		objectClass: top  
		objectClass: organizationalUnit  
		  
		dn: ou=Sale,dc=example,dc=local  
		ou: Sale  
		objectClass: top  
		objectClass: organizationalUnit
	
	4.2 Add entry
	
		ldapadd -x -D cn=Manager,dc=example,dc=local -W -f base.ldif
		Enter LDAP Password:
		adding new entry "dc=example,dc=local"
		
		adding new entry"cn=Manager,dc=example,dc=local"
		
		adding new entry"ou=Account,dc=example,dc=local"

		adding new entry"ou=Marketing,dc=example,dc=local"

		adding new entry"ou=Sale,dc=example,dc=local"


	4.3 confirm settings

		$ slapcat  
		 
		dn: dc=example,dc=local  
		dc: example  
		o: example Organization  
		objectClass: dcObject  
		objectClass: organization  
		structuralObjectClass: organization  
		entryUUID: fe6ddbed-15fa-4ee8-9651-2c2d4345a9b9  
		creatorsName: cn=Manager,dc=example,dc=local  
		createTimestamp: 20190428095513Z  
		entryCSN: 20190428095513.544597Z#000000#000#000000  
		modifiersName: cn=Manager,dc=example,dc=local  
		modifyTimestamp: 20190428095513Z  
		  
		dn: cn=Manager,dc=example,dc=local  
		cn: Manager  
		description: LDAP administrator  
		objectClass: organizationalRole  
		objectClass: top  
		roleOccupant: dc=example,dc=local  
		structuralObjectClass: organizationalRole  
		entryUUID: 01d2463f-88da-47e6-b031-2159874e8825  
		creatorsName: cn=Manager,dc=example,dc=local  
		createTimestamp: 20190428095513Z  
		entryCSN: 20190428095513.668328Z#000000#000#000000  
		modifiersName: cn=Manager,dc=example,dc=local  
		modifyTimestamp: 20190428095513Z

		dn: ou=Account,dc=example,dc=local  
		ou: Account  
		objectClass: top  
		objectClass: organizationalUnit  
		structuralObjectClass: organizationalUnit  
		entryUUID: 3ce92f9a-3cb6-4b3d-857a-9b9bf0efff12  
		creatorsName: cn=Manager,dc=example,dc=local  
		createTimestamp: 20190428095513Z  
		entryCSN: 20190428095513.768210Z#000000#000#000000  
		modifiersName: cn=Manager,dc=example,dc=local  
		modifyTimestamp: 20190428095513Z  

	4.4 Add user to ldap
	
		$ sudo nano ldapuser.ldif
		
		dn: uid=user1,ou=Account,dc=example,dc=local  
		changetype: modify  
		objectClass: inetOrgPerson  
		objectClass: posixAccount  
		objectClass: shadowAccount  
		sn: user1 
		givenName: user1  
		cn: user1  
		displayName: user1  
		uidNumber: 1002  
		gidNumber: 1002  
		userPassword: {SSHA}6rgTV4xwDFTEgXamybrWofP1+C3Ts6MR  
		gecos: user1  
		loginShell: /bin/bash  
		homeDirectory: /home/user1  
		shadowExpire: -1  
		shadowFlag: 0  
		shadowWarning: 7  
		shadowMin: 0  
		shadowMax: 99999  
		shadowLastChange: 18014  
		  
		dn: cn=user1,ou=Account,dc=example,dc=local  
		changetype: modify  
		objectClass: posixGroup  
		cn: user1  
		gidNumber: 1002  
		memberUid: user1

	4.4 Add entry user

		$ ldapadd -x -D cn=Manager,dc=example,dc=local -W -f ldapuser.ldif
		Enter LDAP Password:  
		adding new entry "uid=user1,ou=Account,dc=example,dc=local"

		adding new entry "cn=user1,ou=Account,dc=example,dc=local"


**Configure LDAP Client**

1. ***Install the OpenLDAP client***

		$ sudo pacman -S nss-pam-ldapd

2. ***NSS Configuration***

	Edit `/etc/nsswitch.conf` which is the central configuration file for NSS. It tells NSS which sources to use for which system databases. We need to add the `ldap` directive to the `passwd`, `group` and `shadow` databases, so be sure your file looks like this:

		passwd: files ldap
		group: files ldap
		shadow: files ldap

3. ***PAM Configuration***

	First edit `/etc/pam.d/system-auth` 

		$ sudo nano /etc/pam.d/system-auth

	**auth      sufficient pam_ldap.so**
	auth      required  pam_unix.so     try_first_pass nullok
	auth      optional  pam_permit.so
	auth      required  pam_env.so

	**account   sufficient pam_ldap.so**
	account   required  pam_unix.so
	account   optional  pam_permit.so
	account   required  pam_time.so

	**password  sufficient pam_ldap.so**
	password  required  pam_unix.so     try_first_pass nullok sha512 shadow
	password  optional  pam_permit.so

	session   required  pam_limits.so
	session   required  pam_unix.so
	**session   optional  pam_ldap.so**
	session   optional  pam_permit.so
	#

	Then edit both `/etc/pam.d/su` 
	
		$ sudo nano /etc/pam.d/su

	#%PAM-1.0
	auth      sufficient    pam_rootok.so
	**auth      sufficient    pam_ldap.so**
	#Uncomment the following line to implicitly trust users in the "wheel" group.
	#auth     sufficient    pam_wheel.so trust use_uid
	 Uncomment the following line to require a user to be in the "wheel" group.
	#auth     required      pam_wheel.so use_uid
	auth      required	pam_unix.so **use_first_pass**
	**account   sufficient    pam_ldap.so**
	account   required	pam_unix.so
	**session   sufficient    pam_ldap.so**
	session   required	pam_unix.so
	#

	Enable users to edit their password,edit `/etc/pam.d/passwd`
	
		$ sudo nano /etc/pam.d/passwd

	#%PAM-1.0
	**password        sufficient      pam_ldap.so**
	#password       required        pam_cracklib.so difok=2 minlen=8 dcredit=2 ocredit=2 retry=3
	#password       required        pam_unix.so sha512 shadow use_authtok
	password        required        pam_unix.so sha512 shadow nullok
	#

	 #### Create home folders at login
	 
	If you want home folders to be created at login (eg: if you are not using NFS to store home folders), edit `/etc/pam.d/system-login` and add `pam_mkhomedir.so` to the _session_ section above any "sufficient" items. And edit `/etc/pam.d/su-l` file is used when the user runs `su --login`. 
	
		$ sudo nano /etc/pam.d/systemd-login

	...top of file not shown...
	session    optional   pam_loginuid.so
	session    include    system-auth
	session    optional   pam_motd.so          motd=/etc/motd
	session    optional   pam_mail.so          dir=/var/spool/mail standard quiet
	-session   optional   pam_systemd.so
	session    required   pam_env.so
	**session    required   pam_mkhomedir.so skel=/etc/skel umask=0022**

		$ sudo nano /etc/pam.d/su-l
		
	...top of file not shown...
	**session         required        pam_mkhomedir.so skel=/etc/skel umask=0022**
	session         sufficient      pam_ldap.so
	session         required        pam_unix.so

	#
	#### Enable sudo

	To enable sudo from an LDAP user, edit `/etc/pam.d/sudo`. You will also need to modify sudoers accordingly.
	
		$ sudo nano /etc/pam.d/sudo

	#%PAM-1.0
	**auth      sufficient    pam_ldap.so**
	auth      required      pam_unix.so  **try_first_pass**
	auth      required      pam_nologin.so
	#
	You will also need to add in `/etc/openldap/ldap.conf` the following.

		$sudo nano /etc/openldap/ldap.conf
		
		sudoers_base ou=sudoers,dc=AFOLA

4. Start service 

		$ sudo systemctl enable nslcd
		$ sudo systemctl start nslcd
		$ sudo systemctl status nslcd

5. ***verity on client***

		# getent passwd user1 
		user1:x:1002:1002::/home/kunthy:/bin/bash