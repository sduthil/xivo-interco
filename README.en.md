Configuring multiple XiVO with status updates
=============================================

WARNING1: The different XiVO that you interconnect should be of the same version.

WARNING2: If you clone a VM, or any other duplication of the XiVO database, the UUID of the two XIVO will be the same, so you must regenerate one of them in the table `infos` from database `asterisk`, then restart all services. You must also make consul forget about the old UUID (cf. consul configuration)

WARNING3: Telephony will be interrupted during the configuration period.

WARNING4: This procedure is written for XiVO >= 15.19

WARNING5: You must ensure all ports necessary for communication are open (cf. [Network section in the XiVO doc](http://documentation.xivo.fr/en/stable/contributors/network.html)).

WARNING6: The configuration must be applied on each XiVO that you want to interconnect. For example, if I want to interconnect 6 different XiVO, I have to add for on every XiVO the configuration for the 5 other XiVO. This does not apply to federation policies, where you may use a ring topology (each XiVO talks to its two neighbors)

WARNING7: You should use your firewall to restrict access to the HTTP ports of consul and xivo-ctid, because they don't have any authentication mechanism enabled.

WARNING8: In an architecture with a lot of XiVO, we recommand that you centralize some services, such as xivo-dird, to make your life easier. Don't forget redundancy. This applies also to RabbitMQ and Consul. In this case, the configuration will have to be done entirely manually in YAML config files.


Diagram
-------

![screenshot](schemas/xivo_n2.png?raw=true "schema")


In the rest of this procedure, IP addresses represent:

* XiVO 1: 192.168.1.124
* XiVO 2: 192.168.1.125


Authorize a Webservice user
---------------------------

The first thing is to make XiVO accept remote connections to your internal users directory. For this, you must create a Webservice access by authorizing either an IP address or a login/password.

![create user ws](screenshots/create_user_ws.png?raw=true "create user ws")
![list user ws](screenshots/user_ws.png?raw=true "list user ws")


Add the new source of contacts
------------------------------

Then, you have to add this remote source of contact by creating a new directory source in Configuration->Management->Directories. We recommand that you start without certificate verification, and once you get it working, enable certificate verification, because it is easy to get blocked by this.

![list contact source server](screenshots/list_contact_source_server.png?raw=true "list contact source server")
![add contact source server](screenshots/xivo_add_directory_xivo.png?raw=true "add contact source server")

Then you must map the remote fields to your representation of contacts in Services->CTI Server->Directories->Definitions.

![list definition server](screenshots/definition_server.png?raw=true "list definition server")
![configure definition](screenshots/configure_definition.png?raw=true "configure definition")

Then allow your users to search in the new directory.

![add directory](screenshots/add_directory.png?raw=true "add directory")
![add server directory](screenshots/add_server_directory.png?raw=true "add server directory")

Once all this configuration is done, you must restart your xivo-dird service. You can do so from Services->IPBX, or in the command line:

    service xivo-dird restart


Configure RabbitMQ
------------------

Create a RabbitMQ user

    rabbitmqctl add_user xivo xivo
    rabbitmqctl set_user_tags xivo administrator
    rabbitmqctl set_permissions -p / xivo ".*" ".*" ".*"
    rabbitmq-plugins enable rabbitmq_federation

Restart RabbitMQ

    service rabbitmq-server restart


Setup the federation

    rabbitmqctl set_parameter federation-upstream xivo-dev-2 '{"uri":"amqp://xivo:xivo@192.168.1.125","max-hops":1}'  # remote IP address
    rabbitmqctl set_policy federate-xivo 'xivo' '{"federation-upstream-set":"all"}' --priority 1 --apply-to exchanges


Configure the CTI server
------------------------

Create a configuration file for xivo-ctid, e.g `/etc/xivo-ctid/conf.d/interconnection.yml`

    rest_api:
      http:
        listen: 0.0.0.0
    service_discovery:
      advertise_address: 192.168.1.124  # IP address of this XiVO, reachable from outside
      check_url: http://192.168.1.124:9495/0.1/infos

Restart the CTI server

    service xivo-ctid restart

Ensure the xivo-ctid service is registered with the right IP address, reachable from the others CTI servers

    > apt-get install consul-cli
    > consul-cli agent-services --ssl --ssl-verify=false
    {
      "consul": {
        "ID": "consul",
        "Service": "consul",
        "Tags": [],
        "Port": 8300,
        "Address": ""
    },
      "e546a652-e290-47e2-8519-ec3642daa6e6": {
        "ID": "e546a652-e290-47e2-8519-ec3642daa6e6",
        "Service": "xivo-ctid",
        "Tags": [
          "xivo-ctid",
          "607796fc-24e2-4e26-8009-cbb48a205512"
        ],
        "Port": 9495,
        "Address": "192.168.1.124"  # IP address of this XiVO, reachable from outside
      }
    }

Configure consul
----------------

Make a backup of the KV database. This is not a precautionary measure, we are going to need the backup.

    xivo-backup-consul-kv -o /tmp/backup-consul-kv.json

Stop the XiVO services:n

    xivo-service stop

Wipe out every data in Consul. This is mandatory when changing the IP configuration.

    rm -rf /var/lib/consul/raft/
    rm -rf /var/lib/consul/serf/
    rm -rf /var/lib/consul/services/
    rm -rf /var/lib/consul/tmp/

Add a new config file to consul, e.g. `/etc/consul/xivo/interconnection.json`:

    {
      "client_addr": "0.0.0.0",
      "bind_addr": "0.0.0.0",
      "advertise_addr": "192.168.1.124"  # IP address of this XiVO, reachable from outsidejoignable de l'extérieur
    }

To check that your file is correct:

    consul configtest --config-dir /etc/consul/xivo/

Restart Consul:

    service consul start

Restore the previous backup of the KV database:

    xivo-restore-consul-kv -i /tmp/backup-consul-kv.json

Start all services:

    xivo-service start

Then join a consul member. This has to be done only once per XiVO.

    consul join -wan 192.168.1.125  # remote IP address

Check the membership status:

    consul members -wan
    consul monitor


XiVO Client
-----------

There is no further configuration needed, you should now be able to connect your XiVO Client and lookup contacts from the People Xlet. When looking up contacts of another XiVO, you should see their phone status, their user availability, and agent status dynamically.


Debug
-----

Chances are that everything won't work the first time, here are some interesting commands to help you debug the problem:

    tail -f /var/log/xivo-dird.log
    tail -f /var/log/xivo-ctid.log
    tail -f /var/log/xivo-confd.log
    consul monitor
    consul members -wan
    consul-cli agent-services --ssl --ssl-verify=false
    rabbitmqctl eval 'rabbit_federation_status:status().'
