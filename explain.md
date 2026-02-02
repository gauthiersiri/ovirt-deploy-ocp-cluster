# Explication du Projet: ovirt-deploy-ocp-cluster

## Vue d'ensemble

Ce projet est un ensemble de playbooks Ansible conçu pour **déployer automatiquement des clusters OpenShift sur OpenShift Virtualization** en utilisant la méthode d'installation basée sur agent (agent-based installation).

## Architecture du Projet

### Structure des Répertoires

```
ovirt-deploy-ocp-cluster/
├── deploy.yaml              # Playbook principal pour déployer un nouveau cluster
├── add_node.yaml            # Playbook pour ajouter des nœuds à un cluster existant
├── ansible.cfg              # Configuration Ansible
├── inventory/               # Inventaire des clusters
├── host_vars/               # Variables spécifiques par cluster
├── roles/                   # Rôles Ansible pour chaque phase
└── clusters_data/           # Données générées pour chaque cluster
```

## Fonctionnalités Principales

### 1. Déploiement d'un Nouveau Cluster (`deploy.yaml`)

Le playbook principal orchestre le déploiement complet d'un cluster OpenShift en plusieurs phases:

#### Phase 1: Prérequis (`prereqs`)
- **Validation de la configuration** : Vérifie que tous les paramètres requis sont présents
- **Configuration des répertoires** : Crée la structure de répertoires nécessaire
- **Configuration SSH** : Génère et configure les clés SSH pour l'accès aux nœuds
- **Installation des packages** : Installe les dépendances Python et Ansible nécessaires
- **Téléchargement des binaires OpenShift** : Récupère `openshift-install` et `oc` depuis les miroirs officiels
- **Connexion à oVirt/OpenShift Virtualization** : Établit la connexion au cluster de virtualisation

#### Phase 2: Vérifications Préalables (`precheck`)
- Vérifie que les VMs n'existent pas déjà
- Valide la disponibilité des ressources

#### Phase 3: Déploiement des VMs (`deploy-vms`)
- **Création du namespace Kubernetes** pour le projet
- **Création d'un PVC (PersistentVolumeClaim)** pour l'ISO d'installation
- **Création de PVCs supplémentaires** pour ODF (OpenShift Data Foundation) si configuré
- **Déploiement des VMs** via l'API Kubernetes:
  - VMs Master (control plane)
  - VMs Worker (compute)
  - VMs Infra (infrastructure)

#### Phase 4: Installation du Cluster (`install-cluster`)

##### Génération de la Configuration
1. **Récupération des adresses MAC** des VMs créées via `oc get vm`
2. **Génération de `agent-config.yaml`** :
   - Configuration réseau statique pour chaque nœud
   - Mapping MAC address → IP address
   - Configuration DNS et gateway
   - Définition des rôles (master/worker)
3. **Génération de `install-config.yaml`** :
   - Configuration du cluster (nom, domaine de base)
   - Configuration réseau (CIDR, service network)
   - Pull secret et clé SSH
   - Configuration proxy si nécessaire
   - Manifests supplémentaires (ex: configuration des disques non-rotationnels)

##### Génération de l'ISO
- Utilise `openshift-install agent create image` pour créer l'ISO bootable
- L'ISO contient toute la configuration réseau et les ignition configs

##### Démarrage des Nœuds
- Upload de l'ISO dans le PVC Kubernetes
- Démarrage des VMs qui bootent sur l'ISO
- Les nœuds s'auto-configurent et rejoignent le cluster

##### Finalisation de l'Installation
- Surveillance de la progression de l'installation
- Attente de la disponibilité de l'API
- Validation du cluster

#### Phase 5: Installation ODF (Optionnelle)
Si 3 nœuds ou plus ont la propriété `odf_disk_size` définie:
- Installation de l'opérateur Local Storage
- Configuration des LocalVolumes
- Installation de l'opérateur ODF
- Création du StorageCluster

