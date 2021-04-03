# kubernetes-the-devsres-way

Você vai precisar de dois programas: **kubectl** e **eksctl**.

* https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html
* https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html

## Fazendo laboratórios com AWS EKS

Primeiramente, garanta que você está na 'region' que você quer:

```
aws ec2 describe-availability-zones --output text --query 'AvailabilityZones[0].[RegionName]'
```

Agora você pode criar o cluster EKS.

A maneira mais econômica é subir apenas o control plane desabilitando o Nat Gateway. O Controlplane custa apenas 0,10 centavos a hora. Um Nat Gateway da AWS custa no mínimo 0,045 a hora, e não vai servir para absolutamente nada se você não subir nós em subnets privadas.

```
eksctl create cluster --name devsres-ingress-lab --without-nodegroup --vpc-nat-mode Disable
```

Para criar o nodegroup, obviamente vamos usar um spot nodegroup:

```
eksctl create nodegroup --cluster=devsres-ingress-lab --managed --spot --instance-types=c3.large,c4.large,c5.large,c5d.large,c5n.large,c5a.large --nodes-min=2 --node-smax=5

```

O eksctl irá configurar o kubectl local automaticamente.

## E agora?

Não fiz muita coisa.

* [Ingresses](./ingresses/ingress.md)

