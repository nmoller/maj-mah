# Mise à jour Mahara

| Composante | Lien |
|------------|----|
| Bitbucket | https://bitbucket.org/account/user/uqam/projects/MAH |
| Jenkins | https://jenkins.dev.uqam.ca/view/ENA/job/ENA/job/Mahara/ |
| Wiki | https://wiki.uqam.ca/x/5dQEB |

Passer mahara à k8s: https://wiki.uqam.ca/x/Hv7bAg

On y trouve les dépots et des liens de doc. 

Nous allons faire les ménage des dépôts de suivi mahara. On cherche à avoir une synch automatique pour les branches d'origine.

## Piège à éviter
Le code source de base de mahara ne contient pas les dossier theme/styles. On doit commencer par les créer. 

## Tests locaux

Nous allons utiliser kind 

- https://kind.sigs.k8s.io/docs/user/quick-start/
- https://itnext.io/starting-local-kubernetes-using-kind-and-docker-c6089acfc1c0

Si c'est la première utilisation; pas de cluster local, nous allons le créer:
```
kind create cluster --name k8s-test
Creating cluster "k8s-local" ...
 ✓ Ensuring node image (kindest/node:v1.14.2) 🖼
 ✓ Preparing nodes 📦 
 ✓ Creating kubeadm config 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Cluster creation complete. You can now use the cluster with:

export KUBECONFIG="$(kind get kubeconfig-path --name="k8s-test")"
kubectl cluster-info
```
Une fois le cluster créé pour s'en servir nous allons utiliser 
```
export KUBECONFIG="$(kind get kubeconfig-path --name="k8s-test")"
kubectl cluster-info
```

Explorer le setting. On a besoin de créer les pvc :
```
# créer le ns
kubectl create ns siad-mahara-dev-01
namespace/siad-mahara-dev-01 created

# explorer stockage
kubectl get storageclass

NAME                 PROVISIONER               AGE
standard (default)   kubernetes.io/host-path   37m
```
Nous pouvons créer directement la `pvc` et le provisioner va faire le bounding sans problèmes:
```
kubectl create -f kind/pvc.yaml 
persistentvolumeclaim/pvc-siad-mahara-dev-01-vol01 created

kubectl -n siad-mahara-dev-01 get pvc
NAME                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-siad-mahara-dev-01-vol01   Bound    pvc-b6d1e230-ef4a-11e9-8350-0242ac110002   1Gi        RWX            standard       46s

```

Script `create-mahara-inf`: on y mettra tout le processus pour que ce soit facile à reproduire.

### Set-up de la BD
La taille n'étant pas grosse, nous pouvons aller avec un dump par l'interface phpmyadmin.

Une fois que nous avons le fichier
```
kubectl -n siad-mahara-dev-01 cp \
DB/mahara_dev.sql \
siad-mahara-dev-01/mysql-cc68c9945-d9rtb:/mahara_dev.sql

# lancer des commandes dans le pod
kubectl -n siad-mahara-dev-01 exec -it mysql-cc68c9945-d9rtb \
 -- mysql -uroot -p -e "create database mahara_dev;"

kubectl -n siad-mahara-dev-01 exec -it mysql-cc68c9945-d9rtb  -- bash

mysql -uroot -p mahara_dev < ./mahara_dev.sql 
Enter password: 
ERROR 1064 (42000) at line 316282: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '"usr" SET unread = unread - 1 WHERE id = OLD.usr;
                END IF;
      ' at line 4

```

Problème avec les triggers
```(sql)
--
-- Déclencheurs `module_multirecipient_userrelation`
--
DELIMITER $$
CREATE TRIGGER `update_unread_delete2_trigger` 
AFTER DELETE ON `module_multirecipient_userrelation` 
FOR EACH ROW BEGIN
                    
    IF OLD.read = '0' AND OLD.role = 'recipient' THEN
        UPDATE "usr" SET unread = unread - 1 WHERE id = OLD.usr;
    END IF;
    END
$$
DELIMITER ;
DELIMITER $$
CREATE TRIGGER `update_unread_insert2_trigger` 
AFTER INSERT ON `module_multirecipient_userrelation` 
FOR EACH ROW BEGIN
                    
    IF NEW.role = 'recipient' AND NEW.read = '0' THEN
        UPDATE "usr" SET unread = unread + 1 WHERE id = NEW.usr;
    END IF;
    END
$$
DELIMITER ;
DELIMITER $$
CREATE TRIGGER `update_unread_update2_trigger` 
AFTER UPDATE ON `module_multirecipient_userrelation` 
FOR EACH ROW BEGIN
                    
    IF OLD.read = '0' AND NEW.read = '1' AND NEW.role = 'recipient' THEN
        UPDATE "usr" SET unread = unread - 1 WHERE id = NEW.usr;
    ELSEIF OLD.read = '1' AND NEW.read = '0' AND NEW.role = 'recipient' THEN
        UPDATE "usr" SET unread = unread + 1 WHERE id = NEW.usr;
    END IF;
    END
$$
DELIMITER ;
```

