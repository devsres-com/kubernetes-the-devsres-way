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

## Declarativo

A ideia do **Declarativo** serve para contrastar com o principal paradigma de linguagens de computação, o **Imperativo**, que, em termos técnicos, define unidades sintáticas que mudam o estado do programa.

Se você não entendeu, é só porque é mais difícil de explicar que exemplificar.

Programas imperativos são sequências de ações consecutivas. Se você precisa computar a soma dos dez primeiros números primos, a maneira mais convenional de executar esta operação é por meio de definição de variáveis, laços e testes condicionais.

O paradigma imperativo não é limitado à programação. Ao escrever um script em shell que baixa o binário da aplicação, instala pacotes de dependência e configura o ambiente para executar a aplicação, você está usando de técnicas **imperativas** para fazer sua automação. Um pipeline que executa diversas etapas de "*build*", "*test*", "*deploy*" também opera de maneira imperativa.

A melhor maneira de entender a ideia do declarativo no contexto de infraestrutura é que **não** é imperativo. Enquanto o imperativo foca no **como** operar, **como** atingir seus objetivos, o modo declarativo apenas descreve **o que** deve ser feito, abstraindo o como deve ser feito.

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

Ao submeter este arquivo, descrito na sintaxe YAML, para o Controlplane Kubernetes, ele se encarregará de cumprir os requisitos solicitados. Você não precisa se preocupar com o como!

## Controllers

TODO








