
Gestion de la configuration MIG :

“The NVIDIA GPU Operator version 1.7.0 and above enables OpenShift Container Platform administrators to dynamically reconfigure the geometry of the MIG partitioning. The geometry of the MIG partitioning is how hardware resources are bound to MIG instances, so it directly influences their performance and the number of instances that can be allocated. The A100-40GB, for example, has eight compute units and 40 GB of RAM. When the MIG mode is enabled, the eighth instance is reserved for resource management.”

Peut-être serait-il intéressant d’expliciter plus avant ce que signifie ‘reconfiguration dynamique’

la reconfiguration dynamique des MIG sur un noeud se concrétise par l'application d'un label sur le noeud qui spécifie le profil voulu (nvidia.com/mig.config)
le temps de reconfiguration n'est pas précisé (à ne pas confondre dans la doc avec les 10-20 min pour changer le mode global single/mixed)
il semblerait logique et au vu de la node ci dessus, que la reconfiguration du profil des MIG sur un noeud nécessite le drain du noeud (ou du moins l'éviction des pods qui utilisent déjà des MIG sur ce noeud)
en effet, cela serait incohérent d'avoir des noeuds qui utilisent des MIG existants alors que l'on va changer ces MIG

The node must be free (drained) of GPU workloads before any reconfiguration is triggered. For guidance on draining a node see, the OpenShift Container Platform documentation Understanding how to evacuate pods on nodes.

GPU monitoring :

« Enable the GPU Operator Dashboard

Follow this guidance to provide GPU usage information in the cluster utilization screen in the OpenShift Container Platform web console.”

Ici aussi il pourrait être intéressant de décrire les fonctionnalités, le type de remontées offertes
Comment la solution avec Opesnhift se positionne par rapport à la solution de monitoring GPU dans Kubernetes en général et l’utilisation de DCGM (https://docs.nvidia.com/datacenter/cloud-native/gpu-telemetry/dcgm-exporter.html#integrating-gpu-telemetry-into-kubernetes ) ?
Et la solution avec Openshift autorise-t-elle cette architecture avec Promotheus/Grafana (https://docs.nvidia.com/datacenter/cloud-native/gpu-telemetry/dcgm-exporter.html#gpu-telemetry ) comme d’ailleurs demandée par le Client
 
les métriques GPU sont bien intégrablent sur le prometheus d'openshift (les métriques DCGM), mais le grafana de base n'est pas customisable à ma connaissance mais via l'opérateur grafana, il est possible de rajouter une instance grafana qui pointe sur le prometheus déjà déployé (avec l'opérateur de monitoring) et dans lequel, des dashboard utilisant ces métriques peuvent être installés 

Cartographie des GPUs oéprées avec Openshift :

« Installing the Node Feature Discovery (NFD) Operator …”

J’ai tendance à penser qu’avec ce Node Feature Discovery Operateur, nous avons accès à la dexscription des GPUs et Instances MIG déployées ?
Il serait intéressant de décrire les fonctionalités offertes
 
il y a déjà via le oc describe node, la vision des ressources (capacity, allocatable, request) des GPU/MIG comme pour les cpu, ils sont vus comme des ressources étendues
après, il est possible de lancer des commandes spécifiques sur les daemon set de l'operateur gpu pour voir d'autres détails

Access à NGC (air gapped ou Proxy Cache) :

« NVIDIA AI Enterprise with OpenShift

NVIDIA AI Enterprise is an end-to-end, cloud-native suite of AI and data analytics software, optimized, certified, and supported…”

Dans cettes section il est fait mention du setup pour la connection à NGC…
Quid de la possibilité avec Openshift de télécharger le depot NGC pour l’avoir en local ?
Le client passe par JFROG Artifacory pour accéder aux depots Editeurs externes…avec Openshift ?

En mode déconnecté, si le repo NGC est proxy/caché par l'archifactory, l'openshift peut l'adresser.
Il est aussi possible, d'utiliser l'opérateur registry en interne du cluster et de récupérer localement les images NGC (mais il faudra référencer les images une par une)



monitoring des gpu (que remonte sur prometheus et les dashboard)

The GPU Operator generates GPU performance metrics (DCGM-export), status metrics (node-status-exporter) and node-status alerts. 
For OpenShift Prometheus to collect these metrics, the namespace hosting the GPU Operator must have the label openshift.io/cluster-monitoring=true.

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
| E.0            | 1012     | nvlink_rx_bytes  
Les métriques sur les MIG sont elles aussi disponibles.
Il y a des dashboard grafana DCGM qui peuvent être installés.


vision des quotas gpu pour les projets (sur la console comme pour les cpu)

   les gpu (et ressources MIG) sont considérées comme des "extended ressources" et il est possible de créer des quotas en request (pas en limites)

apiVersion: v1
kind: ResourceQuota
metadata:
  name: gpu-quota
  namespace: nvidia
spec:
  hard:
    requests.nvidia.com/gpu: 1

pas de confirmation sur la présence ou non des quotas gpu sur la page Quota du namespace

a-t-on les mêmes métriques gpu que pour les cpu (pod, labels, ....) pour charge back

pas de confirmation sur la prise en compte des "extended resources" sur le charge back

quels outils de monitoring du gpu ?

des alertes sont configurées sur le status des GPU et les daemonset de l'opérateur GPU peuvent être utilisés pour lancer la commande nvidia-smi

comment un utilisateur requête un type de ressources gpu (requests, node selector, labels ?)

par défault, un pod va utiliser les infos spec/resources/limits et/ou request pour demander des gpu (comme pour les cpu)

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

mais il est aussi possible de rajouter des node selector mais ils ne serviront qu'à rajouter une contrainte supplémentaire sur les noeuds sur lesquels sera cherché la ressource gpu demandée

sur les noeuds qui ont des gpu prêts, la ressource nvidia.com/gpu sera défini en capacity et allocatable comme les cpu

comment on partitionne les gpu sur les noeuds (en mig, on peut partionner différenment sur les noeuds, pas besoin de noeuds identiques)

il est possible de configurer un GPU en MIG hétérogènes (cf https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/openshift/mig-ocp.html#configuring-mig-devices-in-openshift)

pour appliquer la configuration, il faut labelliser le noeud avec le profil de partionning défini
oc label node/$NODE_NAME nvidia.com/mig.config=$MIG_CONFIGURATION --overwrite
donc le partitionnement peut être différent sur chaque noeud

vous aller alors avoir une nouvelle ressource représentant le ou les mig sur le noeud (en remplacement de nvidia.com/gpu)

allocatable : 
"nvidia.com/mig-2g.10gb": "3",
cette ressource sera donc utilisable comme un gpu (spec/resources/...)
    resources:
      limits:
        nvidia.com/mig-2g.10gb: 1
si les noeuds sont configurés de manière homogène (même profil MIG sur un noeud : single versus mixed)
vous pouvez requeter des gpu et rajouter un node selector pour spécifier quel type de MIG vous recherchez
    resources:
      limits:
        nvidia.com/gpu: 1
  nodeSelector:
    nvidia.com/gpu.product: A100-SXM4-40GB-MIG-1g.5gb
quels impacts pour changer les partioning mig  (temps, drain node, ...)

The node must be free (drained) of GPU workloads before any reconfiguration is triggered. For guidance on draining a node see, the OpenShift Container Platform documentation Understanding how to evacuate pods on nodes.
Il faut donc les droits de drain de node (admin) pour reconfigurer un GPU.

qui peut faire le changement de partitioning (profil admin, ...)

           Il faut avoir les droits de labelliser les noeuds et les droits de créer le profil de MIG (si besoin) donc des droits sur l'opérateur GPU.

peut-on monter a 2T de mémoire sur 1 pod (ou du moins 250G)

je n'ai pas vu de limites max sur la mémoire (limites sur les containers), après il faut trouver la mémoire disponible un noeud
