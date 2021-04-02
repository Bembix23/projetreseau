# Création d'un nouvel utilisateur

Toutes ces étapes doivent se faire après avoir créé le serveur openvpn via les commandes dans notre fichier "...".

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

Suivez ensuite ce que vous dit le logiciel, et vous serez connecté à une autre IP.

Pour vérifier ceci, vouss pouvez aller jeter un coup d'oeil au site 
"https://www.ip-tracker.org/" 

Il peut même vous dire où se situe le server auquel vous êtes connecté.

