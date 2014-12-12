Zarafa deliver Mail to Public Folder (the Postfix way)
=======
---
Github Readme to [Zarafa deliver Mail to Public Folder (the Postfix way)](http://www.leckerbeef.de/zarafa-deliver-mail-to-public-folder-the-postfix-way/)

---

Zarafa provides different possibilities to deliver incoming Mails into a Public Folder. The first method is to use procmail or with a Zarafa-Dagent Plugin.

The method I describe here uses only postfix with ActiveDirectory user plugin (also works similar with OpenLDAP).

It was tested with these prerequisites

 - Ubuntu Precise 12.04.4 LTS
 - Zarafa 7.1.9.44268
 - Postfix 2.9.6 (with postfix-ldap package)

Preparation of ActiveDirectory
------------------------------
Create a new Organizational Unit (OU)
In your ActiveDirectory tree it should look like this

    dc=mycompany,dc=local
    |_ ou=Zarafa
       |_ ou=sharedFolders

Add some E-Mail addresses
-------------------------
Next we add one or more E-Mail addresses. Right click onto `sharedFolders` and select `New -> Contact` from the context menu. At Full Name enter the E-Mail address you want to use for public delivery (in my example I used `nagios@mycompany.com`). After you created your Contact double click on it and fill `zpublic:Support\Nagios` into the `Description` text box.

Actually you don’t have to create these Public Folders by yourself, `Zarafa-Dagent` will create the Folder it doesn’t exist. 

Postfix
=======

Before we can start we have to check if postfix is configured correct. In your `/etc/postfix/main.cf` you must have configured these settings:

    virtual_mailbox_domains = mycompany.com
    virtual_alias_maps = ldap:/etc/postfix/ldap-virtual.cf
    transport_maps = ldap:/etc/postfix/ldap-transport.cf

ldap-virtual.cf
---------------

`Postfix` must have a successful lookup for the incoming E-Mail address otherwise they get bounced. So we let postfix search for the address, and if it exist, we just return it.

    server_host = 192.168.0.1
    search_base = ou=sharedFolders,ou=Zarafa,dc=mycompany,dc=local
    version = 3
    
    # should be a User with at least read permissions
    bind = yes
    bind_dn = cn=zarafaadmin,ou=SystemOperators,dc=mycompany,dc=local
    bind_pw = VeryStrongPaSSword
        
    scope = sub
    query_filter = (&(objectClass=contact)(cn=%s))
    result_attribute = cn

To verify it is working use `postmap`

    postmap -q nagios@mycompany.com ldap:/etc/postfix/ldap-virtual.cf

It should return the same address as we requested: `nagios@mycompany.com` !

ldap-transport.cf
-----------------

Create an new file called `/etc/postfix/ldap-transport.cf` with the following content

    server_host = 192.168.0.1
    search_base = ou=sharedFolders,ou=Zarafa,dc=mycompany,dc=local
    version = 3

    # should be a User with at least read permissions
    bind = yes
    bind_dn = cn=zarafaadmin,ou=SystemOperators,dc=mycompany,dc=local
    bind_pw = VeryStrongPaSSword
    
    scope = sub
    query_filter = (&(objectClass=contact)(cn=%s))
    result_attribute = description

To verify these settings use `postmap` again

    postmap -q nagios@mycompany.com ldap:/etc/postfix/ldap-transport.cf

But now it should return the tansport method and the foldersettings we entered at the `Description` text box: `zpublic:Support\Nagios`

master.cf
---------

Add the following lines at the bottom of your `/etc/postfix/master.cf`

    zpublic  unix  -       n       n       -       10       pipe
        flags=DORX argv=/usr/bin/zarafa-dagent --public ${nexthop} -C zarafaadmin

I use the user `zarafaadmin` for delivery. It can be any active Zarafa user, but it must have write access to the Public Folders!

You don’t have to configure the E-Mail addresses in your user account!

Verify your new configuration
=============================

First restart Postfix `/etc/init.d/postfix restart`

Now send a text message to postfix. I prefer to use `heirloom-mailx` client.

    echo "test" | mailx -s "testsubject" nagios@mycompany.com

Check your log files. You should see a line that looks similar to these

    # /var/log/mail.log
    zarafa postfix/pipe[2909]: B1E9E206E1: to=<nagios@mycompany.com>, relay=zpublic, delay=0.26, delays=0.01/0/0/0.25, dsn=2.0.0, status=sent (delivered via zpublic service)
    
    # /var/log/zarafa/dagent.log
    [2910] Delivered message to 'zarafaadmin', Subject: "testsubject", Message-Id: <20141211162001.23DB1207BB@mycompany.com>

In Webaccess your E-Mail should be delivered into the correct Public Folder

Done.

Without LDAP
============

If you don’t use a OpenLDAP or ActiveDirectory as user plugin you can also use the method described above.

main.cf
-------

    # /etc/postfix/main.cf
    virtual_mailbox_domains = mycompany.com
    virtual_alias_maps = hash:/etc/postfix/virtual-public
    transport_maps = hash:/etc/postfix/transport-public

master.cf
---------

Insert the same lines into `/etc/postfix/master.cf` like above.

virtual-public
--------------

We do the same as above. Create a new file called `/etc/postfix/virtual-public` and enter the E-Mail addresses twice.

    # /etc/postfix/virtual-public
    nagios@mycompany.com         nagios@mycompany.com
    serveralarm@mycompany.com    serveralarm@mycompany.com

transport-public
----------------

    # /etc/postfix/transport-public
    nagios@mycompany.com         zpublic:Support\Nagios
    serveralarm@mycompany.com    zpublic:Support\Alarm

Finally use `postmap` to create `.db` files and restart postfix

    postmap /etc/postfix/virtual-public
    postmap /etc/postfix/transport-public

    /etc/init.d/postfix restart

Done. Again.


[http://www.leckerbeef.de/](http://www.leckerbeef.de/)
