# Como o Kubernetes resolve problemas de Infra?

Na primeira sessão, descrevi o Kubernetes como:

> "Um **framework de infraestrutura** declarativo para contêineres operado de maneira autônoma por controladores ("controllers"). 

A principal meta do Kubernetes é permitir que você descreva elementos-chave de uma boa arquitetura de aplicações durante o processo de deploy, sem precisar ter que solicitar qualquer tipo de recurso extra ou trabalho de outra equipe.

Pode-se dizer que uma das principais vantagens do Kubernetes é resolver problemas de infraestrutura **para os desenvolvedores**!

E o que acontece com a equipe de infra?

Nesse ponto de vista, o trabalho da equipe de infraestrutura não é extinto; ele apenas transiciona de atender a demandas específicas de cada projeto para viabilizar a manutenção de todo o conjunto de tecnologias usado pelo cluster Kubernetes para atingir seus objetivos.

# Como o Kubernetes funciona? 

A partir da minha definição acima, o funcionamento do Kubernetes pode ser inferido a partir de apenas duas das palavras:

* **declarativo**
* **controllers**

Mas exatamente o que elas querem dizer?

## O paradigma declarativo

A ideia do **Declarativo** serve para contrastar com o principal paradigma de linguagens de computação, o **Imperativo**, que, em termos técnicos, define unidades sintáticas que mudam o estado do programa.

Se você não entendeu, é só porque é mais difícil de explicar que exemplificar.

### Imperativo 
Programas imperativos são sequências de ações consecutivas. Se você precisa computar a soma dos dez primeiros números primos, a maneira mais convencional de executar esta operação é por meio de definição de variáveis, laços e testes condicionais:

```java
# Código que calcula a soma dos dez primeiros números primos.

public static void main(String[] args) {
        long sum = 0;
        int count = 0;
        int number = 2;
        while (count < 1000) {
            # Cuidado! Não é a soma dos primos menores que dez!
            if (isPrime(number)) {
                sum += number;
                count++;
            }
            number++;
        }
        System.out.println(sum);
    }
```

O paradigma imperativo não é limitado à programação. Ao escrever um script em shell que baixa o binário da aplicação, instala pacotes de dependência e configura o ambiente para executar a aplicação, você está usando de técnicas **imperativas** para fazer sua automação. Um pipeline que executa diversas etapas de "*build*", "*test*", "*deploy*" também opera de maneira imperativa.

O exemplo abaixo mostra um arquivo de **Gitlab CI** que poderia ser usado para automatizar o processo de compilar e testar uma aplicação em **Go**:

```YAML

# Pipeline para Gitlab CI que automatiza operações de uma aplicação Go
image: golang:latest

variables:
  # Trocar pelo nome do projeto aqui.
  REPO_NAME: gitlab.com/namespace/project

before_script:
  - mkdir -p $GOPATH/src/$(dirname $REPO_NAME)
  - ln -svf $CI_PROJECT_DIR $GOPATH/src/$REPO_NAME
  - cd $GOPATH/src/$REPO_NAME

stages:
  - test
  - build
  - deploy

format:
  stage: test
  script:
    - go fmt $(go list ./... | grep -v /vendor/)
    - go vet $(go list ./... | grep -v /vendor/)
    - go test -race $(go list ./... | grep -v /vendor/)

compile:
  stage: build
  script:
    - go build -race -ldflags "-extldflags '-static'" -o $CI_PROJECT_DIR/mybinary
  artifacts:
    paths:
      - mybinary

``` 

### Declarativo

A melhor maneira de entender a ideia do declarativo no contexto de infraestrutura é que **não** é imperativo. Enquanto o imperativo foca no **como** operar, **como** atingir seus objetivos, o modo declarativo apenas descreve **o que** deve ser feito, abstraindo o como deve ser feito.

Nos exemplos acima, temos definições de variáveis, laços, execução de comandos... Várias etapas que, individualmente, não são capazes de indicar qual o objetivo final. Você é responsável por elencar as ações necessárias para atingir o estado final, e o processo de implantação e depuração não raramente passa pela tentativa e erro.

Em contraste, o trecho de código abaixo garante a criação de um **Bucket S3** na **AWS** usando **Terraform**:

```
resource "aws_s3_bucket" "bucket" {
  bucket = "kubernetes-the-long-way-bucket"
  acl    = "private"
}
```
Não há qualquer tipo de preocupação com relação a **como** interagir com a API da AWS para criar a aplicação. Você descreve **o que** você quer, e o **terraform** garante o **como**.

