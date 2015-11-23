Configuration multi XiVO
========================

TODO1: Ajouter encrypt dans la configuration pour chiffrer le RPC.

WARNING1: Il est plus prudent de connecter des XiVO ensemble avec une même version.

WARNING2: En cas de clonage d'une VM, ou toute autre duplication de la base de données XiVO, le UUID du XiVO est le même, donc il faut le changer dans la table `infos` de la base `asterisk`, puis tout relancer. Il faut aussi faire du ménage dans consul (cf. configuration de consul).

WARNING3: Nécessite une coupure de service lors de la configuration

WARNING4: Fonctionne à partir de 15.18.

WARNING5: Vous devez vérifier que tous les ports nécessaires pour les communications réseaux sont bien ouverts (cf. [section networking dans la doc xivo](http://documentation.xivo.fr/en/stable/contributors/network.html))

WARNING6: La configuration doit être faite pour chacun des XiVO que vous souhaitez raccorder ensemble. Donc par exemple si j'ai 6 XiVO à raccorder, il faudra ajouter la configuration sur chaque XiVO pour les 5 autres XiVO. Excepté sur la federation policies, où on peut utiliser une topologie en anneau (chaque XiVO connait deux voisins).

WARNING7: Ajouter des règles de firewall pour restreindre les accès aux ports HTTP de xivo-ctid et consul qui ne sont pas authentifiés pour le moment dans un environnement public.

WARNING8: Dans une architecture avec beaucoup de XiVO, il est conseillé de centraliser certains services comme xivo-dird pour simplifier la gestion. Pensez à les mettre en HA. Ceci est valable aussi pour rabbitmq et consul. La configuration sera manuelle avec des fichiers de configuration YAML.

Schema
------

![screenshot](/schemas/xivo_n2.png?raw=true "schema")

Pour la suite, les adresses IP sont:

* XiVO 1: 192.168.1.124
* XiVO 2: 192.168.1.125

Autoriser un utilisateur webservice
------------------------------------

Vous devez en premier lieu sur chaque XiVO autoriser une connexion distante sur votre annuaire d'utilisateurs interne. Pour cela vous devez créer un accès webservice en l'autorisant soit par IP soit par login/pass.

![create user ws](/screenshots/create_user_ws.png?raw=true "create user ws")
![list user ws](/screenshots/user_ws.png?raw=true "list user ws")

Ajout de la nouvelle source de contacts
---------------------------------------

Après il faut ajouter cette source de contacts en créant le serveur xivo-confd du XiVO d'en face.

Aller dans Configuration->Management->Répertoire

![list contact source server](/screenshots/list_contact_source_server.png?raw=true "list contact source server")
![add contact source server](/screenshots/add_contact_source_server.png?raw=true "add contact source server")

Aller dans Services->Serveur CTI->Répertoires->Définition

Ensuite vous devez ajouter cette source de contact dans votre serveur de contact.

![list definition server](/screenshots/definition_server.png?raw=true "list definition server")
![configure definition](/screenshots/configure_definition.png?raw=true "configure definition")

Puis l'ajouter dans vos recherches autorisées.

![add directory](/screenshots/add_directory.png?raw=true "add directory")
![add server directory](/screenshots/add_server_directory.png?raw=true "add server directory")


Configuration Dird
------------------

Pour xivo-dird une fois configuré le xivo-confd dans l'interface web, nous devons ajouter des modifications que l'interface web ne supporte pas encore. Pour cela ajouter un fichier de config personnalisé pour xivo-dird dans le répertoire `/etc/xivo-dird/conf.d`.

Par exemple, créer un fichier `/etc/xivo-dird/conf.d/xivodev2.yml`

    sources:
      xivodev2:  # le nom de la source de contacts dans Services->Serveur CTI->Répertoires->Définition
        confd_config:
          username: sylvain  # le login de l'accès webservice créé plus haut
          password: sylvain  # le mot de passe de l'accès webservice créé plus haut
          verify_certificate: false  # autres valeurs: `true` si le certificat du XiVO d'en face est signé, `/chemin/vers/le/certificat/distant` s'il est autosigné

Relancer xivo-dird

    service xivo-dird restart

Configurer RabbitMQ
-------------------

Créer un utilisateur RabbitMQ

    rabbitmqctl add_user xivo xivo
    rabbitmqctl set_user_tags xivo administrator
    rabbitmqctl set_permissions -p / xivo ".*" ".*" ".*"
    rabbitmq-plugins enable rabbitmq_federation

Relancer RabbitMQ

    service rabbitmq-server restart

Configurer la fédération

    rabbitmqctl set_parameter federation-upstream xivo-dev-2 '{"uri":"amqp://xivo:xivo@192.168.1.125","max-hops":1}'  # adresse IP distante
    rabbitmqctl set_policy federate-xivo 'xivo' '{"federation-upstream-set":"all"}' --priority 1 --apply-to exchanges

Pour connaître le statut

    rabbitmqctl eval 'rabbit_federation_status:status().'


Configurer le serveur CTI
-------------------------

Faire un fichier de configuration pour xivo-ctid sur chaque serveur, par exemple `/etc/xivo-ctid/conf.d/interconnection.yml` sur chaque serveur.

    rest_api:
      http:
        listen: 0.0.0.0
    service_discovery:
      advertise_address: 192.168.1.124  # adresse IP locale joignable de l'extérieur
      check_url: http://192.168.1.124:9495/0.1/infos

Relancer le serveur CTI

    service xivo-ctid restart

Vérifier que le service est bien enregistré avec une IP joignable par les autres serveurs CTI.

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
        "Address": "192.168.1.124"  # doit être l'adresse IP locale joignable de l'extérieur
      }
    }


Configurer consul
-----------------

Faire un backup de la base KV avant.

    xivo-backup-consul-kv -o /tmp/backup-consul-kv.json

Arrêter les services XiVO

    xivo-service stop

Faire un clean de tout consul (obligatoire car changement de config IP)

    rm -rf /var/lib/consul/raft/
    rm -rf /var/lib/consul/serf/
    rm -rf /var/lib/consul/services/
    rm -rf /var/lib/consul/tmp/


Ajouter un fichier de configuration consul, par exemple `/etc/consul/xivo/interconnection.json`

    {
      "client_addr": "0.0.0.0",
      "bind_addr": "0.0.0.0",
      "advertise_addr": "192.168.1.124"  # adresse IP locale joignable de l'extérieur
    }

Pour vérifier si le fichier est correct.

    consul configtest --config-dir /etc/consul/xivo/

Relancer consul.

    service consul start

Faire une restauration de la base KV.

    xivo-restore-consul-kv -i /tmp/backup-consul-kv.json

Relancer les services.

    xivo-service start

Puis joindre un membre.

    consul join -wan 192.168.1.125  # adresse IP distante

Pour vérifier le status.

    consul members -wan
    consul monitor

XiVO client
-----------

Vous n'avez rien à faire, vous pouvez maintenant vous connecter et rechercher à partir de la Xlet people n'importe quelle personne de vos différents XiVO et vous aurez également leur présence téléphonique, utilisateur et agent.


Debug
-----

Si ça ne fonctionne pas du premier coup, les commandes intéressantes pour diagnostiquer le problème:

    tail -f /var/log/xivo-dird.log
    tail -f /var/log/xivo-ctid.log
    tail -f /var/log/xivo-confd.log
    consul monitor
    consul members -wan
    consul-cli agent-services --ssl --ssl-verify=false