#### Phase 6: Sauvegarde de l'Inventaire
- Génère un fichier d'inventaire avec les informations du cluster déployé

### 2. Ajout de Nœuds (`add_node.yaml`)

Ce playbook permet d'ajouter des nœuds worker ou infra à un cluster existant:

#### Processus d'Ajout de Nœuds

1. **Identification des nœuds à ajouter** (`gather-nodes-to-add`)
   - Compare la configuration actuelle avec les VMs existantes
   - Identifie les nouveaux nœuds à créer

2. **Vérifications préalables**
   - Valide que les nœuds n'existent pas déjà

3. **Authentification au cluster**
   - Utilise le kubeconfig existant si disponible
   - Sinon, utilise un token d'authentification fourni

4. **Déploiement des nouvelles VMs**
   - Crée les VMs pour les nouveaux nœuds

5. **Configuration et ajout au cluster** (`add-nodes-to-cluster`)
   - Génère une configuration spécifique pour les nouveaux nœuds
   - Crée un nouvel ISO avec `openshift-install agent create image`
   - Utilise les scripts `node-joiner.sh` et `node-joiner-monitor.sh` d'OpenShift
   - Boot des nouvelles VMs sur l'ISO
   - Les nœuds rejoignent automatiquement le cluster

## Configuration

### Fichier de Variables (`host_vars/host.sample`)

Chaque cluster nécessite un fichier de variables contenant:

#### Configuration du Cluster de Virtualisation
```yaml
ovirt:
  hostname: "https://api.ocpvirt.cluster.demo:6443"
  token: "sha256~..."  # ou username/password
  bridge: "onn359"     # Réseau bridge
  storage_class: "svc-sc"
```

#### Configuration du Cluster OpenShift
```yaml
config:
  base_domain: cluster.demo
  cluster_name: "mycluster"
  cluster_version: "stable-4.17"
  pull_secret: "{{ lookup('file', '~/pullsecret') }}"
  master_schedulable: false
  root_device: "/dev/vda"
```

#### Définition des Nœuds
```yaml
nodes:
  master:
    - { ip: 10.3.59.141, cpu: 8, ram_gb: 24, disk_size_gb: 120 }
    - { ip: 10.3.59.142, cpu: 8, ram_gb: 24, disk_size_gb: 120 }
    - { ip: 10.3.59.143, cpu: 8, ram_gb: 24, disk_size_gb: 120 }
  worker:
    - { ip: 10.3.59.144, cpu: 8, ram_gb: 16, disk_size_gb: 300 }
  infra:
    - { ip: 10.3.59.146, cpu: 16, ram_gb: 34, disk_size_gb: 120, odf_disk_size: 200 }
```

#### Configuration Réseau
```yaml
static_ip:
  gateway: 10.3.59.254
  netmask: 255.255.255.0
  dns: 10.3.59.140
  network_interface_name: eno1

network_modifications:
  enabled: true
  clusterNetwork:
    - cidr: 100.68.0.0/14
      hostPrefix: 23
  serviceNetwork:
    - cidr: 100.66.0.0/15
  machineNetwork:
    - cidr: 10.3.59.0/24
```

## Méthode d'Installation: Agent-Based

Ce projet utilise la **méthode d'installation agent-based** d'OpenShift:

### Avantages
- **Installation sans infrastructure externe** : Pas besoin de serveur DHCP/PXE
- **Configuration réseau statique** : Idéal pour les environnements d'entreprise
- **Pré-configuration complète** : Toute la configuration est dans l'ISO
- **Installation automatisée** : Les nœuds s'auto-configurent au boot

### Fonctionnement
1. Un ISO bootable est généré avec `openshift-install agent create image`
2. L'ISO contient:
   - L'agent d'installation OpenShift
   - Les configurations réseau (nmstate)
   - Les ignition configs pour chaque nœud
3. Les VMs bootent sur cet ISO
4. L'agent détecte le nœud via son adresse MAC
5. Configure le réseau selon la configuration nmstate
6. Applique l'ignition config correspondante
7. Le nœud rejoint automatiquement le cluster

