

This Document describes how to configure tower for ldap authentication 

The First thing we need is a user in the ldap which has access to read the whole ldap structure.
We can test if we can make successful queries to the ldap server via the below command. 

```

ldapsearch -x  -H ldap://win -D "CN=benz,CN=Users,DC=benno,DC=com" -b"dc=benno,dc=com" -w Ben4Cloud

```


Here “cn=benz,cn=users,dc=benno,dc=com” is the Distinguished Name of the connecting user. 

This can be added to the Parameter “AUTH_LDAP_BIND_DN =“ to specify the user that tower will use to connect (Bind) to the ldap server.

The next parameter would be:
-------------------------------

AUTH_LDAP_BIND_PASSWORD = 'Ben4Cloud'
Which specifies the password to use for the Binding user.

The next section specifies where to search for users while authenticating.
------------------------------------------------------------------------

```

AUTH_LDAP_USER_SEARCH = LDAPSearch(
    'DC=BENNO,DC=COM',   # Base DN
    ldap.SCOPE_SUBTREE,             # SCOPE_BASE, SCOPE_ONELEVEL, SCOPE_SUBTREE
    '(sAMAccountName=%(user)s)',    # Query
)

```


The First line specifies where to search for users in the ldap Tree. so in the above example the users would be searched in recursively starting from “DC=Benno,DC=COM”
The Second line specifies the scope where the users should be searched 

```
		BASE This value is used to indicate searching only the entry at the base DN, resulting in only that entry being returned 
		ONELEVEL This value is used to indicate searching all entries one level under the base DN - but not including the base DN and not including any entries under that one level under the base DN. 
		SUBTREE This value is used to indicate searching of all entries at all levels under and including the specified base DN.  
```

The Third line specifies the key name where the username is stored. for example if we query the AD ldap using for ldapsearch with filter for user as below 

```

ldapsearch -x  -H ldap://win -D "CN=benz,CN=Users,DC=benno,DC=com" -b"dc=benno,dc=com" -w Ben4Cloud objectClass=user

We would get something like:

# Administrator, Users, benno.com
dn: CN=Administrator,CN=Users,DC=benno,DC=com
objectClass: user
cn: Administrator
description: Built-in account for administering the computer/domain
distinguishedName: CN=Administrator,CN=Users,DC=benno,DC=com
instanceType: 4
objectGUID:: lEphZCrQQUKedwWDg/KciQ==
accountExpires: 0
logonCount: 162
sAMAccountName: Administrator
sAMAccountType: 805306368

```

Here we see that name is stored in key ‘sAMAccountName’ hence the line becomes 
'(sAMAccountName=%(user)s)'
similarly for openldap the key is ‘uid’ hence the line becomes 
'(uid=%(user)s)',


The next directives specifies the user attributes.
--------------------------------------------------

```

AUTH_LDAP_USER_ATTR_MAP = {
'first_name': 'givenName',
'last_name': 'sn',
'email': 'mail',
}

```

The above example says that the users last name can be got from the key ‘sn’ in the ldapsearch. we can use the same ldap query for the user to figure out under what keys they are stored.

The Next directive specifies the where the groups should shoudl be searched and how to search them.
---------------------------------------

```

AUTH_LDAP_GROUP_SEARCH = LDAPSearch(
'DC=benno,DC=com',    # Base DN
ldap.SCOPE_SUBTREE,     # SCOPE_BASE, SCOPE_ONELEVEL, SCOPE_SUBTREE
'(objectClass=group)',  # Query
)

```

The first line specifies the BASE DN where the groups should be searched. 
The second lines specifies the scope and is the same as that for the user directive.
The third line specifies what is the objectclass of a group object in the ldap you are using. 
You could make a ldapsearch and check in one group which is the objectclass that it belongs to .
eg:

```

# admin, grp, benno.com
dn: CN=admin,OU=grp,DC=benno,DC=com
objectClass: top
objectClass: group
cn: admin
member: CN=both,CN=Users,DC=benno,DC=com
distinguishedName: CN=admin,OU=grp,DC=benno,DC=com

```

Group Type
--------------

The next directive specifies the type of group.
The supported ones are listed here http://pythonhosted.org/django-auth-ldap/groups.html#types-of-groups

AUTH_LDAP_GROUP_TYPE = ActiveDirectoryGroupType()

With these directives filled in and others commented we should be able to make a successful authentication with LDAP.


Miscelleneous
------------------------

LDAPS:
-------

To enable secure ldap communication with the ldap server change the ldap url to ldaps in directive “AUTH_LDAP_SERVER_URI “
Also make sure the server name in the URI matches the name in the certificate, Also add the server certificate to your tower instance by adding the path which in centos is /etc/openldap/ldap.conf and the directive is  "TLS_CACERT /etc/openldap/certs/cert.pem"

Incase we want to disable the certificate check we can add the following lines to the ldap.py file

```

AUTH_LDAP_GLOBAL_OPTIONS = {
ldap.OPT_X_TLS_REQUIRE_CERT: False,
}

```

Debugging:
----------

Debugging ldap connections can be enabled by adding the below lines in the ldap.py file.

```

LOGGING['handlers']['syslog'] = {
    'level': 'DEBUG',
    'filters': ['require_debug_false'],
    'class': 'logging.handlers.SysLogHandler',
    'address': '/dev/log',
    'facility': 'local0',
    'formatter': 'simple',
}

LOGGING['loggers']['django_auth_ldap']['handlers'] = ['syslog']
LOGGING['loggers']['django_auth_ldap']['level'] = 'DEBUG'

```

Referrals:
-------------

Active directory uses ‘referrals’ in case the queried object is not available in it’s database, we have seen that this doesnt wok properly with the django ldap client and most of the times it helps to disable referrals. we can disable ldap referrals by adding the below lines in ldap.py

```

AUTH_LDAP_GLOBAL_OPTIONS = {
ldap.OPT_REFERRALS: False,
}

```