### Kubernetes por meio do paradigma declarativo

As configurações de aplicações que são entregues para o Kubernetes descrevem **o que** você quer. O Kubernetes irá ler e seus componentes de software são os responsáveis pela execução.

Por exemplo: 

* Faça o deploy do release 1 da aplicação;
* Mantenha duas réplicas no ar em todos os momentos (para garantir alta disponibilidade);
* O ambiente da aplicação deve conter como variáveis de ambiente o host do banco de dados, o login e a senha.

Isso, traduzido de forma declarativa para o Kubernetes, seria descrever:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aplicacao
  labels:
    app: aplicacao
spec:
  replicas: 2
  selector:
    matchLabels:
      app: aplicacao
  template:
    metadata:
      labels:
        app: aplicacao
    spec:
      containers:
      - name: aplicacao
        image: devsres-com/aplicacao:1
        env:
        - name: DB_HOST
          value: "192.168.0.69"
        - name: DB_USER
          value: aplicacao
        - name: DB_PASS
          value: "esta_senha_nao_deveria_estar_aqui"        
```

Ao submeter este arquivo, descrito na sintaxe YAML, para o Controlplane Kubernetes, ele se encarregará de cumprir os requisitos solicitados. Você não precisa se preocupar com o **como**!

A principal vantagem desse modelo fica visível ao imaginar e comparar  os esforços para fazer o deploy de centenas de aplicações. 

Na maneira **imperativa**, você precisaria descrever cada passo para cada aplicação individualmente, levando em consideração suas particularidades. Ainda que você faça extensivo uso de bibliotecas e templates, cada "instanciador" é sua entidade própria passiva de falhas.

Na maneira **declarativa**, você concentra todo o esforço de código em um único conjunto de software que lê as definições descritas pelos usuários e efetivamente implementa as ações necessárias. Em vez de gerar diversos códigos, um para cada aplicação, você cria um único software, gerando agora um único ponto de testes e validação.

O lado negativo desse modelo? É bem simples: a complexidade envolvida na programação desses componentes que fazem tudo provavelmente é muito superior a envolvida em descrever as sequências individuais, já que cada caso de uso deve ser contemplado.

Á parte boa é que, no **Kubernetes**, da mesma forma que no **Terraform**, alguém se responsabiliza pela programação desses componentes. E esses componentes são os **Controllers**.

## Controllers

O Kubernetes define **Conntrollers** como:

> "Controllers are control loops that watch the state of your cluster, then make or request changes where needed. Each controller tries to move the current cluster state closer to the desired state." 
> ([fonte](https://kubernetes.io/docs/concepts/architecture/controller/))

Sem tentar traduzir a expressão, Controllers são simplesmente programas responsáveis em implementar os passos necessários para implementar as solicitações declarativas do usuário.

Existe uma diferença notável aqui: programas que implementam ações (como o Terraform, ou o Ansible) normalmente executam e concluem. Controllers **nunca** concluem. Eles executam indefinidamente (o tal *control loop*), monitorando se há ações a serem tomadas. Não havendo, ficam aguardando até surgirem novas ações.

Em vez de um único grande componente de software que faz tudo, O Kubernetes fraciona as diversas operações necessárias, criando componentes específicos para monitorar e executar ações. Embora comumente reconheça-se o **Kube Controller Manager** como o componente de software responsável por quase todas as operações de "controle" no Cluster, ma verdade ele é formado por dezenas de componentes individuais (38 na versão 1.21).

Daí fazer mais sentido falar que cada controller busca levar o estado do cluster mais próximo ao **estado desejado**; não existe um único componente responsável por ler um arquivo de recurso gerado pelo usuário e implementá-lo. O deploy de um componente demanda o trabalho "em equipe" de diversos controllers!

# Deploy de aplicações no Kubernetes

A maneira mais comum de fazer o deploy de aplicações é criando um objeto do tipo **Deployment** como o descrito no parágrafo sobre linguagem declarativa:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aplicacao
  labels:
    app: aplicacao
spec:
  replicas: 2
  selector:
    matchLabels:
      app: aplicacao
  template:
    metadata:
      labels:
        app: aplicacao
    spec:
      containers:
      - name: aplicacao
        image: devsres-com/aplicacao:1
        env:
        - name: DB_HOST
          value: "192.168.0.69"
        - name: DB_USER
          value: aplicacao
        - name: DB_PASS
          value: "esta_senha_nao_deveria_estar_aqui"        
```

