
## Utilisation des GPU/MIG sur Openshift :

#### Comment requêter un type de ressource GPU ?

Une ressource GPU est considérée comme une ressource étendue et présentée sous le nom **nvidia.com/gpu**.

Un pod va utiliser les champs spec/resources/limits et/ou spec/resources/requests pour demander des GPU (comme pour les ressources standards : CPU, mémoire, ...).

    ex: apiVersion: v1
    kind: Pod
    metadata:
      name: cuda-vectoradd
    spec:
     restartPolicy: OnFailure
     containers:
     - name: cuda-vectoradd
       image: "nvidia/samples:vectoradd-cuda11.2.1"
       resources:
         limits:
           nvidia.com/gpu: 1

Il est aussi possible de rajouter des node selector mais ils ne serviront qu'à rajouter une contrainte supplémentaire sur les noeuds sur lesquels sera cherché la ressource GPU demandée.

      nodeSelector:
        nvidia.com/gpu.product: A100-SXM4-40GB-MIG-1g.5gb

Sur les noeuds qui ont des GPU disponibles, la ressource **nvidia.com/gpu** sera décrite comme pour une ressource CPU :   
 * Capacity: nombre de ressources physiques   
 * Allocatable: nombre de ressources pouvant être utilisée (hors réservations systèmes)   
 * Allocated Resources: nombre de resource déjà réservée sur le noeud.   

#### Comment configurer les MIG ?

Un profil différent peut-être appliqué sur chaque noeud pour spécifier le partitionnement des GPU présents.

Dans un "config map", sera défini l'ensemble des profils de partitionnement possibles.
 
La reconfiguration dynamique des MIG sur un noeud se concrétise par l'application d'un label sur le noeud qui spécifie le profil voulu :    

     oc label node/$NODE_NAME nvidia.com/mig.config=$MIG_PROFIL    

_Le temps de reconfiguration n'est pas précisé_ (à ne pas confondre dans la doc avec les 10-20 min pour changer le mode global single/mixed).

Il faut drainer le noeud (ou du moins l'éviction des pods qui utilisent déjà des MIG sur ce noeud) avant de reconfigurer le partitionnement.
 
#### Quel partitionnement possible avec les MIG ?

Selon le type de GPU, le GPU pourra être découpé en plus petites unités : 

 * exemple : sur le A100-40GB, il est possible de le découper en 8 sous unités, la 8ème unité sera réservée pour le partitionnement.

Chaque GPU pourra être partitionné différement.
 
Un exemple de profils de partitionnement : 
 
     apiVersion: v1
     kind: ConfigMap
     metadata:
       name: mig-parted-config
       namespace: gpu-operator-resources
     data:
       config.yaml: |
         version: v1
         mig-configs:
           all-1g.5gb:
             - devices: all
               mig-enabled: true
               mig-devices:
                 "1g.5gb": 7

           profil1:
             - devices: 0
               mig-enabled: true
               mig-devices:
                 "1g.5gb": 2
                 "2g.10gb": 1
                 "3g.20gb": 1

             - devices: 1
               mig-enabled: true
               mig-devices:
                 "1g.5gb": 5
                 "2g.10gb": 1
 
#### Comment requêter un type de ressource MIG ?
 
Vous aller alors avoir de nouvelles ressources représentant les MIG sur le noeud (en remplacement de nvidia.com/gpu)

Ces ressources seront donc utilisable comme un GPU (spec/resources/...) :
 
    resources:
      requests:
        nvidia.com/mig-2g.10gb: 1

Par exemple, la ressource nvidia.com/mig-2g.10gb sera vue aussi comme une ressource étendue sur le noeud : 
 
     allocatable : 
        "nvidia.com/mig-2g.10gb": "3",

           
## Monitoring des GPU/MIG sur Openshift :

#### Quel monitoring est disponible ?

Les métriques GPU/MIG sont bien intégrées sur le prometheus d'openshift (les métriques DCGM). 

Le grafana de l'opérateur de monitoring n'est pas customisable mais via l'opérateur grafana, il est possible de rajouter une instance grafana qui pointe sur le prometheus déjà déployé (avec l'opérateur de monitoring) et dans lequel, des dashboard utilisant ces métriques peuvent être installés (https://docs.nvidia.com/datacenter/cloud-native/gpu-telemetry/dcgm-exporter.html#gpu-telemetry).

En version Openshift 4.11, des dashboards GPU seront directement disponibles dans la console Openshift (Overview, Observe/Dashboard, ...).

Il y a aussi via la commande **oc describe node**, la vision des ressources (capacity, allocatable, allocated) des GPU/MIG comme pour les cpu.

Il est aussi possible de lancer des commandes spécifiques sur les **daemon set** de l'operateur GPU avec la commande nvidia-smi pour avoir plus de détails.

#### Quels type de métriques sont remontées ?

Les métriques DCGM disponibles : 

            | A.1            | 1002     | sm_active                                            |
            | A.1            | 1003     | sm_occupancy                                         |
            | A.1            | 1004     | tensor_active                                        |
            | A.1            | 1007     | fp32_active                                          |
            | A.2            | 1006     | fp64_active                                          |
            | A.3            | 1008     | fp16_active                                          |
            | B.0            | 1005     | dram_active                                          |
            | C.0            | 1009     | pcie_tx_bytes                                        |
            | C.0            | 1010     | pcie_rx_bytes                                        |
            | D.0            | 1001     | gr_engine_active                                     |
            | E.0            | 1011     | nvlink_tx_bytes                                      |
            | E.0            | 1012     | nvlink_rx_bytes                                      |

Les métriques sur les MIG sont elles aussi disponibles.

#### Les alertes sur les GPU :

Des alertes (prometheus alertmanager) sont configurées (via l'opérateur GPU) sur le status des GPU.

#### Vision des quotas GPU pour les projets :

Les gpu (et ressources MIG) sont considérées comme des ressources étendues et il est possible de créer des quotas en réservation (requests), mais pas en limites.

            apiVersion: v1
            kind: ResourceQuota
            metadata:
              name: gpu-quota
              namespace: nvidia
            spec:
              hard:
                requests.nvidia.com/gpu: 1

_Pas de confirmation sur la présence ou non des quotas GPU sur la page Quota du namespace._

_Pas de confirmation sur la prise en compte des ressources étendues via l'opérateur de Cost management._

## Cartographie des GPUs opérées avec Openshift :

_En utilisant le Node Feature Discovery Operateur, il y a définition automatique sur les noeuds des GPUs disponibles._
 
