# Mise à jour Mahara

| Composante | Lien |
|------------|----|
| Bitbucket | https://bitbucket.org/account/user/uqam/projects/MAH |
| Jenkins | https://jenkins.dev.uqam.ca/view/ENA/job/ENA/job/Mahara/ |
| Wiki | https://wiki.uqam.ca/x/5dQEB |

Passer mahara à k8s: https://wiki.uqam.ca/x/Hv7bAg

On y trouve les dépots et des liens de doc. 

Nous allons faire les ménage des dépôts de suivi mahara. On cherche à avoir une synch automatique pour les branches d'origine.

Ne pas oublier d'ajouter truc pour que le lien pour log admin fonctionne.

Valider. Ils semblent avoir ajouté un paramètre `override`; on va à
```
https://uqam.ngrok.io/?login&override=true
```

On doit corriger l'image. Problème de session après la mise à jour:
```
[Tue Nov 12 09:15:05.192881 2019] [php7:notice] [pid 67] [client 127.0.0.1:40964] [WAR] ed (lib/errors.php:536) [Error]: Failed to create 
session ID: memcached (path: 3;/var/www/maharadata/sessions), 
referer: https://uqam.ngrok.io/?login&override=true
[Tue Nov 12 09:15:05.192907 2019] [php7:notice] [pid 67] [client 127.0.0.1:40964] Call stack (most recent first):, 
referer: https://uqam.ngrok.io/?login&override=true
[Tue Nov 12 09:15:05.192913 2019] [php7:notice] [pid 67] [client 127.0.0.1:40964]   * exception(object(Error)) 
at Unknown:0, referer: https://uqam.ngrok.io/?login&override=true
[Tue Nov 12 09:15:05.192918 2019] [php7:notice] [pid 67] [client 127.0.0.1:40964] , 
referer: https://uqam.ngrok.io/?login&override=true
[Tue Nov 12 09:15:05.194417 2019] [php7:error] [pid 67] [client 127.0.0.1:40964] PHP Fatal error:  Uncaught Error: 
Failed to create session ID: memcached (path: 3;/var/www/maharadata/sessions) 
in /var/www/html/auth/session.php:416\nStack trace:\n#0 /var/www/html/auth/session.php(416): 
session_start()\n
#1 /var/www/html/auth/session.php(333): Session->ensure_session()\n
#2 /var/www/html/lib/pieforms/pieform.php(602): Session->add_error_msg('There was an er...')\n
#3 /var/www/html/lib/pieforms/pieform.php(168): Pieform->__construct(Array)\n
#4 /var/www/html/lib/mahara.php(5432): Pieform::process(Array)\n
#5 /var/www/html/auth/lib.php(2493): pieform(Array)\n
#6 /var/www/html/lib/web.php(175): auth_generate_login_form()\n
#7 /var/www/html/lib/errors.php(682): smarty(Array, Array, Array, Array)\n
#8 /var/www/html/lib/errors.php(547): MaharaException->handle_exception()\n
#9 [internal function]: exception(Object(SystemException))\n
#10 {main}\n  thrown in /var/www/html/auth/session.php on line 416, 
referer: https://uqam.ngrok.io/?login&override=true
```

Les paramètres de configuration ont changé; ajouter dans `config.php`:
``
$cfg->memcacheservers = getenv('MAH_MCROUTER');
$cfg->sessionhandler = 'memcached';`
```

### N'est plus nécessaire
Pour que ça fonctionne la passe:
```
https://mahara.dev.uqam.ca/?login&admin_login
```
Dans `auth/lib.php` ligne `1123`
```
if ($externallogin) {
        $get = $_GET;
        $externallogin = preg_replace('/{shorturlencoded}/', urlencode(get_relative_script_path()), $externallogin);
        $externallogin = preg_replace('/{wwwroot}/', get_config('wwwroot'), $externallogin);
        if (isset($get['login']) && isset($get['admin_login'])){
            ;
        }
        else {
            redirect($externallogin);
        }
    }
```

et nous avons ajouté dans le fichier de config:
```
$cfg->externallogin = 'https://mahara.dev.uqam.ca/auth/saml/\
index.php?hostwwwroot={wwwroot}&wantsurl={shorturlencoded}';

```


Quand on part la machine, on reçoit:
```
Mahara: Site Unavailable
Must upgrade to 2017031605 (17.04.0 (release tag 17.04.0_RELEASE)) first \
(you have 2016090212 (16.10.2testing)
```