Os Controllers serão os responsáveis pelo **COMO** realizar o deploy.

As etapas são as seguintes:
* O **Deployment Controller** irá monitorar a criação destes recursos, e irá criar os **ReplicaSets** associados à primeira geração do Deployment;
* O **ReplicaSet Controller**, por sua vez, monitora o estado dos **ReplicaSets**, criando quantos **Pods** foram descritos no campo *Replicas*. Os **Pods** são criados usando o *Template* descrito no objeto.
* O **Scheduler**, que existe **fora** do **Kube Controller Manager**, mas que também pode ser entendido como um Controller, monitora o processo de criação de Pods. Ele irá decidir **em que máquina** do cluster Kubernetes cada **Pod** irá executar.
* Por fim, cada máquina do cluster Kubernetes executa um **Kubelet**, que também pode ser entendido como um Controller. Ele monitorará a existência de **Pods** que estejam associados à máquina em que se encontra e irá fazer a operação necessária para subir os contêineres descritos pelo usuário, fazendo efetivamente o trabalho junto ao software responsável pela criação de containers no sistema operacional (componentes que implementam o **CRI**, como **containerd** ou o **docker-shim**, uma implementação descontinuada para permitir ao Kubernetes usar Docker).

O Kubelet também é o responsável por garantir que os contêineres criados respeitem todas as especificações descritas pelo usuário, como por exemplo as variáveis de ambiente, montagem de volumes e conectividade de rede. 

Parece confuso? É porque realmente é complexo. Esse é o preço de implementar um único conjunto de software que implementa todas as ações: é necessário que ele implemente não apenas o básico, mas todo o conjunto de funcionalidades que permita o deploy de **qualquer** coisa, seja simples ou complexa.

Esse modelo de alta complexidade, por outro lado, viabiliza, em software, o atendimento de diversas demandas de infraestrutura para o usuário final.

E a melhor parte: a maioria dos **usuários** podem ignorar solenemente esses detalhes e apenas usar o framework de infraestrutura que é disponibilizado. Os **administradores** dos Clusters Kubernetes, em vez de se preocupar em provisionar infraestrutura para as equipes, podem focar em manter o cluster operacional, funcional e atualizado.

Kubernetes é um framework de infraestrutura que visa atender aos **desenvolvedores** diretamente, viabilizando operações que, normalmente precisariam de interação com outras equipes para acontecerem!

## E o redeploy com novas versões?

A maneira mais comum de lançar atualizações de versões de software [é promover o redeploy usando a nova versão (no caso acima, seria atualizar o artefato de imagem de container, possivelmente mudando sua *Tag*).

Um redeploy substituindo todas as instâncias em execução causa indisponibilidade e é pouco desejável; o Kubernetes implementa a ideia de **Rolling Update**, em que as diversas instâncias de uma aplicação lançada por meio de um **Deployment** são substituídas aos poucos, em um ritmo configurável pelo próprio usuário.

```yaml
# Para fazer o redeploy, modifiquei a versão. 
# Além disso, a estratégia de rollingupdate foi customizada.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aplicacao
  labels:
    app: aplicacao
spec:
  replicas: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1  
  selector:
    matchLabels:
      app: aplicacao
  template:
    metadata:
      labels:
        app: aplicacao
    spec:
      containers:
      - name: aplicacao
        image: devsres-com/aplicacao:2
        env:
        - name: DB_HOST
          value: "192.168.0.69"
        - name: DB_USER
          value: aplicacao
        - name: DB_PASS
          value: "esta_senha_nao_deveria_estar_aqui"
```
O trecho relevante para análise aqui é o excerto abaixo:

```
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1 
```

A estratégia de redeploy **RollingUpdate** viabilizará a atualização de algumas das réplicas da aplicação enquanto as mais antigas persistem; assim havendo o mínimo de impacto possível. É possível configurar:

* Quantas instâncias da aplicação podem ficar indisponíveis durante a atualização (o parâmetro **maxUnavailable**);
* Quantas instâncias podem ser executadas acima do número originalmente especificado (o parâmetro **maxSurge**).

> Atenção: não foque na necessidade de 'decorar' as sintaxes ou os parâmetros agora. Esta é a sessão de **apresentação** das funcionalidades do Kubernetes, e o objetivo não é explorar cada detalhe dos objetos!