## Rôles Ansible Détaillés

### `prereqs`
Prépare l'environnement d'exécution du playbook.

### `precheck`
Vérifie que les VMs n'existent pas déjà pour éviter les conflits.

### `deploy-vms`
Crée les ressources Kubernetes (VMs, PVCs) via l'API Kubernetes du cluster de virtualisation.

### `install-cluster`
Orchestre l'installation complète du cluster OpenShift.

### `add-nodes-to-cluster`
Gère l'ajout de nouveaux nœuds à un cluster existant.

### `odf`
Installe et configure OpenShift Data Foundation pour le stockage distribué.

### `save-inventory`
Génère un fichier d'inventaire Ansible avec les informations du cluster déployé.

## Prérequis Système

### Infrastructure
- Un cluster OpenShift avec OpenShift Virtualization installé
- Un load balancer configuré pour le cluster à déployer
- DNS configuré pour résoudre:
  - `api.<cluster_name>.<base_domain>`
  - `*.apps.<cluster_name>.<base_domain>`
  - Tous les nœuds individuels

### Poste de Travail
- Python 3 avec les packages:
  - `jmespath`
- Collections Ansible:
  - `kubernetes.core`
  - `kubevirt.core`
  - `community.crypto`
  - `community.general`
  - `ansible.posix`
- `nmstatectl` (pour macOS: `brew install nmstatectl`)
- Pull secret OpenShift (dans `~/pullsecret`)

## Utilisation

### Déployer un Nouveau Cluster

1. **Créer le fichier de variables**:
   ```bash
   cp host_vars/host.sample host_vars/mycluster
   # Éditer host_vars/mycluster avec votre configuration
   ```

2. **Ajouter le cluster à l'inventaire**:
   ```ini
   [cluster_to_deploy]
   mycluster
   ```

3. **Lancer le déploiement**:
   ```bash
   ansible-playbook deploy.yaml
   ```

4. **Attendre 30-40 minutes** pour l'installation complète

### Ajouter des Nœuds

1. **Modifier le fichier de variables** du cluster existant:
   ```yaml
   nodes:
     worker:
       - { ip: 10.3.59.109, cpu: 24, ram_gb: 48, disk_size_gb: 300 }  # Nouveau
   ```

2. **Lancer l'ajout de nœuds**:
   ```bash
   ansible-playbook add_node.yaml
   ```

## Fonctionnalités Avancées

### Support du Proxy
Configuration possible d'un proxy HTTP/HTTPS pour l'accès Internet.

### Déploiement Air-Gap
Support du déploiement dans des environnements déconnectés avec registry miroir.

### Manifests Personnalisés
Possibilité d'ajouter des manifests Kubernetes personnalisés (ex: MachineConfig pour la configuration des disques).

### Types de Nœuds
- **Master** : Control plane du cluster
- **Worker** : Nœuds de calcul pour les workloads applicatifs
- **Infra** : Nœuds dédiés aux composants d'infrastructure (monitoring, logging, registry)

## Sécurité

- Utilisation de clés SSH générées automatiquement
- Pull secret OpenShift requis pour l'accès aux images
- Support de l'authentification par token ou username/password pour oVirt
- Possibilité d'activer FIPS mode

## Logs et Débogage

- Logs Ansible dans `logs_deployed/deploy_VM_playbook.log`
- Données de cluster dans `clusters_data/<cluster_name>/`
- Configurations sauvegardées (install-config.yaml, agent-config.yaml)
- Kubeconfig généré dans `clusters_data/<cluster_name>/install-dir/auth/kubeconfig`

## Conclusion

Ce projet fournit une solution complète et automatisée pour déployer des clusters OpenShift sur OpenShift Virtualization. Il gère l'ensemble du cycle de vie, du déploiement initial à l'ajout de nœuds, en passant par la configuration du stockage distribué avec ODF.
