# ldap_server_setup

## Usage

This is a rough draft of a bolt plan that installs and sets up an LDAP server in a docker container.

Yes, I know it is a terrible idea to ship certificates along with the package.  I will replace this with openssl commands soon.

Usage:

```bash
/opt/puppetlabs/bin/bolt plan run ldap_server_setup::setup -t TARGETS
```

For example, using ``-t localhost`` will setup ldap up on the current host.  See Appendix for example output
  
Steps:

1. This does require bolt, so install it first.  I used a bolt.yaml file of:

```bash
# cat ~/.puppetlabs/bolt/bolt.yaml
ssh:
  user: root
  private-key: ~/.ssh/id_rsa-acceptance
  host-key-check: false
```

Of course, this will differ based on setup.

1. Switch into the ldap_server_setup directory
1. Create a directory for forge modules:

```# mkdir modules```

1. Install modules locally:

```bash
# /opt/puppetlabs/bin/bolt puppetfile install -m modules --puppetfile ./Puppetfile
```

5. Use the setup plan to install an LDAP server on a target node:

```bash
# /opt/puppetlabs/bin/bolt plan run ldap_server_setup::setup -m modules:.. -t <TARGET>
```

Once installed, either PE or CDPE can be set up to use it.  It defaults to a bind dn user of `cn=Service Bind User.dc=puppetdebug,dc=vlan` with a password of `password`.  There are two other users present initially:

```yaml
User: "ldapuser1@puppetdebug.vlan"
Password: "ldapuser1"
User: "ldapuser2@puppetdebug.vlan"
Password:  "ldapuser2"
```

Along with a group called `admins`.  All of these can be altered by changing the ldif files prior to installation.  After installation, normal ldapmodify commands will need to be used to make updates to the existing LDAP server.

There are also a couple tasks for connecting PE and CDPE to the created LDAP server:

```bash
# /opt/puppetlabs/bin/bolt task run ldap_server_setup::connectcdpe -m modules:.. -t <target CDPE host> ldap_host=<LDAP server> root_user="<CDPE root user> root_password=<CDPE password>
```

```bash
# /opt/puppetlabs/bin/bolt task run ldap_server_setup::connectpe -m modules:.. -t <target PE master> ldap_server_hostname=<LDAP server>
```

To setup CDPE manually to use this LDAP server, add settings as per the following:

![CDPE Settings](/cdpe-settings.png)

To setup PE manually, use the `ds` endpoint to inject the following:

```json
{
    "base_dn": "dc=puppetdebug,dc=vlan",
    "connect_timeout": 10,
    "disable_ldap_matching_rule_in_chain": false,
    "display_name": "My LDAP",
    "group_lookup_attr": "cn",
    "group_member_attr": "memberUid",
    "group_name_attr": "cn",
    "group_object_class": "posixGroup",
    "group_rdn": null,
    "help_link": "",
    "hostname": "ldap.puppetdebug.vlan",
    "login": "cn=Service Bind User,dc=puppetdebug,dc=vlan",
    "password": "password",
    "port": 636,
    "search_nested_groups": false,
    "ssl": true,
    "ssl_hostname_validation": false,
    "ssl_wildcard_validation": false,
    "start_tls": false,
    "type": null,
    "user_display_name_attr": "cn",
    "user_email_attr": "mail",
    "user_lookup_attr": "cn",
    "user_rdn": ""
}
```

Change the "hostname" to match the target where the bolt plan installed the ldap server.

## Appendix

### Setup sample output

