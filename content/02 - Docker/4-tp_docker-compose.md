---
title: 'Docker 4 - Créer une application multiconteneur'
visible: true
---


## Articuler deux images avec Docker compose

- Vérifiez que docker-compose est bien installé avec `sudo apt-get install docker-compose`.
- A la racine de notre projet précédent `identidock` (à côté du Dockerfile), créez un fichier déclaration de notre application `docker-compose.yml` avec à l'intérieur:
  
```yml
version: '2'
services:
  identidock:
    build: .
    ports:
      - "9090:9090"
    environment:
      APP_ENVIRONMENT: DEV
    volumes:
      - ./app:/app
```

- Plusieurs remarques :
  - la première ligne déclare le conteneur de notre application
  - les lignes suivantes permettent de décrire comment lancer notre conteneur
  - `build: .` d'abord l'image d'origine de notre conteneur est le résultat de la construction du répertoire courant
  - la ligne suivante décrit le mapping de ports.
  - on définit ensuite la valeur de l'environnement de lancement du conteneur
  - on définit un volume (le dossier `app` dans le conteneur sera le contenu de notre dossier de code)

  - n'hésitez pas à passer du temps à explorer les options et commandes de `docker-compose`. Ainsi que [la documentation du langage (DSL) des Compose files](https://docs.docker.com/compose/compose-file/).

- Lancez le service (pour le moment mono-conteneur) avec `docker-compose up`
- Visitez la page web.
- Essayez de modifier l'application et de recharger la page web. Voilà comment, grâce à un volume on peut développer sans reconstruire l'image à chaque fois !

- Ajoutons maintenant un deuxième conteneur. Nous allons tirer parti d'une image déjà créée qui permet de récupérer une "identicon". Ajoutez à la suite du fichier Compose ***(attention aux indentations !)*** :

```yml
version: '2'
services:
  identidock:
    build: .
    ports:
      - "9090:9090"
    environment:
      APP_ENVIRONMENT: DEV
    volumes:
      - ./app:/app

    links:
      - dnmonster

  dnmonster:
    image: amouat/dnmonster:1.0
```

Cette fois plutôt de construire l'image, nous indiquons simplement comment la récupérer sur le Docker Hub. Nous ajoutons également un lien qui indique à Docker de configurer le réseau convenablement.


- Ajoutons également un conteneur redis, la base de données qui sert à mettre en cache les images et à ne pas les recalculer à chaque fois :

```yml
version: '2'
services:
  identidock:
    build: .
    ports:
      - "9090:9090"
    environment:
      APP_ENVIRONMENT: DEV
    volumes:
      - ./app:/app

    links:
      - dnmonster

  dnmonster:
    image: amouat/dnmonster:1.0

  redis:
    image: redis:3.0
```

- Et un deuxième lien `- redis`  ***(attention aux indentations !)***.

- Créez un deuxième fichier compose `docker-compose.prod.yml` pour lancer l'application en configuration de production.

- Vérifiez dans les logs de l'application quand l'image a été générée et quand elle est bien mise en cache dans redis.

- N'hésitez pas à passer du temps à explorer les options et commandes de `docker-compose`. Ainsi que [la documentation du langage (DSL) des Compose files](https://docs.docker.com/compose/compose-file/).



## TP 2 : Déployons du nginx et des journaux distribués avec la stack ELK

On se propose ici d'essayer de déployer plusieurs conteneurs nginx.

- A partir de cette [stack d'exemple](https://discuss.elastic.co/t/nginx-filebeat-elk-docker-swarm-help/130512) trouvez comment installer `filebeat` pour récupérer les logs de nginx et les envoyer à un Elasticsearch (décrire sur papier comment faire avant).
  
- Ajoutez un Kibana pour explorer nos logs qui sont dans Elasticsearch.


---

<!-- TODO: wave-ify code -->
```yaml
version: "3"

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.5.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    networks:
      - logging-network

  # logstash:
  #   image: docker.elastic.co/logstash/logstash:7.5.0
  #   depends_on:
  #     - elasticsearch
  #   ports:
  #     - 12201:12201/udp
  #   volumes:
  #     - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
  #   networks:
  #     - logging-network

  filebeat:
    image: docker.elastic.co/beats/filebeat:7.5.0
    user: root
    depends_on:
      - elasticsearch
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - logging-network
    environment:
      - -strict.perms=false

  kibana:
    image: docker.elastic.co/kibana/kibana:7.5.0
    depends_on:
      - elasticsearch
    ports:
      - 5601:5601
    networks:
      - logging-network



  # httpd:
  #   image: httpd:latest
  #   depends_on:
  #     - logstash
  #   ports:
  #     - 80:80
  #   logging:
  #     driver: gelf
  #     options:
  #       # Use udp://host.docker.internal:12201 when you are using Docker Desktop for Mac
  #       # docs: https://docs.docker.com/docker-for-mac/networking/#i-want-to-connect-from-a-container-to-a-service-on-the-host
  #       # issue: https://github.com/lvthillo/docker-elk/issues/1
  #       gelf-address: "udp://localhost:12201"

networks:
  logging-network:
    driver: bridge
```

```yaml
version: '2.2'
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.5.0
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      # - discovery.seed_hosts=es02,es03
      # - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data01:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - elastic
  # es02:
  #   image: docker.elastic.co/elasticsearch/elasticsearch:7.5.0
  #   container_name: es02
  #   environment:
  #     - node.name=es02
  #     - cluster.name=es-docker-cluster
  #     - discovery.seed_hosts=es01,es03
  #     - cluster.initial_master_nodes=es01,es02,es03
  #     - bootstrap.memory_lock=true
  #     - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
  #   ulimits:
  #     memlock:
  #       soft: -1
  #       hard: -1
  #   volumes:
  #     - data02:/usr/share/elasticsearch/data
  #   networks:
  #     - elastic
  # es03:
  #   image: docker.elastic.co/elasticsearch/elasticsearch:7.5.0
  #   container_name: es03
  #   environment:
  #     - node.name=es03
  #     - cluster.name=es-docker-cluster
  #     - discovery.seed_hosts=es01,es02
  #     - cluster.initial_master_nodes=es01,es02,es03
  #     - bootstrap.memory_lock=true
  #     - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
  #   ulimits:
  #     memlock:
  #       soft: -1
  #       hard: -1
  #   volumes:
  #     - data03:/usr/share/elasticsearch/data
  #   networks:
  #     - elastic

volumes:
  data01:
    driver: local
  # data02:
  #   driver: local
  # data03:
  #   driver: local

networks:
  elastic:
    driver: bridge
```
---
  

<!-- Pour les gens en avance :
 Enfin, créez un fichier Docker Compose pour faire fonctionner [l'application Flask finale du TP précédent](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xix-deployment-on-docker-containers)  (à partir du tag git `v0.18`) avec MySQL et Elasticsearch. Puis, centralisez les logs grâce à des tags filebeat -->
