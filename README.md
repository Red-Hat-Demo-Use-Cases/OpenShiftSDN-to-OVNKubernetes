## Migration d'OpenShiftSDN vers OVN-Kubernetes
https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/networking/ovn-kubernetes-network-plugin



0. Faire les validations
- S'assurer qu'il n'y a pas d'UDN pour OCP-V
- https://access.redhat.com/solutions/7057169

- Valider qu'il n'y a pas d'egress utilisé:
`oc get pods -A -o json | jq '.items[] | select(.metadata.annotations."pod.network.openshift.io/assign-macvlan" == "true") | {name: .metadata.name, namespace: .metadata.namspace}'`



1. Débuter la migration en indiquant au cluster que nous désirons un `networkType: OVNKubernetes`.

`oc patch Network.config.openshift.io cluster --type='merge' --patch '{"metadata":{"annotations":{"network.openshift.io/network-type-migration":""}},"spec":{"networkType":"OVNKubernetes"}}'`

- Après l'exécution de cette commande, le processus de migration démarre. Durant ce processus, l'opérateur de configuration de la machine redémarre les nœuds de votre cluster à deux reprises. La migration prend environ deux fois plus de temps qu'une mise à niveau du cluster.

- On peu valider que les configuration ont bien été prise en charge via: `oc get networks.operator/cluster -o yaml` <-- Regarder dans `spec.migration: { mode: Live, networkType: OWNKubernetes}`

- Et que la migration a débuté comme cela: `oc get networks.config/cluster -o yaml` <-- Regarder dans `status.conditions[]: { reason: NetworkTypeMigrationStarted, status: "True" }`



2. Durant la migration ...

- L'état des pods du SDN et du OVN  vont peu à peu changer via `oc get pods -n openshift-sdn` et `oc get pods -n openshift-ovn-kubernetes`. Donc les deux SDN fonctionnent en même temps, le temps de la migration.

- Être patient ... ca peu prendre un certain temps avant que les MachineConfig soit mit en état de mise-à-jour et que les node redémarrent.

- On peu regarder l'état des `MachineConfigPool` via cette commande `oc get mcp`, cela nous indiquera dans quel phase les MachineConfigPool sont. Un MCP regroupe plusieurs regroupe plusieurs MachineConfig, et s'occupera de redémarrer dans une séquence afin de ne pas affecter le travail en cours du cluster.

- Une fois que les MCP indiquent être en Updating, on peu regarder le travail qui est fait sur chaque node en utilisant `oc get nodes`, on verra que certaines nodes sont en SchedulingDisabled, ce qui signifi qu'elles sont en train d'être mise-à-jour.

- Enfin, une fois que `oc get mcp` indiquera que toutes les MCP ont été mise-à-jour, on peut valider avec `oc get nodes` que toutes les nodes sont maintenant de retour et fonctionnement.

- ATTENTION, la mise-à-jour n'est pas terminée, un cycle de redémarrage sera fait de nouveau, cette fois pour retirer les configuration openshift-sdn. Continuer de valider avec les commandes `oc get mcp`, `oc get nodes` et `oc get networks.config/cluster -o yaml`, cette dernière commande vous donnera le plus de détails quant-à l'état de le migration, dans la sections status.



3. Valider que la migration est terminé

- En utilisant, encore une fois, la commande `oc get networks.config/cluster -o yaml`, et en prêtant attention aux champs: `NetworkTypeMigrationNotInprogress` et `NetworkMigrationCompleted`

- Valider que les ClusterOperator sont OK  `oc get co`



4. Faire le ménage ...

- Indiquer que la migration est termié: `oc patch Network.operator.openshift.io cluster --type='merge' --patch '{ "spec": { "migration": null } }'`

- Retirer les configuration OpenShiftSDN dans le network operator: `oc patch Network.operator.openshift.io cluster --type='merge' --patch '{ "spec": { "defaultNetwork": { "openshiftSDNConfig": null } } }'`

- Supprimer le namespace openshift-sdn `oc delete ns openshift-sdn` ( attendre… )

5. Dernière validation

- S'assurer que tous les ClusterOperateur sont encore en santé `oc get co`



## Commandes utilies:

| Commande                                       | Description                                     |
| :--------------------------------------------- | : --------------------------------------------- |
|                                                |                                                 |
|  `oc get clusterversion`                       |  Voir quel réseautique est utilisé par le       |
|                                                |  cluster                                        |
|                                                |                                                 |
|  `oc get networks.config/cluster -o yaml`      |  Prendre l'information réseau du cluster        |
|                                                |                                                 |
|  `oc get networks.operator/cluster -o yaml`    |  Valider l'état de la migration                 |
|                                                |                                                 |
|  `oc get co`                                   |  Valider que les configuration sont bien        |
|                                                |  fonctionnels en regardant les cluster operators |
|                                                |                                                 |
|  `oc get nodes`                                |  Regarder l'états des nodes                     |
|                                                |                                                 |
|  `oc get mcp`                                  |  Regarder l'états des machines pool             |
|                                                |                                                 |


## Métrique pouvant être regardé durant la migration

Counter: openshift_network_operator_live_migration_condition

## Resources
- https://www.youtube.com/watch?v=7Jr8k86p7Q0
- https://www.youtube.com/watch?v=_1mULoOtTwA
