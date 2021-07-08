# Por que usar Kubernetes?

O maior desafio para implantar Kubernetes não é a gigantesca quantidade de "partes móveis" e "opcionais"; é, na verdade, entender que problema o Kubernetes veio resolver e se ele pode ajudar a sua equipe/empresa a atingir os objetivos. 

Levando em consideração minha tendência natural de ter dificuldade de concisão, precisamos de um pouco de história da infra-estrutura para entender os problemas a contêinerização e o Kubernetes (que são duas coisas bem diferentes) vieram resolver.

# História da infraestrutura

## Pré-história - Instalações de máquinas físicas

### Ops

Não precisamos voltar tão longe para essa análise ser relevante; basta voltar poucas décadas atrás, para um tempo em que se usava exclusivamente instalação de sistemas operacionais "convencionais" diretamente nos hardwares físicos.

Nessa época, era difícil consolidar múltiplos serviços em um único servidor - você precisava que o servidor de aplicação instalado na máquina fizesse isso nativamente, ou seria necessário instalar múltiplas instâncias em portas diferentes (alguém pegou a época disso?).

Além disso, como consolidar diferentes sistemas operacionais? Se sua empresa contasse com um ambiente híbrido com servidores Linux e Windows, você precisaria de instalações individuais com hardware dedicado para cada coisa. 

Se, por acaso, você contasse com ambientes Linux com diferentes distribuições, por exemplo algumas "Enterprise" com licenças e suporte e outras gratuitas, você precisaria de hardware dedicado para todos.

Essa é a época em que máquinas tinham nomes inspirados em constelações, personagens de filmes ou quadrinhos. Eram tratadas e cuidadas como bens preciosos. Implantar alta disponibilidade em ambientes desse tipo é altamente custoso e penoso.

### Dev

Para os desenvolvedores, essa é a época em que os "deploys" eram eventos rarissimos, algumas vezes trimestrais, semestrais ou anuais. 

Os Devs desenvolviam suas aplicações e entregavam, entre os "artefatos", documentos descrevendo as especificações do ambiente em que ela roda, como dependências a serem instaladas, versões do sistema operacional e outras particularidades. Obviamente, não era incomum que esses documentos não refletissem a realidade ou os administradores não seguissem os inúmeros passos.

Essa é a época de ouro do "na minha máquina funciona!".

### Infraestrutura

As principais necessidades de uma boa arquitetura de aplicação não vão mudar muito com o tempo; com poucos equipamentos físicos, a **tolerância a falhas** acaba desempenhando um papel importante, com equipamentos com múltiplas fontes para redundância, uso extensivo de RAID, RAM com ECC e etc., não apenas em termos de hosts mas para todos os ativos de rede e storages.

Mas os principais aspectos de uma boa arquitetura sempre levam em consideração três eixos principais:

* Alta disponibilidade;
* Escalabilidade;
* Monitoração;

Além disso, dois requisitos particularmente inconvenientes também devem ser levados em consideração:

* Segurança;
* Eficiência na operação.

## História recente - Virtualização

### Ops

O uso de hardware dedicado para execução de sistemas e aplicações normalmente gera desperdício computacional. Com o objetivo de consolidar recursos e melhorar o TCO, as tecnologias de virtualização (re)apareceram na história da computação, com diversas ofertas de softwares livres, gratuitos ou pagos para viabilizar essa missão.

Agora, os hardwares (cada vez maiores) agora podiam contar com diversas máquinas virtuais, variando de umas poucas a dezenas ou centenas, dependendo da qualidade do equipamento a que você tem acesso.

Com a substituição das máquinas físicas por virtuais, observamos uma "explosão" em termos de hosts. Isso significa que, agora, em vez de um punhado de máquinas com nomes bonitos, temos dezenas (ou mesmo centenas) de máquinas virtuais, cada uma precisando de um conjunto de configuração, controle de atualizações, patches de segurança e tudo mais que você conseguir imaginar como trabalho de Ops.

