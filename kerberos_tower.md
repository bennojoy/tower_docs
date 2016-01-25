
####The Following article explains how user authentication via AD (kerberos) can be done for Ansible Tower.

---------------------

The first thing we need to setup is the kerberos packages in the tower system so that we can successfully generate a kerberos ticket. For the steps are as follows
Install the packages: (Centos)

```
yum install krb5-workstation
yum install krb5-devel
yum install krb5-libs
pip install kerberos

```

Once installed edit the /etc/krb.conf file as follows to indicate the address of AD is and the Domain etc…

```

[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 default_realm = BENNO.COM
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 
[realms]
 BENNO.COM = {
  kdc = WIN-SA2TXZOTVMV.benno.com
  admin_server = WIN-SA2TXZOTVMV.benno.com
 }

[domain_realm]
 .benno.com = BENNO.COM
 benno.com = BENNO.COM

```



Once the the above configurations are done we should be able to successfully authenticate and get a valid token, 
The below steps shows how to authenticate and get a token.

```

[root@ip-172-31-26-180 ~]# kinit benz
Password for benz@BENNO.COM: 
[root@ip-172-31-26-180 ~]# 

Check if we got a valid ticket.
 
[root@ip-172-31-26-180 ~]# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: benz@BENNO.COM

Valid starting     Expires            Service principal
01/25/16 11:42:56  01/25/16 21:42:53  krbtgt/BENNO.COM@BENNO.COM
	renew until 02/01/16 11:42:56
[root@ip-172-31-26-180 ~]# 

```

Once we have a valid ticket we can check if everything works fine from command line, for that make sure your inventory looks as below 

```

[windows]
win01.BENNO.COM

[windows:vars]
ansible_ssh_user = benz@BENNO.COM
ansible_connection = winrm
ansible_ssh_port = 5986

```


Make sure the hostname is the proper client hostname matching the entry in AD and not the ip address 
also in the username make sure the domain name (text after @)is in proper case as kerberos is case sensitive.
For Tower make sure the inventory looks the same.

After this if we run the playbook it should run fine, Do try running the playbook as user 'awx' .

Once this works integration with Tower is easy. Generate the kerberos ticket as user 'awx' and tower should automatically pickup the 
generated ticket for authentication.
Note: Make sure the python package 'kerberos' is installed. Ansible checks if kerberos package is installed, if so it uses kerberos authentication.

The problem now would be generate the ticket every 24 hours as the default life time of a ticket is 24 hours (can be changed in /etc/krb.conf file)
One solution would be to cron the kinit process every 24 hours. for this to be automated we need to generate a keytab file which stores the
user password and kinit wont prompt for user password, the below steps oultine how to generate this keytab file and then get the kerberos ticket.


```

> ktutil
  ktutil:  addent -password -p benz@BENNO.COM -k 1 -e rc4-hmac
  provide password
  ktutil:  wkt benz.keytab
  ktutil:  quit 

Now we can add the below command to cron

kinit benz@BENNO.COM -k -t benz.keytab

```




