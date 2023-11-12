# TP Loadbalancing

## Introduction

### L'objectif

L’objectif de ce TP est d’étudier et de mettre en place l’application web Nextcloud (https://nextcloud.com/) de manière à la rendre hautement disponible. Par hautement disponible, vous devez bien évidemment entendre tolérance de panne et répartition de charge.

En effet, l’application doit tolérer la perte de n’importe quel équipement matériel sans engendrer de coupure de service.

De plus l’application doit pouvoir s’adapter, sans changement d’architecture majeure, à une augmentation de la charge (nombre de connexions clientes importantes). Votre travail devra faire apparaître tous les éléments matériels nécessaires ainsi que les composants logiciels à mettre en place. Toutes les solutions techniques sont envisageables à partir du moment où elles répondent aux contraintes de haute disponibilité. 

Afin de rendre un travail complet, vous vous attacherez à prendre en compte tous les composants importants pour cette application :
• Frontal http / https
• Gestion des sessions de connexion
• Gestion du stockage de fichiers
• Gestion de la base de données

### L'infrastructure

Tout d'abord notre infrastucture sera dans le réseau 10.0.2.0/24. Nous aurons un client avec l'IP 10.0.2.100/24, qui accèdera au LOAD Balancer HaPROXY qui aura pour role répartir le trafic entrant sur deux serveurs HTTP hébergeant NextCloud, garantissant ainsi une distribution équilibrée de la charge, une meilleure disponibilité, et une haute fiabilité des applications. 

Ce HA Proxy repartira la charge entre deux serveurs Apache en 10.0.2.21/24 et 10.0.2.22/24. Nous installerons sur ces deux serveurs le service Nextcloud qui est une plateforme de collaboration en ligne open source permettant de stocker, partager et accéder à des fichiers, ainsi que d'intégrer des applications telles que le calendrier et la messagerie. Pour faire fonctionné ce service nous avons besoin d'une base de donnée MariaDB SQL ainsi qu'un espace de stockage. 

Pour la base de donnée nous allons mettre en place un Cluster Innodb avec un router MySQL (MYSQLRT - 10.0.2.30/24) avec deux serveurs BDD (MariaDB) en 10.0.2.31/24 et 10.0.2.32/24. 

Pour la partie stockage nous allons mettre en place un stockage GlusterFS Actif/Passif qui est une configuration où deux nœuds de stockage fonctionnent en mode actif et passif, assurant une haute disponibilité et la reprise sur incident en cas de défaillance d'un nœud, avec deux serveurs en 10.0.2.41/24 et 10.0.2.42/24.

![image](Images/schema.png)

## Mise en place

### Configuration MYSQL

Tout d'abord nous configurons notre interface réseau en IP fixe

```bash
# nano /etc/netplan/00-installer-config.yaml

network:
  ethernets:
    enp0s3:
      addresses:
        - 10.0.2.31/24
      routes:
        - to: default
          via: 10.0.2.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
      dhcp4: false
  version: 2
```

Nous changeons ensuite notre fichier host afin que nos serveurs communiquent plus simplement entre eux :

![image](Images/Image1-host.png)

Nous pouvons enfin commencer à installer MySQL :

```bash
apt install mysql-server 
snap install mysql-shell 
# Remplacer bind-address = 127.0.0.1 par bind-address = 0.0.0.0 dans le fichier /etc/mysql/mysql.conf.d/mysqld.cnf

systemctl restart mysql.service
```

Nous mettons en place le cluster en faisant les commandes suivante sur les deux noeuds pour créer un admin remote sur chaque node :

```bash
mysql -p
CREATE USER 'admin'@'%' IDENTIFIED BY 'admin'; 
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' WITH GRANT OPTION; 
FLUSH PRIVILEGES;
```

Depuis MYSQL1 :

```bash
mysqlsh 
dba.configure_instance('admin@MYSQL1') 
dba.configure_instance('admin@MYSQL2') 

\c admin@MYSQL1
cluster = dba.create_cluster('cluster1') 
cluster.status()

// Ajout des nodes en mode clone 
cluster.add_instance('admin@MYSQL2')  
```

![image](Images/Image2.png)

Sur le MYSQLRT :

```bash
wget https://dev.mysql.com/get/mysql-apt-config_0.8.25-1_all.deb 
dpkg -i mysql-apt-config_0.8.25-1_all.deb 
apt update 
apt install mysql-router mysql-shell mysql-client 

mysqlrouter --bootstrap admin@MYSQL1 --user=root --directory  /etc/innodbcluster1
```
![image](Images/Image3.png)

Transformation en Multiprimary :
```bash
mysqlsh
\c admin@M1
cluster = dba.get_cluster()
cluster.switch_to_multi_primary_mode()
```

Pour terminer la partie MYSQL, voici une vidéo montrant le role du Routeur MySQL ainsi que lolérance à la panne :

![video](Videos/MYSQL_failback.gif)

### Mise en place d'un pool de stockage (GlsuterFS)

Pour mettre en place un pool de stockage redondant, nous allons choisir la technologie GlusterFS qui est un système de fichiers distribué open source conçu pour fournir une gestion transparente du stockage sur plusieurs serveurs. Il permet d'agrégation de l'espace disque et offre une haute disponibilité et une extensibilité.

Nous allons donc utiliser deux serveurs, NFS1 (10.0.2.41/24) et NFS (10.0.2.42/24). Nous configurons donc dans un premiers temps les IP et les fichiers hosts sur les machines :

![image](Images/Image4.png)

Nous installons ensuite GlusterFS sur les deux serveurs :

```bash
sudo apt install glusterfs-server
sudo systemctl start glusterd.service
sudo systemctl enableglusterd.service
```

Nous activons le pare-feu et nous n'autorisons que les flux venant de notre réseau :

```bash
sudo ufw allow from 10.0.2.0/24 to any port 24007
```

Nous allons maintenant établir le connexion entre les deux noeuds grace à la commande suivante :

```bash
 gluster peer probe NFS2
```
![image](Images/Image5.png)

Une fois que les deux noeuds sont liés, nous allons créer pool de stockage sur NFS1 :

```bash
sudo gluster volume create volume1 replica 2 NFS1:/storage-gluster NFS2:/storage-gluster force
sudo gluster volume start volume1
```

![image](Images/Image6.png)

Nous pouvons voir que le cluster est bien créé:

![image](Images/Image7.png)

Nous devons maintenant ouvrir les ports suivants sur notre réseau afin que les clients puissent s'y connecté :

```bash
sudo ufw allow from 10.0.2.0/24 to any port 55978
sudo ufw allow from 10.0.2.0/24 to any port 56308
```

Connectons nous maintenant sur le client pour y connecter le pool de stockage, nous devons dans un premier temps installer glusterfs client et créer un dossier sur le serveur afin d'y monter la partition :

```bash
apt install glusterfs-client
mkdir /storage-client
mount -t glusterfs NFS1:/volume1 /storage-client
```

![image](Images/Image8.png)

Nous allons maintenant voir que les fichiers sont bien répliqué entre les deux serveurs :
![video](Videos/glusterFS.gif)

Nous pouvons aussi voir que le pool de sockage est tolérant à la panne :
![video](Videos/glusterfs_failback.gif)

### Mise en place de NextCloud

Nous allons mettre en place Nextcloud est une plateforme de collaboration en ligne open source qui offre des services de stockage de fichiers, de partage et de collaboration, le tout hébergé sur votre propre infrastructure. Il permet aux utilisateurs de synchroniser et de partager des fichiers, ainsi que de collaborer sur des documents en temps réel.

Pour cela nous allons mettre en place deux serveurs apache2, HTTP1 (10.0.2.21/24) et HTTP2 (10.0.2.22/24).

Pour commencer nous allons créer completement le serveur HTTP1 puis nous allons ensuite le cloner afin d'avoir la même configuration sur le HTTP2.
Pour cela nous allons configurer le fichier /etc/hosts que l'on peut copié sur tous nos serveurs :

![image](Images/Image9.png)

Nous installaons ensuite apache2 sur le serveurs : 

```bash
apt install apache2
systemctl enable apache2
systemctl start apache2
```

Une fois installer, nous pouvons voir la page par défaut d'apache sur le serveur :

![image](Images/Image10.png)

Pour le besoin de NextCloud nous devons activer plusieurs module d'apache2 :

```bash
Nous devons ensuite ajouter plusieurs module :
sudo a2enmod headers
sudo a2enmod env
sudo a2enmod dir
sudo a2enmod mime
sudo a2enmod ssl 
sudo a2ensite default-ssl

systemctl restart apache2
```