A "escalabilidade" das equipes de operações obviamente não podiam seguir na mesma proporção, e essa foi a época em que se observou alta demanda por ferramentas de gestão de configuração ao estilo de Puppet, Chef e Salt, com o propósito de padronizar e viabilizar o lançamento. O modelo principal varia de simples "push" de configurações para até a detecção de falhas e divergências ("drift"). A monitoração precisou escalar igualmente.

### Dev

Para adotar o modelo de virtualização, começou-se a cobrar maior maturidade profissional dos profissionais de operações buscando automação para viabilizar a instalação e configuração das máquinas. 

Os processos de deployment começaram a melhorar; a ideia de "integração contínua" surgiu, evoluindo para o "continuous delivery" e posteriormente o tão sonhado "continuous deployment". O tal do "Manifesto Ágil" começa a ser mais acessível e viável para uma gama maior de pessoas.

As principais tecnologias de pipelines e CI/CD começaram a surgir e o processo de desenvolvimentou passou a se reformular a partir daí, culminando no que temos hoje que chamamos de "Devops", tanto a cultura quanto o cargo. 

### Infraestrutura

A tecnologia de virtualização não tornou desnecessárias as ideias implementadas visando tolerância a falhas, mas trouxe auxílios como as operações de "live migration" e facilidades de recuperar VMs em caso de falhas do Hypervisor hospedeiro. A perda de um equipamento já não é tão crítica quanto no passado.

Combinado a isso, as tecnologias de automação começaram a sugerir que, em vez de recuperar uma máquina que falhou, pode ser uma ideia melhor simplesmente fazer o redeploy de uma nova configurada para a mesma finalidade. Combinar esta ideia com a possibilidade de descartar máquinas ociosas ataca o problema de **escalabilidade**, embora não exista uma maneira padronizada para implementar este recurso - muitas vezes há a dependência de tecnologias proprietárias caras que atendam a este problema.

## História moderna - Conteinerização

### Ops

Se você analisar criticamente, o uso de virtualização introduz uma quantidade significativa de desperdício do ponto de vista de recursos computacionais. Naturalmente, o overhead introduzido pelo uso de dez máquinas virtuais em um hardware físico tende a ser menor do que usar uma máquina dedicada para um único serviço. Ainda assim, este overhead **não** é desprezível.

Uma vantagem adicional da tecnologia de virtualização, além de otimizar o uso de recursos, é a segurança oferecida pelo isolamento das aplicações e sistemas em diferentes instâncias. O processo não é perfeito, mas com todo um hardware "falso" por trás, isso cumpria boa parte do requisito de isolar sistemas diferentes de uma forma que o sistema operacional nativamente não conseguiria.

Mas as coisas mudaram! 

