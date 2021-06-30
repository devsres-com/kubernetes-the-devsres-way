* https://kubernetes.io/docs/reference/labels-annotations-taints/


## Taints

```
kubectl taint nodes node1 tenant=marcelo:NoSchedule

# Nota: e como TIRA o taint?

kubectl taint nodes ip-192-168-62-21.ec2.internal tenant-

```

Código do Toleration:
```
      tolerations:
      - key: "tenant"
        operator: "Equal"
        value: "marcelo"
        effect: "NoSchedule"
```
Observe que existe um campo 'operator' a ser especificado, pois é possível passar uma igualdade (Equal), como foi o caso, mas também é possível usar Exists - o campo 'value' pode ser desprezado neste caso!

Operator:
* Equal
* Exists

```
$ k explain pod.spec.tolerations.operator
KIND:     Pod
VERSION:  v1

FIELD:    operator <string>

DESCRIPTION:
     Operator represents a key's relationship to the value. Valid operators are
     Exists and Equal. Defaults to Equal. Exists is equivalent to wildcard for
     value, so that a pod can tolerate all taints of a particular category.
```

Além do 'label' em si, você também precisa explicitar um **efeito**:
* NoSchedule: Nós não receberão pods; 
* PreferNoSchedule: O Scheduler fará o possível para não atribuir Pods ao nó, mas se não houver alternativa, eles podem subir lá.
* NoExecute: Se existirem Pods em execução no nó naquele momento, eles sofrerão eviction. Você pode controlar o tempo com a diretiva adicional tolerationSeconds.  

```
$ k explain pod.spec.tolerations.effect
KIND:     Pod
VERSION:  v1

FIELD:    effect <string>

DESCRIPTION:
     Effect indicates the taint effect to match. Empty means match all taint
     effects. When specified, allowed values are NoSchedule, PreferNoSchedule
     and NoExecute.
```

## Affinities

A sintaxe é bem maior, prolixa e complicada. Mas obviamente oferece muito mais.

Apenas a título de exemplo: a maior parte as configurações de Kubernetes se limitam a escolher entre labels existentes. Com nodeAffinity, por exemplo, você pode usar a diretiva matchFields em vez de matchExpressions para fazer associações com o nome do nó de maneira consistente (e, aparentemente, apenas com o campo metadata.name).

Existem três tipos de affinities:
* nodeAffinity
* podAffinity
* podAntiAffinity

### nodeAffinity

É um superconjunto dos nodeSelectors.

O seguinte SPEC implementa uma nodeAffinity idêntica ao NodeSelector:

```
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd   
```

O mapa de campos de affinity varia imensamente de acordo com a affinity que você escolher.