```bash
root@hapam-c38f73-0 ~/development/tools/puppet/@usage/modules/abottchen/ldap_server_setup (development)$ pbpr ldap_server_setup::setup -t localhost
+ /opt/puppetlabs/bin/bolt plan run --verbose ldap_server_setup::setup -t localhost
Starting: plan ldap_server_setup::setup
Starting: plan ldap_server_setup::createcerts
Starting: install puppet and gather facts on localhost
Finished: install puppet and gather facts with 0 failures in 1.89 sec
Starting: apply catalog on localhost
Started on localhost...
Finished on localhost:
  Notice: /Stage[main]/Main/File[/etc/ldapserver]/ensure: created
  Notice: /Stage[main]/Main/File[/etc/ldapserver/certs]/ensure: created
  Notice: /Stage[main]/Main/File[/var/lib/ldap]/ensure: created
  Notice: /Stage[main]/Main/File[/etc/ldap]/ensure: created
  Notice: /Stage[main]/Main/File[/etc/ldap/slapd.d]/ensure: created
  Notice: /Stage[main]/Main/File[/etc/ldapserver/certs/ca.pem]/ensure: defined content as '{sha256}c0af81c98e4dc036aeec9352064ce07d4adf3ee30dac9abf3a365c2255483f2b'
  Notice: /Stage[main]/Main/File[/etc/ldapserver/certs/ldap.key]/ensure: defined content as '{sha256}3a7a58069afdafc80dfc23f9c76ee3dba5d34e3ecc1ee2ec3de29864c086f345'
  Notice: /Stage[main]/Main/File[/etc/ldapserver/certs/ldap.crt]/ensure: defined content as '{sha256}3a7a58069afdafc80dfc23f9c76ee3dba5d34e3ecc1ee2ec3de29864c086f345'
  Notice: /Stage[main]/Main/File[/etc/pki/ca-trust/source/anchors/ca.pem]/ensure: defined content as '{sha256}c0af81c98e4dc036aeec9352064ce07d4adf3ee30dac9abf3a365c2255483f2b'
  Notice: /Stage[main]/Main/Exec[update trust]: Triggered 'refresh' from 1 event
  changed: 10, failed: 0, unchanged: 0 skipped: 0, noop: 0
Finished: apply catalog with 0 failures in 3.92 sec
Finished: plan ldap_server_setup::createcerts in 5.82 sec
Starting: plan ldap_server_setup::install_docker
Starting: install puppet and gather facts on localhost
Finished: install puppet and gather facts with 0 failures in 1.82 sec
Starting: apply catalog on localhost
Started on localhost...
Finished on localhost:
  Notice: /Stage[main]/Docker::Repos/Yumrepo[docker]/ensure: created
  Notice: /Stage[main]/Docker::Install/Package[docker]/ensure: created
  Notice: /Stage[main]/Docker::Service/File[/etc/sysconfig/docker-storage-setup]/ensure: defined content as '{sha256}8b192c418a4bfcdd63a54fec1bdd99ddb03f0543e9c2c059a494a2280636415f'
  Notice: /Stage[main]/Docker::Service/File[/etc/systemd/system/docker.service.d]/ensure: created
  Notice: /Stage[main]/Docker::Service/File[/etc/systemd/system/docker.service.d/service-overrides.conf]/ensure: defined content as '{sha256}3bd857f6bb451132d507d8f05446abd4436ca38808b87cbb8046e2438d7b22eb'
  Notice: /Stage[main]/Docker::Service/Exec[docker-systemd-reload-before-service]: Triggered 'refresh' from 1 event
  Notice: /Stage[main]/Docker::Service/File[/etc/sysconfig/docker-storage]/ensure: defined content as '{sha256}1a0bfc5c866936867a3519a80104980d85c5ba5c91a5575f0fc0c5266cc98571'
  Notice: /Stage[main]/Docker::Service/File[/etc/sysconfig/docker]/ensure: defined content as '{sha256}f601c95aa6bb7f9f490ec066780a4c44a2e8d8837aa94ece46d0a1d544839479'
  Notice: /Stage[main]/Docker::Service/Service[docker]/ensure: ensure changed 'stopped' to 'running'
  Notice: /Stage[main]/Main/Docker::Run[ldap]/File[/usr/local/bin/docker-run-ldap-start.sh]/ensure: defined content as '{sha256}9d05ad546ce7396fea42b3533a24628706e683d59fc8439146559e1eae81cf33'
  Notice: /Stage[main]/Main/Docker::Run[ldap]/File[/usr/local/bin/docker-run-ldap-stop.sh]/ensure: defined content as '{sha256}de6bc579d0a6c4a1ac77e6b9dc5f434616785b5a1182ae313fee720f3b67e591'
  Notice: /Stage[main]/Main/Docker::Run[ldap]/File[/etc/systemd/system/docker-ldap.service]/ensure: defined content as '{sha256}488abb0404c4ff20738a01da468ee8fe288ebafdf3ea74f394c6130954482444'
  Notice: /Stage[main]/Main/Docker::Run[ldap]/Exec[docker-ldap-systemd-reload]: Triggered 'refresh' from 3 events
  Notice: /Stage[main]/Main/Docker::Run[ldap]/Service[docker-ldap]/ensure: ensure changed 'stopped' to 'running'
  changed: 14, failed: 0, unchanged: 1 skipped: 0, noop: 0
Finished: apply catalog with 0 failures in 42.4 sec
Starting: task ldap_server_setup::waitfordocker on localhost
Started on localhost...
Finished on localhost:
  Waiting for slapd to start
  Waiting for slapd to start
  Waiting for slapd to start
  Waiting for slapd to start
  Waiting for slapd to start
  Waiting for slapd to start
  Waiting for slapd to start
  Waiting for slapd to start
  Waiting for slapd to start
  Waiting for slapd to start
  LDAP container running!
  62d131ad slapd starting
Finished: task ldap_server_setup::waitfordocker with 0 failures in 50.48 sec
Finished: plan ldap_server_setup::install_docker in 1 min, 35 sec
Starting: plan ldap_server_setup::populate_ldap_entries
Starting: install puppet and gather facts on localhost
Finished: install puppet and gather facts with 0 failures in 2.08 sec
Starting: apply catalog on localhost
Started on localhost...
Finished on localhost:
  Notice: /Stage[main]/Main/Package[openldap-clients]/ensure: created
  Notice: /Stage[main]/Main/File[/etc/ldapserver/ou.ldif]/ensure: defined content as '{sha256}d07d42a09a34149d87647d94f03850f9a21cb7e280df58b6ea5247cec4abe54f'
  Notice: /Stage[main]/Main/File[/etc/ldapserver/user.ldif]/ensure: defined content as '{sha256}a8ebf12d0e5ffd78d7f187c217c47c82a21eab4447db208754e76fd0a4094d7e'
  Notice: /Stage[main]/Main/File[/etc/ldapserver/groups.ldif]/ensure: defined content as '{sha256}13c13968892378f56d710b0e9de8f34e52b393599c8ab3c6cb2b4776ce6a82e8'
  Notice: /Stage[main]/Main/File[/var/lib/docker/volumes/ldap_data/_data/indexmod.ldif]/ensure: defined content as '{sha256}969f8c921d374b85fd4c84ff948f0f175dca820bb3ed6edbafabbddddad564f3'
  Notice: /Stage[main]/Main/Exec[import ou]/returns: executed successfully
  Notice: /Stage[main]/Main/Exec[import user]/returns: executed successfully
  Notice: /Stage[main]/Main/Exec[import groups]/returns: executed successfully
  Notice: /Stage[main]/Main/Exec[import indexmod]/returns: executed successfully
  changed: 9, failed: 0, unchanged: 0 skipped: 0, noop: 0
Finished: apply catalog with 0 failures in 9.09 sec
Finished: plan ldap_server_setup::populate_ldap_entries in 11.18 sec
Starting: task ldap_server_setup::test on localhost
Started on localhost...
Finished on localhost:
  # extended LDIF
  #
  # LDAPv3
  # base <dc=puppetdebug,dc=vlan> with scope subtree
  # filter: (objectclass=*)
  # requesting: cn 
  #
  
  # puppetdebug.vlan
  dn: dc=puppetdebug,dc=vlan
  
  # admin, puppetdebug.vlan
  dn: cn=admin,dc=puppetdebug,dc=vlan
  cn: admin
  
  # Service Bind User, puppetdebug.vlan
  dn: cn=Service Bind User,dc=puppetdebug,dc=vlan
  cn: Service Bind User
  
  # Group, puppetdebug.vlan
  dn: ou=Group,dc=puppetdebug,dc=vlan
  
  # People, puppetdebug.vlan
  dn: ou=People,dc=puppetdebug,dc=vlan
  
  # ldap, People, puppetdebug.vlan
  dn: uid=ldap,ou=People,dc=puppetdebug,dc=vlan
  cn: OpenLDAP server
  
  # ldapuser1, People, puppetdebug.vlan
  dn: uid=ldapuser1,ou=People,dc=puppetdebug,dc=vlan
  cn: ldapuser1
  
  # ldapuser2, People, puppetdebug.vlan
  dn: uid=ldapuser2,ou=People,dc=puppetdebug,dc=vlan
  cn: ldapuser2
  
  # admins, Group, puppetdebug.vlan
  dn: cn=admins,ou=Group,dc=puppetdebug,dc=vlan
  cn: admins
  
  # search result
  search: 2
  result: 0 Success
  
  # numResponses: 10
  # numEntries: 9
Finished: task ldap_server_setup::test with 0 failures in 0.03 sec
Finished: plan ldap_server_setup::setup in 1 min, 52 sec
Finished on localhost:
  # extended LDIF
  #
  # LDAPv3
  # base <dc=puppetdebug,dc=vlan> with scope subtree
  # filter: (objectclass=*)
  # requesting: cn 
  #
  
  # puppetdebug.vlan
  dn: dc=puppetdebug,dc=vlan
  
  # admin, puppetdebug.vlan
  dn: cn=admin,dc=puppetdebug,dc=vlan
  cn: admin
  
  # Service Bind User, puppetdebug.vlan
  dn: cn=Service Bind User,dc=puppetdebug,dc=vlan
  cn: Service Bind User
  
  # Group, puppetdebug.vlan
  dn: ou=Group,dc=puppetdebug,dc=vlan
  
  # People, puppetdebug.vlan
  dn: ou=People,dc=puppetdebug,dc=vlan
  
  # ldap, People, puppetdebug.vlan
  dn: uid=ldap,ou=People,dc=puppetdebug,dc=vlan
  cn: OpenLDAP server
  
  # ldapuser1, People, puppetdebug.vlan
  dn: uid=ldapuser1,ou=People,dc=puppetdebug,dc=vlan
  cn: ldapuser1
  
  # ldapuser2, People, puppetdebug.vlan
  dn: uid=ldapuser2,ou=People,dc=puppetdebug,dc=vlan
  cn: ldapuser2
  
  # admins, Group, puppetdebug.vlan
  dn: cn=admins,ou=Group,dc=puppetdebug,dc=vlan
  cn: admins
  
  # search result
  search: 2
  result: 0 Success
  
  # numResponses: 10
  # numEntries: 9
Successful on 1 target: localhost
Ran on 1 target
+ set +x
root@hapam-c38f73-0 ~/development/tools/puppet/@usage/modules/abottchen/ldap_server_setup (development)$ 
```
