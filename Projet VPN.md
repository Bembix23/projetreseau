# Projet VPN 

## Introduction

**Le VPN est une technologie utile utilisée par des dizaines de millions de personnes, VPN (traduit de l'anglais Virtual Private Network - réseau privé virtuel) est une technologie spéciale qui fournit une connexion réseau sécurisée et cryptée sur un autre réseau. 
C’est un outil essentiel pour la navigation générale. La fraude sur Internet est en hausse, ce qui signifie que la sécurité Internet est plus importante que jamais.**

**Lorsque vous utilisez Internet, vous visitez des sites Web, faites des achats dans des magasins en ligne, consultez vos pages sur les réseaux sociaux, etc. À l'aide d'outils spéciaux, n'importe qui peut suivre les activités sur Internet et utiliser les informations reçues à des fins personnelles. Un VPN aide à éviter cela et offre également un large éventail d'options en ligne.**

**Le VPN permet aux utilisateurs d’être connecté sur un réseau sécurisé et crypté, cela signifie que toutes les informations envoyées ou reçues via le VPN sont protégées. Cela présente plusieurs avantages : les données sensibles sont sûres, les sites susceptibles d'être bloqués par les réseaux publics sont accessibles et l'utilisateur reste anonyme.
Avec la montée de la criminalité sur Internet, le VPN gratuit est un outil inestimable.**

**VPN montre virtuellement l'appareil que vous connectez à Internet dans un emplacement différent. Pour cela, il ouvre un tunnel crypté vers le réseau opposé. Les informations transmises sont mutuellement partagées via ce tunnel. Ainsi, si vous vous connectez à Wikipédia via VPN, les données circuleront entre votre appareil et le réseau VPN auquel il est connecté (par exemple: Canada). Étant donné que ce flux est chiffré, il ne peut pas être visualisé de l'extérieur. Donc le trafic se fait à travers le Canada, vous êtes sur Internet selon les procédures réseau du Canada.**

---


## Installation

Pour suivre cette installation, il vous faudra : 

1. Un serveur Ubuntu 20.04 pour OpenVPN en SSH

2. Votre machine locale comme client OpenVPN



### Etape 1: Installation d'OpenVPN et d'Easy-RSA

Mettre à jour l'index des paquets: 

```
sudo apt update
```

Installer OpenVPN et Easy-RSA: 

```
sudo apt install openvpn easy-rsa
```

On créer un dossier :

```
mkdir ~/easy-rsa
```

On créer un lien entre ce dossier et du site easyrsa installé par le paquet qui nous permettra de faire la mise à jour automatique dans ce dossier :

```
ln -s /usr/share/easy-rsa/* ~/easy-rsa/
```

On donne les droits sur ce dossier à notre utilisateur : 

```
sudo chown sacha ~/easy-rsa
chmod 700 ~/easy-rsa
```

### Etape 2: Créer une infra d'ip publique(ICP) pour OpenVPN:

L'ICP permettra de demander et gérer les certificats TLS pour les clients et les autres serveurs qui se connecteront à notre VPN.

Pour cela on doit créer un fichier et modifier vars dans easyrsa

```
cd ~/easy-rsa
nano vars
```

et dedans : 

```
set_var EASYRSA_ALGO "ec"
set_var EASYRSA_DIGEST "sha512"
```

Ces deux lignes nous permettront de générer des clés et des signatures sécurisées pour les clients du serveur OpenVPN.

On peut donc créer le répertoire ICP. On utilise donc le script easyrsa grace à init-pki:

```
./easyrsa init-pki
```

On peut donc passer à la prochaine étape qui consiste à créer une demande de certificat et une clé privée.

### Étape 3 - Demande de certificat de serveur OpenVPN et clé privée

L'objectif est de générer une clé privée (exemple: fichier.key) et une demande de signature de certificat (CSR).
Puis on la fait signer pour créer le certificat.
Tout d'abord : 

```
cd ~/easy-rsa
```

```
./easyrsa gen-req server nopass
```

Permet de créer la clé et la demande:
```
Output
Common Name (eg: your user, host, or server name) [server]:

Keypair and certificate request completed. Your files are:
req: /home/sacha/easy-rsa/pki/reqs/server.req
key: /home/sacha/easy-rsa/pki/private/server.key
```

On copie ensuite la clé du serveur dans /etc/openvpn/server:

```
sudo cp /home/sammy/easy-rsa/pki/private/server.key /etc/openvpn/server/
```

### Étape 4 - Signer la demande de certificat

Pour cette partie on a suivi ce tutoriel:

https://www.xpresservers.com/how-to-set-up-and-configure-a-certificate-authority-ca-on-ubuntu-20-04/

### Étape 5 - tls-crypt

Cet étape permet de rajouter de la sécurité à notre VPN. Elle agit au niveau des certificats, elle obscurcit les certificats, elle peut gérer le trafic non authentifié et rend plus difficile l'identification du trafic du réseau OpenVPN.

Donc on génère la clé tls-crypt:

```
cd ~/easy-rsa
openvpn --genkey --secret ta.key
```

Cela nous donnera un fichier ta.key que l'on copiera vers /etc/openvpn/server/:

```
sudo cp ta.key /etc/openvpn/server
```

On peut donc créer maintenant des certificats et des clés pour les clients afin de nous connecter en VPN.

### Etape 6 - Génération d'un certificat de client et d'une paire de clés

Pour que tout fonctionne bien, il faut générer un certificat et une clé par client/utilisateur.
Pour cette installation nous allons générer qu'une seule paire certificat/clé pour un seul utilisateur.
Nous devons créer un endroit où les clés vont être stockées:
```
mkdir -p ~/clients-conf/clés
```
On peut lui donner les droits:
```
chmod -R 700 ~/clients-conf
```

On va ensuite créer cette fameuse clé grâce au script EasyRSA qui nous le permet:
```
cd ~/easy-rsa
./easyrsa gen-req nom_client nopass
```

Le fichier nom_client.key est donc généré et contient sa clé, il faut donc déplacer ce fichier dans notre dossier créé juste avant:
```
cp pki/private/nom_client.key ~/clients-conf/clés/
```

On transfère le fichier ".req" créé par le script dans notre serveur d'autorité de certification:
```
scp pki/reqs/nom_client.req sacha@ip_certification:/tmp

```

Passez ensuite sur ce serveur afin d'y mettre le certificat, et on le signe:
```
cd ~/easy-rsa
./easyrsa import-req /tmp/nom_client.req nom_client

./easyrsa sign-req client nom_client

```

Cela va créer un fichier en ".crt" que l'on doit transférer sur le serveur OpenVPN:
```
scp pki/issued/nom_client.crt sacha@ip:/tmp
```

Sur le serveur OpenVPN, il faut que le certificat créé auparavant soit bien placé:
```
cp /tmp/nom_client.crt ~/clients-conf/clés/
```

Ensuite il faut aussi bien placer les fichiers ca.crt et ta.key dans notre dossier /clés, on mets ensuite les bonnes permissions pour les fichiers dans le dossier:
```
cp ~/easy-rsa/ta.key ~/clients-conf/clés/
sudo cp /etc/openvpn/server/ca.crt ~/clients-conf/clés/
sudo chown sacha.sacha ~/clients-conf/clés/*
```

### Etape 7 - Configurer OpenVPN

OpenVPN possède beaucoup d'outils de configuration, tout d'abord nous allons déplacer le fichier server.conf.gz et le dézipper:
```
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/server/
sudo gunzip /etc/openvpn/server/server.conf.gz
```
Ca nous crée donc le fichier server.conf, on doit aller le modifier et décommenter quelques lignes, ce sont les suivantes:
```
sudo nano /etc/openvpn/server/server.conf
```
Nous devons tout d'abord commenter la ligne "tls-auth" en ajoutant un ";" et ajouter une ligne "ls-crypt ta.key":
```
;tls-auth ta.key 0
tls-crypt ta.key
```

Ce sont les lignes de type de code d'authentification de message utilisant la fonction de hachage cryptographique.

On fait la même chose avec les lignes concernant le chiffrement, "AES-256-CBC" et "AES-256-GCM":
```
;cipher AES-256-CBC
cipher AES-256-GCM
```

Et juste après ces deux lignes nous devons rajouter la ligne désignant l'algorithme de digestion des messages HMAC, nous utilisons SHA256 qui est la fonction de hachage la plus optimisée:
```
auth SHA256
```
Il est ensuite nécessaire de commenter les lignes concernant les paramètres de Diffie-Hellman (dh). Effectivement nous n'en avons pas besoin car nous avons déjà configuré nos certificats auparavant. Nous pouvons même rajouter une ligne "dh none":
```
;dh dh2048.pem
dh none
```

Ensuite nous voulons que OpenVPN tourne sans privilèges, sans utilisateur ni group, il faut donc trouver les lignes correspondantes dans le fichier, et on les décommente:
```
user nobody
group nogroup
```

### Etape 8 - Ajustement de la configuration réseau du serveur OpenVPN

Pour qu'OpenVPN puisse fonctionner correctement, nous devons paramétrer comme il se doit l'acheminement réseau, il faut alors modifier le fichier "sysctl.conf":
```
sudo nano /etc/sysctl.conf
```
 Il faut seulement rajouter cette ligne en bas du fichier:
 ```
net.ipv4.ip_forward = 1
```

Vous pouvez quitter ce fichier en l'enregistrant et grâce à ceci, le serveur OpenVPN sera en mesure de transférer le trafic entrant d'un appareil à un autre.

### Etape 9 - Configuration du pare-feu

Après avoir configuré les clés et les certificats, nous devons maintenant fournir à OpenVPN les instructions sur l'endroit où envoyer le trafic web des clients.

Il faut alors installer ufw (Uncomplicated Firewall) et le mettre en route, il va créer des dossiers et fichiers. Celui qui nous interesse ici est:
```
sudo nano /etc/ufw/before.rules
```
Comme le nom du fichier le stipule, ce sont les règles exécutées avant le chargement des règles du firewall conventionnelles. Il faut donc rajouter en haut du fichier ces lignes suivantes:
```
# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0]
# Allow traffic from OpenVPN client to eth0 (change to the interface you discovered!)
-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
COMMIT
# END OPENVPN RULES
```
Il ne faut pas oublier de remplacer "eth0" par le nom de l'interface de votre route par defaut.
Vous pouvez ensuite fermer en enregistrant.

Nous devons ensuite indiquer au Firewall d'autoriser les paquets transmis par défaut:
```
sudo nano /etc/default/ufw
```
Nous devons trouver la ligne "DEFAULT_FORWARD_POLICY" et changer sa valeur qui est actuellement "DROP" en "ACCEPT":
```
DEFAULT_FORWARD_POLICY="ACCEPT"
```

Fermez le fichier en l'enregistrant.

Le pare-feu doit être lui même paramétré pour qu'OpenVPN puisse fonctionner. Il faut donc modifier le port et le protocole dans le fichier "/etc/openvpn/server.conf" et ouvrir le trafic UDP au port 1194. Si ce port avait déjà été modifié auparavant, il faut utiliser les valeurs que vous avez prescrites:
```
sudo ufw allow 1194/udp
sudo ufw allow OpenSSH
```

Un redémarrage de UFW est ensuite nécessaire pour exécuter les modifications:
```
sudo ufw disable
sudo ufw enable
```


### Etape 10 - Démarrer OpenVPN

Nous pouvons utiliser systemctl pour gérer OpenVPN. On veut que notre OpenVPN se lance tout seul au démarrage. Pour ce faire, activez le service OpenVPN en l'ajoutant à systemctl : 

```
sudo systemctl -f enable openvpn-server@server.service
```
 
Ensuite, lancez le service OpenVPN :

```
sudo systemctl start openvpn-server@server.service
```
 
Vérifiez que le service OpenVPN est actif avec la commande suivante. Vous devriez voir "active" (en cours) dans le résultat : 

```
sudo systemctl status openvpn-server@server.service
```

La configuration serveur est terminée , il faut maintenant se diriger sur sa machine client/hôte pour s'y connecter.


### Étape 11 – Création de l'infrastructure de configuration client

Chaque client OpenVPN doit avoir sa configuration et doit respecter les paramètres du fichier de configuration du serveur.
Au lieu de cela, nous utilisons une infrastructure de configuration client permettant de générer plusieurs fichiers de ces fichiers de configuration. Nous allons d'abord créer un fichier de configuration « type », puis un script qui sert à générer des fichiers de configuration client, des certificats et des clés uniques.

Créer un nouveau répertoire pour stocker les fichiers de configuration du client:

```
mkdir -p ~/client-configs/files
```

Copier un fichier de configuration client pour l'utiliser comme configuration de base :
```
cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf
```
Ouvrir le dossier
```
nano ~/client-configs/base.conf
```
Dans celui-ci, trouver "remote". Le client est dirigé vers l'adresse de notre serveur OpenVPN. Si on modifie le port sur lequel le serveur OpenVPN écoute, il faut changer 1194 pour le port sélectionné :
```
. . .
remote your_server_ip 1194
. . . 
```


Décommenter user et group :
```
. . .
user nobody
group nogroup
. . . 
```
Trouvez ca, cert et key. Commenter les car les certificats et les clés seront ajouté dans le dossier :
```
;ca ca.crt
;cert client.crt
;key client.key
```

Commenter la tls-auth, car a.key sera ajouté directement dans le fichier de configuration client:
```
;tls-auth ta.key 1
```
 
Indiquer les paramètres de chiffrement et auth définis dans le fichier /etc/openvpn/server/server.conf :
```
cipher AES-256-GCM
auth SHA256
```

 
Ajouter key-direction dans le fichier avec paramètre sur « 1 » pour que le VPN fonctionne correctement:
```
key-direction 1
```

 
## Script pour un nouveau client

### Création d'un nouvel utilisateur


Créer un fichier de script (".sh") pour copier notre code de création d'utilisateur. Dans le fichier coller ce code :
```
#!/bin/bash
#
# Sacha Voisin
#


echo
echo "Donner un nom à votre client:"
read -p "Nom: " unsanitized_client
client=$(sed 's/[^0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ_-]/_/g' <<< "$unsanitized_client")
while [[ -z "$client" || -e /etc/openvpn/server/easy-rsa/pki/issued/"$client".crt ]]; do
    echo "$client: nom invalide."
    read -p "Nom: " unsanitized_client
    client=$(sed 's/[^0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ_-]/_/g' <<< "$unsanitized_client")
done
cd /etc/openvpn/server/easy-rsa/
EASYRSA_CERT_EXPIRE=3650 ./easyrsa build-client-full "$client" nopass
    # Générer un fichier client.ovpn
    {
    cat /etc/openvpn/server/client-common.txt
    echo "<ca>"
    cat /etc/openvpn/server/easy-rsa/pki/ca.crt
    echo "</ca>"
    echo "<cert>"
    sed -ne '/BEGIN CERTIFICATE/,$ p' /etc/openvpn/server/easy-rsa/pki/issued/"$client".crt
    echo "</cert>"
    echo "<key>"
    cat /etc/openvpn/server/easy-rsa/pki/private/"$client".key
    echo "</key>"
    echo "<tls-crypt>"
    sed -ne '/BEGIN OpenVPN Static key/,$ p' /etc/openvpn/server/tc.key
    echo "</tls-crypt>"
    } > ~/"$client".ovpn
echo
echo "$client ajoutée. La configuration est disponible dans:" ~/"$client.ovpn"
exit
```

Cela compile tous les fichiers de certificats et de clés créés pour le client, extrait leur contenu, les ajoute à la copie du fichier puis exporte tout ce contenu dans un nouveau fichier de configuration du client. Au lieu de devoir s'occuper séparement de chacun  de ses fichiers ils sont tous reunis au même endroit.

Une fois cela fait, vous pouvez exécuter le script avec la commande:
```
./nomdevotrefichier.sh
```

Le script va donc vous demander le nom de votre client, puis il va créer le fichier ".ovpn" correspondant à votre nom de client.


Ce fichier va se trouver dans le dossier /root de votre machine, pour les besoins de la suite, nous devons changer l'emplacement de celui-ci. Nous le mettons donc dans le dossier /tmp :
```
mv /root/nomdevotreutilisateur.ovpn /tmp
```

Ensuite nous devons utiliser ce fichier dans le logiciel OpenVPN Connect, vous devez donc l'installer.
Une fois installé, vous devez avoir cela en accueil:

![](https://i.imgur.com/iv6AvnA.png)

Pour choisir notre fichier ovpn, on va dans "File" puis on va chercher notre fichier. 

Si, comme dans notre cas, le fichier ne se trouve pas sur la même machine que votre "OpenVPN Connect", suivez ceci.

Ouvrez un Windows Powershell et executez cette commande :
```
scp -r -p nom@ip:/tmp/nomdevotreutilisateur.ovpn .
```

Ca récupère le fichier sur votre machine virtuelle et la met sur votre machine de base.
Ainsi vous aurez sur votre machine de base le fichier qui nous intéresse pour le mettre dans OpenVPN Connect.

Suivez ensuite ce que vous dit le logiciel, et vous serez connecté au VPN.

Pour vérifier ceci, vous pouvez aller jeter un coup d'oeil au site 
https://www.ip-tracker.org/

Il peut même vous dire où se situe le server auquel vous êtes connecté.



## NetData

### Ajouter Netdata pour faire du monitoring

#### définition

Netdata fournit des informations inégalées , en temps réel , sur tout ce qui se passe sur les systèmes sur lesquels il s'exécute, à l'aide de tableaux de bord Web hautement interactifs .

Netdata nous permet de faire de la surveillance en temps réel des performances et de l' intégrité d'un système et d'applications. Il s'agit d'un agent de surveillance hautement optimisé.

#### Installation

Pour l'installer il nous suffit d'utiliser cette commande:

```bash <( curl -Ss https://my-netdata.io/kickstart.sh )```

Cette commande nous permet de:

-Détecte la distribution Linux et installe les packages système requis pour la construction de Netdata.

-Télécharge la dernière arborescence source de Netdata sur /usr/src/netdata.git.

-Installe Netdata en s'exécutant à ./netdata-installer.shpartir de l'arborescence source.

-Imprime un message indiquant si l'installation a réussi ou échoué à des fins d'assurance qualité.

Ensuite on autorise le port suivant afin de se connecter et on reload les firewalls:

**```sudo firewall-cmd --add-port=19999/tcp --permanent```**

**```sudo firewall-cmd --reload```**

Puis accéder à http://HOST:19999/ après avoir remplacé HOST par l'adresse IP de ce système. Après ça vous avez donc accés au tableau de bord de netdata et vous pouvez explorer et connaitre les infos du système.

#### Usage

On peut donc connaitre l’usage CPU, RAM, des disques avec chaque partition mais également le détail des applications. Cependant, à la différence d'autres solutions, Netdata ne liste absolument aucune information pouvant présenter un risque sur le plan sécurité. L’avantage est qu’il n’est pas nécessaire de limiter l’accès à la dashboard et cette dernière dispose de son propre web serveur qui ne démarre que lorsque vous consulter la page.

![](https://i.imgur.com/V86i4On.png)


## Borg et Cron

Nous avons utiliser borg et cron afin de faire des backups quotidiennes de nos fichiers.

On a initialiser borg:

```
mkdir save
borg init -e repokey /save
```

Puis on a fait un script pour automatiser les sauvegardes des clients:

```
nano save.sh
```
 on oublie pas de lui donner les bon droits.
 
 et dedans:
 
```
borg create /save::Client-$(date '+%m-%d-%Y') /root/clients/*
```

et enfin du coté de cron:

```
crontab -e
```

et dedans on ajoute:

```
00 07 * * * /home/ubuntu/save.sh
```

Elle fera une sauvegardes tout les jours à 7h que l'on trouveras dans le dossier /save.