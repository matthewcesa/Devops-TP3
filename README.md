# Devops-TP3

## Inventaire Ansible

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