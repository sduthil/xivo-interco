Configuration multi XiVO
========================

WARNING1: Il est plus prudent de connecter des xivo ensemble avec une même version.

WARNING2: En cas de copie d'une VM à une autre ou du meme genre le UUID est le même, donc il faut le changer. dans la table infos de la base asterisk. puis tout relancer. Il faut aussi faire du menage dans consul.

WARNING3: Nécessite une coupure de service lors de la configuration

WARNING4: Fonctionne à partir de 15.18.

WARNING5: Vous devez vérifier que tous les ports nécessaires pour les communications réseaux sont bien ouverts (cf. section networking dans la doc xivo)

WARNING6: La configuration doit être faite pour chacun des XiVO que vous souhaitez raccorder ensemble. Donc par exemple si j'ai 6 XiVO à raccorder, il faudra ajouter la config pour 6 XiVO par XiVO. Excepter sur la federation policies.

WARNING7: Ajouter des régles de firewall sur le port http de ctid et consul qui ne sont pas authentifiés pour le moment dans un environnement ouvert à tous.

Schema
------

![screenshot](/schemas/xivo_n2.png?raw=true "schema")

Autoriser un utilisateur webservice
------------------------------------

Vous devez en premier lieu sur vos XiVO autoriser une connexion distante sur votre annuaire d'utilisateur interne. Pour cela vous devez créer un utilisateur en l'autorisant soit par IP soit par login/pass.

![create user ws](/screenshots/create_user_ws.png?raw=true "create user ws")
![list user ws](/screenshots/user_ws.png?raw=true "list user ws")

Ajout XiVO
----------

Après il faut ajouter cette source en créant le serveur.

Aller dans IPBX->Serveur CTI->Management->Directories

![list contact source server](/screenshots/list_contact_source_server.png?raw=true "list contact source server")
![add contact source server](/screenshots/add_contact_source_server.png?raw=true "add contact source server")

Après vous devez ajouter ce source de contact dans votre serveur de contact.

![add directory](/screenshots/add_directory.png?raw=true "add directory")
![add server directory](/screenshots/add_server_directory.png?raw=true "add server directory")

Puis ajouter l'ajouter dans vos recherches authorisées.

![list definition server](/screenshots/definition_server.png?raw=true "list definition server")
![configure definition](/screenshots/configure_definition.png?raw=true "configure definition")


Configuration Dird
------------------

Pour dird une fois configuré le confd dans l'interface web, ajouter un utilisateur sur le web service distant ou l'IP puis ajouter un fichier de config dans dird.

genre myconfig.yml dans /etc/xivo-dird/conf.d

Pour trouver le nom du xivo ajouté lancer la commande :

    xivo-confgen dird/sources.yml
    
Ajouter votre fichier myconfig.yml

    sources:
      monxivo:
        confd_config:
          verify_certificate: false

Relancer dird

    service xivo-dird restart

Configurer rabbitMQ
-------------------

    rabbitmqctl add_user xivo xivo
    rabbitmqctl set_user_tags xivo administrator
    rabbitmqctl set_permissions -p / xivo ".*" ".*" ".*"
    rabbitmq-plugins enable rabbitmq_federation

Relancer rabbitMQ

    service rabbitmq-server restart

Configurer la fédération

    rabbitmqctl set_parameter federation-upstream xivo-dev-2 '{"uri":"amqp://xivo:xivo@192.168.1.125","max-hops":1}'
    rabbitmqctl set_policy federate-xivo 'xivo' '{"federation-upstream-set":"all"}' --priority 1 --apply-to exchanges

Pour connaître le status

    rabbitmqctl eval 'rabbit_federation_status:status().'


Configurer CTI
---------------

Faire un fichier custom dans /etc/xivo-ctid/conf.d sur chaque serveur ex. myconfig.yml

    rest_api:
      http:
        listen: 0.0.0.0
    service_discovery:
      advertise_address: 192.168.1.124
      check_url: http://192.168.1.124:9495/0.1/infos

Relancer le serveur CTI

    service xivo-ctid restart

Vérifier que le service est bien enregistré avec une IP joignable par les autres serveurs CTI.

    apt-get install consul-cli

    consul-cli agent-services --ssl --ssl-verify=false

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
        "Address": "192.168.1.124"
      }
    }


Configurer consul
-----------------


Faire un backup des données avant.

    xivo-backup-consul-kv -o /tmp/backup-consul-kv.json

Arrêter les services XiVO

    xivo-service stop

Faire un clean de tout consul (obligatoire car changement de config ip)

    rm -rf /var/lib/consul/raft/
    rm -rf /var/lib/consul/serf/
    rm -rf /var/lib/consul/services/
    rm -rf /var/lib/consul/tmp/


Ajouter un fichier dans /etc/consul/xivo/ par exemple myconfig.json

    {
      "client_addr": "0.0.0.0",
      "bind_addr": "0.0.0.0",
      "advertise_addr": "192.168.1.124"
    }

Pour vérifier si le fichier est correct.

    consul configtest --config-dir /etc/consul/xivo/

Relancer consul

    service consul start

Faire un restaure

    xivo-restore-consul-kv -i /tmp/backup-consul-kv.json

Relancer les services

    xivo-service start

Puis joindre un membre 

    consul join -wan 192.168.1.125

Pour vérifier

    consul members -wan
    consul monitor
