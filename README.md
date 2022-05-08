# Projet Docker ITS 2

-----------------

## Objectif du projet
Mise en place d'une infrastructure composée d'un client, d'un serveur et d'un firewall. Chacun d'eux sont situés dans un container Docker. 
Afin de faciliter la mise en place des containers nous avons utilisé *Docker_compose*. 

-----------------

## Installation de Docker
Pour débuter l’installation de Docker sur Debian il faut commencer par une mise à jour :
``apt update && apt full-upgrade -y``

Installation des dépendances : 
    ```apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release```
	
Ajout de la clé GPG officielle de Docker :
``curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg``

Ajout du repository Docker dans les sources :
```
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
  
Mise à jour des sources :
``apt update``

Téléchargement de docker depuis les sources :
``apt install docker-ce docker-ce-cli containerd.io``

Vérification de l’installation de Docker :
``docker run hello-world``

-----------------

## Installation de Docker-compose
Installation de Docker-Compose via Github
``sudo curl -L https://github.com/docker/compose/releases/download/v2.0.1/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose``

Changement des permissions
``sudo chmod +x /usr/local/bin/docker-compose``

On vérifie la version installée
``docker-compose --version``

-----------------

## Infrastucture utilisée
```
Projet/
├── docker-compose.yml
│
├── client/
│   ├── client.py   # Envoi une requête au serveur afin de récupérer le contenu de la page web
│   ├── Dockerfile  # Fichier Dockerfile
│
├── server/
│   ├── server.py   # Lance le serveur web
│   ├── Dockerfile  # Fichier Dockerfile
│   ├── index.html  # Page web
│
├── firewall/
│   ├── clean.sh    # Réinitialise toutes les configurations des tables (iptables)
│   ├── Dockerfile  # Fichier Dockerfile
│   ├── firewall.sh # Toutes les règles du firewall
```
-----------------

## Contenu des fichiers

### Docker-compose

Fichier docker-compose.yml : 

```
version: "3"
services:
  server:
    build: server/
    container_name: Serveur
    command: python ./server.py
    ports:
      - 1234:1234

  client:
    build: client/
    container_name: Client
    command: python ./client.py
    network_mode: host
    depends_on:
      - server

  firewall:
    build: firewall/
    container_name: Firewall
    cap_add:
      - NET_ADMIN
    sysctls:
      - net.ipv4.ip_forward=1
    command: >
      bash -c "./clean.sh && ./firewall.sh"
    depends_on:
      - client
```

### Client

Fichier client.py : 

```
#!/usr/bin/env python3

import urllib.request

fp = urllib.request.urlopen("http://localhost:1234/")

encodedContent = fp.read()
decodedContent = encodedContent.decode("utf8")

print(decodedContent)

fp.close()

variable = 10

# Pour garder le container actif
while True:
  if variable != 10:
    break
```

Fichier Dockerfile (client) : 

```
FROM python:latest

ADD client.py /client/

WORKDIR /client/
```

### Server

Fichier index.html : 

```
Serveur web de Thomas
```


Fichier server.py : 

```
#!/usr/bin/env python3

import http.server
import socketserver

handler = http.server.SimpleHTTPRequestHandler

with socketserver.TCPServer(("", 1234), handler) as httpd:
    httpd.serve_forever()
	
```

Fichier Dockerfile (server) :

```
FROM python:latest

ADD server.py /server/
ADD index.html /server/

WORKDIR /server/
```


### Firewall

Fichier clean.sh : 

```
#!/bin/bash

yes | apt update
yes | apt-get install iptables

# On affiche nos chaines initiales
echo '====== Etat chaines initiales ======'
iptables -L

# On efface les configurations dans les chaines INPUT, OUTPUT, FORWARD
iptables -F INPUT
iptables -F OUTPUT
iptables -F FORWARD

# On définit les polices INPUT, OUTPUT, FORWARD en ACCEPT
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT

echo '====== Etat chaines finales ======'
# On affiche nos chaines finales
iptables -L
```


Fichier firewall.sh

```

# Autorise tout le trafic de loopback (lo0) et supprime tout le trafic vers 127/8 qui n'utilise pas lo0
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT ! -i lo -d 127.0.0.0/8 -j REJECT

# Accepte toutes les connexions entrantes établies
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Autorise tout le trafic sortant
iptables -A OUTPUT -j ACCEPT

# Autorise les connexions HTTP et HTTPS de n'importe où (les ports normaux pour les sites Web)iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Autorise les connexions SSH
iptables -A INPUT -p tcp -m state --state NEW --dport 22 -j ACCEPT

# Autoriser les pings
iptables -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT

# Rejeter tous les autres messages entrants - refus par défaut :
iptables -A INPUT -j REJECT
iptables -A FORWARD -j REJECT

# Pour permettre aux noeuds du LAN avec des adresses IP privées de communiquer avec les réseaux public externes
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Pour rendre le serveur disponible de façon externe 
# Retransmettre des requêtes HTTP à notre système de Serveur HTTP 
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 1234 -j DNAT --to 172.17.0.2:1234


# Pour garder le container actif
while true
do
        sleep 1
done
```


Fichier Dockerfile (firewall) :

```
FROM python:latest

ADD clean.sh /firewall/
ADD firewall.sh /firewall/

WORKDIR /firewall/
```

-----------------

## Mettre en place l'infrastructure

Une fois l'infrastructure détaillée ci-dessus est mise en place, il suffit d'effectuer les deux commandes suivants : 

```
docker-compose build
```
```
docker-compose up
```

Le serveur va alors se lancer et mettre à disposition une page web. Le client va effectuer un *curl* afin de récupérer et afficher le contenu de la page. 
Et enfin, le firewall va mettre en place plusieurs règles afin de filtrer les adresses IP accédant au serveur


## Schéma finale de l'infrastructure
```
												VM 															# Machine virtuelle avec une interface eth0 en 192.168.1.67/24
									  (eth0: 192.168.1.67/24)
												|
												|
			--------------------------------------------------------------------------------
			|																			   |
			|							Docker0: 172.17.0.1/16)							   |				# Interface réseau Docker0 de la VM en 172.17.0.1/16
			|																			   |
			--------------------------------------------------------------------------------
				  VethX							VethY						VethZ
					|							  |							  |
					|							  |							  |
				 Serveur 						Client					   Firewall							# Les containers 
			 (172.17.0.2/16)				(172.17.0.3/16)				(172.17.0.4/16)
			
```
