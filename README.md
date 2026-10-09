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