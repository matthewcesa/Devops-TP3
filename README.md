# Devops-TP3


## 3-1 Documentez votre inventaire et les commandes de base

L'inventaire YAML `inventories/setup.yml` décrit les hôtes qu'Ansible peut administrer :

- `all` contient les variables communes à tous les hôtes.
- `ansible_user` définit l'utilisateur SSH (`admin`).
- `ansible_ssh_private_key_file` indique le chemin local de la clé privée SSH. Ce chemin doit être adapté à la machine qui lance Ansible; ne jamais ajouter la clé privée au dépôt.
- Le groupe `prod` contient l'hôte `matthew-frederick.cesa.takima.school`.

La commande suivante affiche l'inventaire interprété par Ansible :

```sh
ansible-inventory -i inventories/setup.yml --list
```

## Commandes de base
Depuis la racine du dépôt, tester la connexion SSH vers tous les hôtes de l'inventaire :

```sh
ansible all -i inventories/setup.yml -m ping
```

Le module `setup` collecte les faits de la machine distante. Le filtre ci-dessous affiche les variables dont le nom commence par `ansible_distribution` :

```sh
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

Le module `apt` veille à ce que le paquet Apache2 soit absent. `--become` demande les privilèges nécessaires à la gestion des paquets :

```sh
ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become
```

Ansible applique l'état demandé. Une première exécution peut supprimer Apache2; les exécutions suivantes ne le modifient plus et indiquent normalement `changed: false`.


## 3-2 Documentez votre plan de jeu

Un rôle regroupe les tâches d'une configuration dans une structure réutilisable. Le rôle Docker se trouve dans `ansible/roles/docker`; ses tâches principales sont définies dans `ansible/roles/docker/tasks/main.yml`.

Un rôle peut être initialisé avec Ansible Galaxy depuis la racine du dépôt :

```sh
ansible-galaxy init ansible/roles/docker
```
Les tâches d'installation des dépendances, du dépôt Docker, du paquet Docker, du SDK Python et du service Docker sont placées dans `tasks/main.yml`. Le playbook `ansible/playbook.yml` appelle le rôle avec `roles: - docker`, cible les hôtes de l'inventaire, collecte les faits et active l'élévation de privilèges. Aucun gestionnaire n'étant utilisé, le répertoire `handlers/` n'est pas nécessaire au fonctionnement actuel du rôle.

Pour appliquer le playbook et vérifier le rôle ainsi que l'installation sur les hôtes :

```sh
ansible-playbook -i ansible/inventories/setup.yml ansible/playbook.yml --ask-become-pass
```

`--ask-become-pass` demande le mot de passe sudo si l'utilisateur distant en a besoin. Une exécution réussie affiche `failed=0` pour chaque hôte; si Docker et ses dépendances sont déjà dans l'état demandé, une nouvelle exécution ne doit normalement signaler aucune modification.

## 3-3 Documentez la configuration de vos tâches docker_container

Les rôles `database`, `app` et `proxy` utilisent le module `community.docker.docker_container` pour déclarer les conteneurs. Chaque tâche fournit un nom de conteneur, une image Docker, `pull: true` pour récupérer l'image avant son lancement, et `restart_policy: always` pour redémarrer automatiquement le conteneur.
     
- **Base de données** (`ansible/roles/database/tasks/main.yml`) : le conteneur `{{ db_container }}` utilise l'image `eucko/tp-devops-database`. Les variables `POSTGRES_DB`, `POSTGRES_USER` et `POSTGRES_PASSWORD` sont alimentées par `db_name`, `db_user` et `db_password`. Le volume nommé `db-data` conserve les données PostgreSQL dans `/var/lib/postgresql/data`.
- **API** (`ansible/roles/app/tasks/main.yml`) : le conteneur `{{ api_container }}` utilise l'image `eucko/tp-devops-simple-api`. Ses variables d'environnement indiquent le nom de l'hôte de base de données, l'URL JDBC et les identifiants PostgreSQL, provenant de `db_container`, `db_name`, `db_user` et `db_password`.
- **Proxy HTTP** (`ansible/roles/proxy/tasks/main.yml`) : le conteneur `httpd` utilise l'image `eucko/tp-devops-http-server` et publie le port `80` de l'hôte sur le port `80` du conteneur avec `published_ports: "80:80"`.

Les trois conteneurs rejoignent le réseau Docker `app-network`, créé au préalable par le rôle `network`. Les variables de nom et de configuration de la base sont définies dans `ansible/playbook.yml`. Le premier play installe le SDK Docker pour Python dans `/opt/docker_venv`; le second définit `ansible_python_interpreter` vers cet environnement, nécessaire à l'exécution des modules de la collection `community.docker`.

## 3-4 Sécurité du déploiement automatique des images

Déployer automatiquement une image n'est pas sûr par défaut. Le workflow `.github/workflows/main.yaml` lance le playbook à chaque push sur `main`, mais ne construit ni ne publie d'image sur Docker Hub. Les rôles Ansible tirent les images indiquées dans leurs tâches avec `pull: true`. Comme aucune balise n'est précisée, Docker utilise implicitement `latest` : cette balise est modifiable et peut désigner un contenu différent d'une exécution à l'autre. Une image vulnérable, compromise ou simplement non testée pourrait donc être déployée automatiquement.

Pour renforcer la sécurité :

- Construire et tester les images en CI, puis effectuer une analyse de vulnérabilités avant toute publication ou mise en production.
- Publier depuis un pipeline contrôlé, uniquement après validation du code et sur une branche ou une version protégée; ne pas publier depuis les contributions externes non approuvées.
- Remplacer `latest` par une balise de version immuable et, pour garantir exactement le même artefact, épingler l'image par son digest (`image@sha256:...`). Promouvoir en production l'image déjà testée plutôt que de reconstruire une autre image.
- Protéger la branche `main` avec des revues et des vérifications CI obligatoires; demander une approbation d'environnement avant un déploiement en production.
- Stocker les clés et jetons dans les secrets GitHub, leur donner le minimum de droits et les faire tourner régulièrement. Utiliser une clé SSH dédiée au déploiement avec un compte distant aux privilèges limités.
- Épingler les actions GitHub à un SHA de commit vérifié et réactiver la vérification des clés d'hôte SSH (`ANSIBLE_HOST_KEY_CHECKING`), actuellement désactivée dans le workflow, en configurant à l'avance les clés d'hôte attendues.