Nous allons recommencer sans les triggers:
```
rm -rf mahara_dev.sql
mysql -uroot -p -e "drop database mahara_dev;"

```
Il y'en a d'autres:
```
--
-- Déclencheurs `notification_internal_activity`
--
DELIMITER $$
CREATE TRIGGER `update_unread_delete_trigger` AFTER DELETE ON `notification_internal_activity` FOR EACH ROW BEGIN
                    
                IF OLD.read = 0 THEN
                    UPDATE "usr" SET unread = unread - 1 WHERE id = OLD.usr;
                END IF;
                END
$$
DELIMITER ;
DELIMITER $$
CREATE TRIGGER `update_unread_insert_trigger` AFTER INSERT ON `notification_internal_activity` FOR EACH ROW BEGIN
                    
                IF NEW.read = 0 THEN
                    UPDATE "usr" SET unread = unread + 1 WHERE id = NEW.usr;
                END IF;
                END
$$
DELIMITER ;
DELIMITER $$
CREATE TRIGGER `update_unread_update_trigger` AFTER UPDATE ON `notification_internal_activity` FOR EACH ROW BEGIN
                    
                IF OLD.read = 0 AND NEW.read = 1 THEN
                    UPDATE "usr" SET unread = unread - 1 WHERE id = NEW.usr;
                ELSEIF OLD.read = 1 AND NEW.read = 0 THEN
                    UPDATE "usr" SET unread = unread + 1 WHERE id = NEW.usr;
                END IF;
                END
$$
DELIMITER ;

--
-- Déclencheurs `usr`
--
DELIMITER $$
CREATE TRIGGER `unmark_quota_exceed_upd_usr_set_trigger` AFTER UPDATE ON `usr` FOR EACH ROW BEGIN
                    
                UPDATE "usr_account_preference", "artefact_config"
                SET "usr_account_preference".value = 0
                WHERE "usr_account_preference".field = 'quota_exceeded_notified'
                AND "usr_account_preference".usr = NEW.id
                AND "artefact_config".plugin = 'file'
                AND "artefact_config".field = 'quotanotifylimit'
                AND NEW.quotaused/NEW.quota < "artefact_config".value/100;
                END
$$
DELIMITER ;
```
Faire le setu-up dans la bd pour ne pas avoir à modifier les vrais données:
```
```

## Script `debase64.sh`
Tout simple pour ne pas avoir à taper à chaque fois:
```
#!/usr/bin/env bash

echo "$1" |base64 -d
```

Pour ne pas avoir des sauts de ligne dans le contenu encodé:
```
echo -n 'mahara_dev' |base64 -w 0
```

## Extras création du dashboard dans kind

Télécharger le fichier localment à partir de

https://github.com/kubernetes/dashboard 

avec le lien pour le déployer.

Suivre les instructions de 

https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md

en modifiant le namespace à `kube-system` parce que c'est là que le dashboard a été créé.

```
kubectl -n kube-system describe \
secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

## Valider qu'on est correct avec l'ancienne version
```
https://uqam.ngrok.io/?login&admin_login
```

## Thème

Le code source de mahara ne contient pas le dossier style (dans chaque thème, il y a un fichier `.gitignore` pour éviter que ce soit inclus). 
Effacer tous les fichirs `.gitignore` des thèmes:
```
rm -f htdocs/theme/*/.gitignore
```
La première chose à faire pour avoir une nouvelle version est de 
```
# Partir un containeur dormant nommé pour y lancer les commandes
docker run -it --name mahara-theme \
-v $(pwd):/opt/mahara \
-w /opt/mahara -u 1000:1000 \
node:10-jessie sleep 3h

# Installer dépendences
docker exec -it mahara-theme npm install

# Installer gulp globalement dans le containeur
docker exec -it -u 0 mahara-theme npm install -g gulp

# compiler les css
docker exec -it mahara-theme make css

# Installer simplesamlphp
# Pour faker le système composer est dans le PATH du containeur php
ln -s /usr/bin/composer external/composer.phar

docker run -it --rm -v $(pwd):/opt/mahara -w /opt/mahara \
-u 1000:1000 nmolleruq/phpcomposer:7.2 make ssphp

# En cas de problèmes, pour recommencer:
docker run -it --rm -v $(pwd):/opt/mahara -w /opt/mahara \
-u 1000:1000 nmolleruq/phpcomposer:7.2 make cleanssphp
```

# Ce qui reste à faire

- Nouvelle image mahara
- Configuration ssphp 

