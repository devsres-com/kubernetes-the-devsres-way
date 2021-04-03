# Ingresses

## Criação de aplicações para testes

Temos este guia aqui:

https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/

```
$ kubectl create deployment web --image=gcr.io/google-samples/hello-app:1.0
$ kubectl expose deployment web --type=NodePort --port=8080
$ kubectl create deployment web2 --image=gcr.io/google-samples/hello-app:2.0
$ kubectl expose deployment web2 --port=8080 --type=NodePort
```

Os *Services* não precisam ser *NodePort*, mas ajudam a testar o funcionamento do programa.

Você pode especificar o *NodePort* também. Se você não o fizer, ele vai abrir uma porta aleatória no range padrão 30000-32767. Eu preferi fazer isso para mostrar o poder de jsonpath na próxima linha de comando!

Com ver a porta criada?

```
$ kubectl get svc -n hello-app -o custom-columns="Name:.metadata.name,Nodeport:.spec.ports[?(@.nodePort)].nodePort"
Name   Nodeport
web    32427
web2   31765 
```

Para testar a saída dos programas, pode usar o curl com resolve apontando para os endereços IPs dos nós:

```
$ k get nodes -o custom-columns="Name:.metadata.name,ExternalIP:.status.addresses[?(.type=='Ex
ternalIP')].address"
Name                             ExternalIP
ip-192-168-2-94.ec2.internal     54.227.74.57
ip-192-168-53-125.ec2.internal   34.205.50.116
```

Agora você pode montar a linha de comando do curl com --resolve para simular o hostname da seguinte maneira:

```
$ curl --resolve hello.devsres.com:32427:54.227.74.57 hello.devsres.com:32427
Hello, world!
Version: 1.0.0
Hostname: web-6785d44d5-bpdpw

$ curl --resolve hello.devsres.com:31765:54.227.74.57 hello.devsres.com:31765
Hello, world!
Version: 2.0.0
Hostname: web2-8474c56fd-pg4cc
```

Ou, com shell script:

```

$ for port in $( kubectl get svc -n hello-app -o jsonpath="{.items[*].spec.ports[?(@.nodePort)].nodePort}" ) ; do curl --resolve hello.devsres.com:$port:54.227.74.57 hello.devsres.com:$port ; done
Hello, world!
Version: 1.0.0
Hostname: web-6785d44d5-8297r
Hello, world!
Version: 2.0.0
Hostname: web2-8474c56fd-pg4cc

```

## Nginx Ingress Controller

https://kubernetes.github.io/ingress-nginx/deploy/#aws

O modo padrão de operação é posicionar o Ingress Controller por tras de um NLB (network load balancer) da AWS, que atua na camada 4, e delegar a atividade 'ALB' para o próprio Ingress Controller

Mas também é possível configurar o NLB para fazer a terminação TLS. Como este é um use case bem específico, vou deixar para lá.

A instalação padrão vai fazer com que o Ingress Controller apenas funcione com objetos ingress especificados com a ingress class "nginx" explicitamente. Ative isso ou passe vergonha nas lives do Twitch!

O deploy do Nginx Ingress Controller irá instanciar todos os recursos necessários para seu funcionamento.

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.44.0/deploy/static/provider/aws/deploy.yaml
...
```

**IMPORTANTE**: a configuração acima irá instanciar um *Elastic Load Balancer* por meio de um Service do tipo LoadBalancer. Esses caras geram cobrança, então lembre-se disso ao subir o cluster.

Observe o *Service* gerado:

```
$ k get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                             PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.228.213   ae4ae75c4b0b44df89c72a0998fa66e5-16548135.us-east-1.elb.amazonaws.com   80:31266/TCP,443:30314/TCP   139m
```

A coluna *External-IP* mostra a associação com o hostname *Load Balancer*. Mas vamos abstrair essa etapa por enquanto e concentrar no NodePort associado ao *Service*:

```
$ kubectl get svc -n ingress-nginx -o custom-columns="Name:.metadata.name,Nodeport:.spec.ports
[?(@.nodePort)].nodePort"
Name                                 Nodeport
ingress-nginx-controller             31266,30314
ingress-nginx-controller-admission   <none>
```

São duas portas, porque uma está associada à porta 80, e a outra a 443. Para testar o acesso das aplicações, vamos usar a que está associada à 80(dá para ver na saída do outro comando de cima).

## Criando objeto ingress para a aplicação

O yaml abaixo irá criar um objeto ingress para as duas aplicações da namespace já associando a *IngressClass* apropriada. Como o cluster é 1.18, ainda estou usando as annotations (**TODO** não sei se o nginx ingress controller já suporta o objeto nativo IngressClass do Kubernetes).

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
  namespace: hello-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: hello.devsres.com
    http:
      paths:
      - path: /v1
        backend:
          serviceName: web
          servicePort: 8080
      - path: /v2
        backend:
          serviceName: web2
          servicePort: 8080
```

Uma vez criado o arquivo acima, podemos testar o acesso às aplicações usando o NodePort do Ingress Controller.

```
curl http://hello.devsres.com:31266/v2 --resolve hello.devsres.com:31266:34.205.50.116
Hello, world!
Version: 2.0.0
Hostname: web2-8474c56fd-pg4cc

marcelo@devsres:~/tmp/ingress$ curl http://hello.devsres.com:31266/v1 --resolve hello.devsres.com:31266:34.205.50.116
Hello, world!
Version: 1.0.0
Hostname: web-6785d44d5-8297r
```

Se quiser acessar por meio do *Load Balancer* associado ao *Ingress Controller*, basta resolver o endereço IP e passar ao mesmo comando - mas agora, na porta 80:

```
$ host ae4ae75c4b0b44df89c72a0998fa66e5-16548135.us-east-1.elb.amazonaws.com
ae4ae75c4b0b44df89c72a0998fa66e5-16548135.us-east-1.elb.amazonaws.com has address 100.25.250.37

$ curl http://hello.devsres.com/v1 --resolve hello.devsres.com:80:100.25.250.37
Hello, world!
Version: 1.0.0
Hostname: web-6785d44d5-bpdpw

$ curl http://hello.devsres.com/v2 --resolve hello.devsres.com:80:100.25.250.37
Hello, world!
Version: 2.0.0
Hostname: web2-8474c56fd-pg4cc
```


