
## Etape 0

IDE visual studio code installé

maven installé

    sudo apt-get install maven
    mvn -version      # 3.8.7
    si blocage faire ceci
    ***Linux/macOS :
Ajoutez la ligne suivante à votre fichier ~/.bashrc, ~/.bash_profile ou ~/.zshrc :

 - export PATH=/usr/local/apache-maven-3.x.x/bin:$PATH
Chargez les modifications avec :
 - source ~/.bashrc
Vérifier l'installation :
 - mvn -v
Cela devrait afficher la version de Maven installée.

version de java installée

    java -version

Generer la paire de clé ssh

    ssh-keygen -t rsa -b 4096

Il nous faudra ensuite donner les droits les plus restreints à ce fichier par sécurité et pour les exigences d’OpenSSH :

    chmod 700 ~/.ssh -Rf

Installation de docker deja fait



## Etape 1

Le serveur openssh est installé par defaut sur notre serveur et permet de nous connecter initialement avec le compte root via login/mdp
Creons le compte de service

    ssh root@216.225.202.241

    useradd ruth --home /home/ruth/ --create-home  --shell /bin/bash
    usermod -aG sudo ruth
    passwd ruth

exit  # pour me deconnecter

Push de la clé public du compte de service au compte du developpeur

    ssh-copy-id -i  ~/.ssh/id_rsa.pub ruth@216.225.202.241

je me connecte au serveur avec ma clé privé et sans mot de passe

    ssh -i ~/.ssh/id_rsa.pub  ruth@216.225.202.241

Installation de docker

Installer docker sur linux
Ajour des dépots mirroirs pour télécharger les dépendances et mise à jour du systeme

    sudo apt-get update -y & sudo apt-get upgrade -y
    sudo apt-get install ca-certificates curl  gnupg lsb-release python3-pip -y
    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt-get update -y & sudo apt-get upgrade -y

    sudo apt-get install docker-ce docker-ce-cli -y

Pour tester l'installation de docker nous lancons le conteneur hello-world qui naturelement affiche hello world

    sudo docker run hello-world

Exécuter Docker en tant qu’utilisateur sans privilège root

    sudo usermod -aG docker ruth


Installer docker-compose

out d'abord, téléchargez le binaire de Docker Compose à l'aide de la commande suivante :

    curl -s https://api.github.com/repos/docker/compose/releases/latest | grep browser_download_url | grep docker-compose-linux-x86_64 | cut -d '"' -f 4 | wget -qi -
Une fois le téléchargement terminé, définissez la permission d'exécution du binaire téléchargé :

    chmod +x docker-compose-linux-x86_64
Ensuite, déplacez le binaire téléchargé dans le chemin du système :

    sudo mv docker-compose-linux-x86_64 /usr/bin/docker-compose

Maintenant, vérifiez l'installation de Docker Compose à l'aide de la commande suivante :

    docker-compose --version


### conteneurisation et execution du serveur nginx

copier le contenu du repertoire infra/nginx/data vers les repertoire corespondants sur le serveur

    scp -i ~/.ssh/id_rsa -r infra/nginx/docker-compose-nginx.yml ruth@216.225.202.241:~/docker-compose/docker-compose-nginx.yml

Connexion au serveur distant pour executer notre conteneur

    ssh -i ~/.ssh/id_rsa  ruth@216.225.202.241

    mkdir docker-compose
    cd docker-compose

Lancer le conteneur nginx

    docker-compose -f docker-compose-nginx.yml up -d

Consulter les conteneurs lancer à partir du fichier docker-compose

    docker-compose -f docker-compose-nginx.yml ps

## Etape 2
Fork du repos initial
    https://github.com/christus2024/mkutano-frontend.git

clone du repos git@github.com:Ruthtchassi/mkutano-frontend.git en local

recupérer une branche depuis son repos distant pour le repos local faire

     git fetch origin
     git branch -r
pour creer une branche featureruth/sprint1 qui as le contenue de la branche situé sur le repos github qui est feature/sprint1
     git checkout -b featureruth/sprint1 origin/feature/sprint1

Build de l'artefact .jar avec maven

    mvn clean package

Build de l'image docker

    docker build -t code-frontend:1.0.0 --build-arg VERSION=1.0.0-SNAPSHOT .

Tag du conteneur avec mon compte docker hub

    docker tag code-frontend:1.0.0 tchassiruth/code-frontend:1.0.0
    verification si sa marche
    - sudo docker run -d -p 8080:8080 tchassiruth/code-frontend:1.0.0
    - sudo docker ps
    - http://localhost:8080
    - sudo docker stop nom_ou_id_du_contenaire
## Etape 3

Creation du token docker hub pour la connexion au registry
Pour cela il faut se rendre sur le registry dockerhub et s'y connecter.
cliquer sur le profil et aller sur account setting ->> Personal access tokens ->> Generate new token
Reseigner le nom du token et les access en lecture/ ecriture sur le repo. Pour generer le token

Connexion au registry docker hub avec mon token generé

    docker login -u tchassiruth -p token

push de l'image sur le repository docker hub
creation du dockerhub tchassiruth/code-frontend
    docker push tchassiruth/code-frontend:1.0.0


## Etape 4 & 5

Push du docker-compose sur le serveur

    scp -i ~/.ssh/id_rsa -r infra/docker-compose-code-frontend.yml ruth@216.225.202.241:~/docker-compose/docker-compose-code-frontend.yml

connexion au serveur

    ssh  ruth@216.225.202.241

Execution du conteneur

    docker-compose -f docker-compose/docker-compose-code-frontend.yml up -d


Configuration du virtualhost (reverse proxy)
la configuration du virtualhost pour notre serveur se trouve dans le repertoire suivant:infra/nginx/data/conf.d/code-frontend.conf
nous alons le copier sur le serveur dans le repertoire de configuration de nginx ou le creer directement sur le serveur 

    scp -i ~/.ssh/id_rsa  infra/nginx/data/conf.d/code-frontend.conf ruth@216.225.202.241:~/nginx/conf.d/code-frontend.conf

en suite nous nous connectons sur le serveur et nous redemarrons nginx

    ssh  ruth@216.225.202.241
    docker stop nginx
    docker start nginx

## Etape 6

Tests utilisateurs

http://216.225.202.241:80 ou 8080
On obtien la page d'acceuille de l'aplication

NB:
Nginx au démarrage va créer des répertoires et générer sa configuration de base
Ces répertoires sont ceux définis dans le docker compose
En vu de rendre les données de ces répertoires persistant il a fallu qu'on les mape a des volumes
Les volumes au sens docker permettent de mapper un répertoire sur le serveur a un répertoire sur le conteneur
 Par contre on a deux types de mapping de volumes qui diffèrent selon l'usage
 Le mapping de type mount qui consiste à créer un répertoire sur le serveur et le mapper a un répertoire dans le conteneur:  -v /home/ruth/nginx:/var/nginx
Dans cette configuration les fichiers créer sur le serveur dans /home/ruth/nginx par un utilisateur sont lus par le conteneurs lorsqu'il lit son répertoire mappe /var/nginx
Et le conteneur par contre n'écris pas dans ce répertoire
 Et le mapping de type bind -v data:/var/nginx ou data est un volume docker
Dans cette configuration le conteneur en écrivant dans son répertoire écrit dans le volume
Et l'utilisateur système du serveur peut aussi écrire dans le volume
 C'est pour cette raison que tu as deux types de volumes dans le docker compose
On s'est arrangé a ce que le conteneur puis écrire sa configuration par défaut dans les volumes monté.
