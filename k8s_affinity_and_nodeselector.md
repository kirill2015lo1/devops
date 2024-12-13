Документация на использование affinity для pods:

https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector

Используется для указание deployment на каких узлах надо разворачивать pods, в данном случае на нодах с label, которые умеют название <SOME_KEY> и со значением <SOME_VALUE>, 
буду создавать поды
```
kind: Deployment
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: <SOME_KEY>  # Указать label, по которому будет идти отбор, пример key: node-role.kubernetes.io/control-plane
                operator: In     # In, NotIn, Exists, DoesNotExist, Gt и Lt
                values:
                - <SOME_VALUE>   # значение label, пример -""
                - <SOME_VALUE>
```
Конкретный пример, если при создании deployment мы укажем такие параметры, то все Поды будут создаваться на узлах с label - topology.kubernetes.io/zone="antarctica-east1", 
или на узлах с label: topology.kubernetes.io/zone="antarctica-west1", но если из этих узлов будет узел с label - high: veryvery, все поды будут запускаться на нем, 
пока ресурсы не истекут, ведь preferredDuringSchedulingIgnoredDuringExecution указывает предпочтительный узел:
```
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                - antarctica-east1
                - antarctica-west1
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1   #чем больше значение, тем выше приоритет для создание подов на этом узле
            preference:
              matchExpressions:
              - key: high
                operator: In
                values:
                - veryvery
```