En allant par `admin/cli/upgrade.php`:
```
root@mah01-5b98f88f7c-lbxjv:/var/www/html/admin/cli# php upgrade.php 
[11-Nov-2019 21:30:47 UTC] PHP Warning:  PHP Startup: Unable to load dynamic library 'memcache.so' (tried: /usr/lo
cal/lib/php/extensions/no-debug-non-zts-20180731/memcache.so (/usr/local/lib/php/extensions/no-debug-non-zts-20180
731/memcache.so: cannot open shared object file: No such file or directory), /usr/local/lib/php/extensions/no-debu
g-non-zts-20180731/memcache.so.so (/usr/local/lib/php/extensions/no-debug-non-zts-20180731/memcache.so.so: cannot 
open shared object file: No such file or directory)) in Unknown on line 0
[11-Nov-2019 16:30:47 America/Toronto] PHP Deprecated:  PHP Startup: memcached.sess_lock_wait and memcached.sess_l
ock_max_wait are deprecated. Please update your configuration to use memcached.sess_lock_wait_min, memcached.sess_
lock_wait_max and memcached.sess_lock_retries in Unknown on line 0
[11-Nov-2019 16:30:47 America/Toronto] PHP Deprecated:  PHP Startup: memcached.sess_lock_wait and memcached.sess_l
ock_max_wait are deprecated. Please update your configuration to use memcached.sess_lock_wait_min, memcached.sess_
lock_wait_max and memcached.sess_lock_retries in Unknown on line 0
[11-Nov-2019 16:30:47 America/Atikokan] [WAR] 92 (lib/upgrade.php:87) Must upgrade to 2017031605 (17.04.0 (release
 tag 17.04.0_RELEASE)) first  (you have 2016090212 (16.10.2testing)
[11-Nov-2019 16:30:47 America/Atikokan] Call stack (most recent first):
[11-Nov-2019 16:30:47 America/Atikokan]   * check_upgrades() at /var/www/html/admin/cli/upgrade.php:46
[11-Nov-2019 16:30:47 America/Atikokan] 
Must upgrade to 2017031605 (17.04.0 (release tag 17.04.0_RELEASE)) first  (you have 2016090212 (16.10.2testing)
```

```
git clone https://github.com/MaharaProject/mahara.git gh-mahara
git checkout origin/17.04_STABLE -b 17.04
Branch '17.04' set up to track remote branch '17.04_STABLE' from 'origin'.
Switched to a new branch '17.04'
nmoller@nm-UQAM-HP:~/dev/uqam/infra/gh-mahara (17.04)$ cp ../docker-mdluqam/mahara/config.php htdocs/

```
J'avais oublié que les css ne sont pas prêts.

Problème lors de la migration vers 17.04:
```
A column of your database is using a collation that is not the same as the 
database default. Please ensure all columns use the same collation as the 
database
```
https://mahara.org/interaction/forum/topic.php?id=7318

Le problème a été reglé avec
```
ALTER DATABASE mahara_dev CHARACTER SET = utf8 COLLATE = utf8_general_ci;
```
ALTER DATABASE mahara_dev CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
Voir: 

https://mathiasbynens.be/notes/mysql-utf8mb4#utf8-to-utf8mb4

Changer tout à utf8mb4 donne:
```
ERROR 1071 (42000) at line 471282: Specified key was too long; max key length is 767 bytes
```

https://stackoverflow.com/questions/1814532/1071-specified-key-was-too-long-max-key-length-is-767-bytes

## Création du container
Le lien trouvé dans 

https://wiki.uqam.ca/x/Hv7bAg

Voir (pour aller dans la branche contenant ce qui correspond à mahara): 

https://bitbucket.org/uqam/docker-mdluqam/src/MAH_WEB_UID48_PHP56NR/



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

Pour utiliser phpmyadmin
```
kubectl -n siad-mahara-dev-01 port-forward pod/phpmyadmin 8888:80
```
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
```
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
```
Faire le setu-up dans la bd pour ne pas avoir à modifier les vrais données:


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

# Installer dépendencesinstall
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

## Outils pour thème.

Préparer un thème basé sur modern avec des coleurs plus uqam; utiliser un outil comme:

https://material.io/resources/color/#!/?view.left=0&view.right=0&primary.color=2962FF

## Image

https://talk.plesk.com/threads/install-memcache-php7-error.340289/

https://stackoverflow.com/questions/41605999/how-can-i-find-php-smart-string-h-instead-of-php-smart-str-h
# Ce qui reste à faire

- Nouvelle image mahara
- Configuration ssphp 

## Mcrouter

```
jphalip/mcrouter:0.36.0
```

Partir un container ubuntu:18.04, suivre les instructions du `Readme` du release de mcrouter

Même sans avoir netoyé;
Bâtir une image plus lègère:
```
docker ps -l -q
8577ce715a2a
nmoller@nm-UQAM-HP:~$ docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                                  NAMES
8577ce715a2a        ubuntu:18.04           "bash"                   6 minutes ago       Up 6 minutes                                               boring_shtern
nmoller@nm-UQAM-HP:~$ docker commit -m "mcrouter installed" -a "Nelson Moller" 8577ce715a2a nmolleruq/mcrouter:0.41.0
sha256:c599ab98c7c3f8eccb204d972d970d2780d2db45eb2e785ef891bf012a77c10a
nmoller@nm-UQAM-HP:~$ docker images |grep mcrouter
nmolleruq/mcrouter                                    0.41.0              c599ab98c7c3        12 seconds ago      165MB
```