Contêineres não são novidade na computação, já existindo desde antes do conceito de microcomputadores. Da época moderna, vale citar iniciativas como Free BSD Jails e o OpenVZ, mas a ideia começou a ganhar tração com a ideia de **Cgroups**[https://man7.org/linux/man-pages/man7/cgroups.7.html] no Kernel Linux. O propósito deste recurso é:

* limitar o uso de recursos da máquina;
* viabilizar a priorização de consumo de CPU ou IO;
* contabilizar o uso de recursos;
* controlar estes grupos implementando checkpoints e suspensão.

As funcionalidades acima são excepcionalmente úteis pensando no requisito de otimização de recursos ao máximo, mas e a segurança?

Entra na jogada o recurso de (**namespaces**)[https://man7.org/linux/man-pages/man7/namespaces.7.html], cujo foco é particionar recursos do Kernel, garantindo o isolamento. O recurso é mais abrangente que isso, mas muita gente acha que só existe o "PID isolation mode", que é usado amplamente pelas tecnologias de contêineres existentes hoje.

Ao combinar o uso de **Cgroups** com **Namespaces**, temos exatamente aquilo que gostávamos na virtualização - também não de maneira perfeita, mas a ideia é a mesma - sem o a necessidade de ter que manter diversos sistemas operacionais virtuais paralelos. Surge o fundamento tecnológico do que chamamos hoje de **contêiner**.

O ganho em descartar o hardware virtual e o sistema operacional "guest" da virtualização não é só em performance e redução de overhead do consumo de recursos: com menos máquinas virtuais, temos menos atualizações de sistema operacionais, menos necessidade de controle de patches de segurança e todo um outro overhead não tecnológico, mas **administrativo** que existe em lidar com centenas ou milhares de hosts.

Se bem que, na verdade, as tecnologias apresentadas não resolvem o problema de gestão de configuração do que vai "dentro" dos Cgroups e Namespaces - estes recursos permitem isolar e controlar os recursos usados por programas em execução nos hosts, mas ainda temos o problema de fazer tais programas (e suas configurações, e seus ambientes) chegarem aos hosts. 

Como fazer em casos de programas que precisam de dependências distintas? Ou ainda pior, dependências incompatíveis com o sistema operacional que hospeda esses contêineres? 

Ainda faltava uma terceira ideia para viabilizar o uso generalizado da tecnologia.

Essa ideia veio por meio do conceito de **Imagens** de contêineres. Essas "imagens" são, na verdade, pacotes imutáveis de softwares cujo propósito é servir de "blueprint" para os contêineres em execução. Existiram (existem?) diversas técnicas para fazer isso, mas a que se tornou o padrão de mercado foram as imagens de contêiner (**Docker**)[https://www.docker.com/].

O projeto Docker trivializou diversas etapas do processo de trabalhar com contêineres, justificando sua enorme popularidade. Para executar basicamente qualquer coisa na sua máquina sem precisar instalar nada (além do próprio Docker), basta executar um único comando:

```
# docker run nginx
# docker run -it busybox 
docker run --name mongodb -p 27017:27017  -e MONGO_INITDB_ROOT_USERNAME=balta -e MONGO_INITDB_ROOT_PASSWORD=e296cd9f
# docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.13.2
```

O Docker faz o download da imagem apropriada via HTTP, 
lida com toda a burocracia operacional de Cgroups, Namespaces e replicação do conteúdo da imagem para algo executável no filesystem, oferece a possibilidade de customizar o ambiente de várias maneiras (como com variáveis), cria uma infra de rede própria,  pode fazer "port forwarding" do contêiner para o host e faz efetivamente o "bootstrap" dos programas existentes.

Então, por fim, a tríade máxima da conteinerização estava completa: o controle dos recursos com **Cgroups**, o isolamento com **Namespaces** e o conceito de **Imagens** viabilizaram a massificação do uso de contêineres como conhecemos hoje, combinados com um software - o Docker - excepcionalmente útil que faz várias coisas extremamente complexas parecerem triviais.

### Dev

O Docker e a conteinerização foi uma vitória especial para o lado "Dev". Foi a implantação para o mundo das pessoas "normais" ideias e tecnologias que já estavam em uso há anos pelas grandes empresas, mas que eram tratadas como "trade secrets". Agora, o universo de possibilidades de agilidade, dinamicidade e produtividade dependiam apenas de adequar os processos de operação e desenvolvimento à esta nova realidade.

Contêineres passaram a ser o artefato final do **Delivery**; desenvolvimento colaborativo usando Github e similares passou a ser a regra e agora se discute quais são as melhores estratégias para otimizar os pipelines de desenvolvimento de forma a minimizar ao máximo a interação com as áreas de operação. Do deploy anual a milhares de vezes por dia - quem diria!

Combinado ao uso de Clouds públicas, que oferecem diversos recursos que a esmagadora maioria das empresas e pessoas jamais teriam acesso, o limite da capacidade de criação está basicamente na capacidade criativa. Por mais caro que seja usar esses recursos, nem se comparam com os tempos passados de necessidade de compra de máquinas, montagem de infraestrutura física, de rede e refrigeração, sem contar com as possibilidades técnicas de uso de softwares de terceiro a custo de centavos.

### Problemas da conteinerização

Okay, contêineres são a melhor coisa que surgiu e todo mundo adora. Tem um lado negativo?

Claro. O primeiro é um problema semelhante às VMS: se trocamos dezenas de hosts físicos por centenas de máquinas virtuais, agora temos que lidar com milhares (milhões?)! Trabalhar com esta tecnologia da maneira "antiga" se mostra rapidamente inviável em todos os aspectos.

Além disso, os contêineres, por si só, não resolvem os problemas tradicionais de infraestrutura.

Eles não tratam, por exemplo:

* alta disponibilidade
* escalabilidade
* balanceamento de carga

Apenas te oferecem uma unidade básica de lançamento de software excepcionalmente conveniente. Cabe ao administrador usar as técnicas já conhecidas. 

# O que é o Kubernetes

A definição oficial do (https://kubernetes.io/)[seu próprio site] descreve o Kubernetes como um sistema código aberto para automatizar o deploy, o scale e a gerência de contêineres de aplicações.

A minha definição pessoal do Kubernetes descreve o software como um **framework de infraestrutura** declarativo para contêineres operado de maneira autônoma por controladores ("controllers"). 

A ideia é resolver muitos dos principais problemas de infraestrutura usando contêineres como unidade de operação.

## Orquestração

O processo conhecido como "orquestração" é promover o deploy do software desejado no ambiente. O Kubernetes garante não apenas a execução do contêiner em uma das máquinas que compõem seu cluster, mas viabiliza também a criação da rede, volumes, comunicação, resolução de nomes e deploy de configurações adicionais que possam ser necessárias para a execução.

Um diferencial do Kubernetes é permitir a definição do que será "lançado" no cluster de maneira declarativa, por meio de arquivos texto em um formato bem definido (YAML ou JSON) modelado de acordo com os objetos de sua API em vez de descrever ações a serem tomadas para conduzir o mesmo processo (a maneira imperativa).

## Deploy e rollback

O Kubernetes conta com uma técnica de deploy que versiona as modificações realizadas na configuração declarativa do seu componente e oferece a possibilidade de executar um processo de "**Rolling Update**"  da aplicação, na qual as instâncias anteriores são substituídas aos poucos por novas versões. Em caso de falha, o processo é suspenso; o número de instâncias substituídas é configurável.

Por ser versionável, o processo de "**Rollback** para versões anteriores é igualmente trivial.

## Escalabilidade

Não apenas o Kubernetes pode orquestrar o deploy de sua aplicação com suas configuraçòes, mas ele também é capaz de implementar escalabilidade usando apenas diretivas de configuração.

É possível determinar quantas réplicas de um determinado componente da sua aplicação irá executar - assumindo que ela seja capaz de escalar horizontalmente, o chamado **scale out** -, permitindo aumentar a capacidade de processamento de uma aplicação quando for necessário, e reduzindo para dar vez a outras (ou mesmo consumir menos, em caso de nuvens em que se paga pelo uso) .

Mais que isso, também é possível delegar ao Kubernetes que incremente ou decremente automaticamente o número de réplicas de acordo com métricas pré-definidas como CPU ou memória - o recurso de **Horizontal Pod AutoScaling**. Caso o usuário tenha maturidade suficiente, é possível inclusive usar métricas customizadas da própria aplicação, como por exemplo número de requisições, conexões ou tempo de resposta.

## Alta disponibilidade

Já que há a possibilidade de escalar a aplicação, muito do que é necessário para implementar soluções com alta disponibilidade já está pronto. Basta garantir que sempre haja o número mínimo de instâncias em execução em um dado momento.

Recursos como  **PodAffinity** e **NodeAffinity** podem ser usados para garantir que a aplicação espalhará (ou agrupará) seus componentes nos diversos hosts componentes do cluster, limitando o efeito de falhas individuais de máquinas no ambiente.

## Balanceamento de carga

Por meio de técnicas de sistemas operacionais, o Kubernetes implementa nativamente recursos para distribuir a carga entre as diversas réplicas de uma mesma aplicação sem qualquer componente adicional.

Naturalmente, as tecnologias envolvidas são limitadas e não se comparam, em termos de funcionalidades, a balanceadores de carga dedicados; são implementadas, em Linux, por exemplo, por meio de Iptables, IPsets e IPVS, então é de se esperar alguma limitação.

Entretanto, a oferta nativa de um nome e endereço próprio dedicado a esta finalidade elimina diversos problemas de integração entre aplicações e a necessidade de componentes dedicados para esta funcionalidade no cluster.

## Detecção de falhas e auto recuperação

Um dos maiores problemas da infraestrutura é saber se uma aplicação está executando em perfeito estado e, não estando, que ela seja finalizada ou a menos removida do pool que atende aos clientes.

O primeiro recurso que atende a este requisito é o simples fato de que, em caso de falha do contêiner, como finalização do seu processo principal, o Kubernetes irá automaticamente reiniciar aquele contêiner. Isso não resolve o problema causador da falha, mas é excepcionalmente útil para mitigar os efeitos de falhas pontuais causadas por interrupção temporária de serviços, por exemplo.

Além dessa funcionalidade básica, o Kubernetes implementa diversos recursos que permitem aos desenvolvedores programarem detecção automática de falhas a partir de checagens periódicas nos contêineres que fazem parte de uma aplicação.

É possível implementar um **Readiness Probe** que marque o contêiner como problemático, assim removendo-o do pool de balanceamento e garantindo que ele não receberá novas conexões.

Também é possivel implementar um **Liveness Probe** que finaliza o contêiner que não se recupere sozinho após determinado tempo.

## Conclusão 

Essas são apenas algumas das funcionalidades do Kubernetes relevantes neste contexto de operações descritos no histórico, o suficiente para ser considerado como uma alternativa exepcional para servir como a base da infraestrutura para muitos projetos.

Porém, não se deixe seduzir pelas promessas de funcionalidades incríveis - tudo isso vem com um custo associado, que é a relativa complexidade de suas partes, bem como a especial dificuldade que é implantar fora de provedores de Clouds públicos. 

Por exemplo, Kubernetes não implementa qualquer recurso para tratar monitoração e logs: paralelamente ã sua implantação, você precisará acoplar soluções que atendam aos seus requisitos.

Kubernetes resolve muitos dos problemas de infraestrutura que empresas com maturidade no processo de desenvolvimento de software normalmente enfrentam. Se este processo for ruim, o Kubernetes possivelmente fará pouco para contribuir com a organização. Em especial cito situações e arquiteturas que possam inviabilizar completamente o uso do Kubernetes por sua postura de tratar as instâncias de maneira "descartável" - nem todo software lida bem com essa característica, em especial os antigos. 

Muitos softwares não foram desenvolvidos para escalar horizontalmente ou lidar adequadamente com concorrência; embora possa ser uso neste caso de uso, dificilmente justifica a adoção do Kubernetes.

A grande dificuldade que uma empresa pode ter frente ao uso do Kubernetes é decidir se seu uso é vantajoso ou não. E, infelizmente, sem o apoio de especialistas, é bastante difícil prever certas dificuldades adicionais que podem inviabilizar o uso antes de uma implantação.

# "Bibliografia"

## Historia

* https://en.wikipedia.org/wiki/Cgroups
* https://en.wikipedia.org/wiki/Namespace
* https://www.section.io/engineering-education/history-of-container-technology/
* https://d2iq.com/blog/brief-history-containers
* https://docs.openshift.com/enterprise/3.0/whats_new/carts_vs_images.html

## Documentação do Kubernetes

* https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase
* https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
* https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes
* https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/


## Projetos 

* https://github.com/google/lmctfy: versão opensource da stack de containers do Google, posteriormente teve os esforços direcionados para a libcontainer.
* https://github.com/opencontainers/runc/tree/master/libcontainer: projeto original 








