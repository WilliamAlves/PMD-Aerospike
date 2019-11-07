# AEROSPIKE

## Introdução
O Aerospike é um banco de dados NoSQL  Key-Value distribuído, desenhado para aplicações que devem estar sempre disponíveis e lidar com big data de maneira confiável.
Desenvolvida em C, opera em três camadas: Uma camada de dados, uma camada de distribuição autogerenciada (o *"Aerospike Smart Cluster"*) e uma camada de cliente (o *"Aerospike Smart Client"*).
A camada de distribuição é replicada através de diferentes nós, para garantir consistência. Permitindo também que o banco de dados se mantenha operacional frente a falhas em nós individuais, ou a remoção de algum deles.
A camada de cliente é utilizada para manter controle sobre as configurações do *cluster* e a localização dos dados, e gerenciar o servidor.
A camada de dados é otimizada para armazenar os dados em discos rígidos e memórias flash, sendo os índices do banco armazenados em RAM para rápida disponibilidade, e os dados escritos escritos em blocos, para diminuir a latência.
Para o desenvolvimento deste documento e exemplos, utilizamos versão  4.7.0.2 do Aerospike.

### Instalação e configuração no Ubuntu 18.04+
Nesta parte do tutorial ensinaremos como fazer a instalação do Aerospike e configuração para Ubuntu 18.04 ou mais recente.

#### Aerospike Server
O primeiro passo deste tutorial é instalar o Aerospike Server e para isso começaremos com o *download*:
* No terminal do linux, cole o seguinte comando: **wget -O aerospike.tgz 'https://www.aerospike.com/download/server/latest/artifact/ubuntu18'**
* Para extrair o conteúdo da pasta compactada, entre na pasta onde realizou o download pelo terminal e cole o seguinte comando: **tar -xvf aerospike.tgz**
* O conteúdo será extraído para uma pasta de nome: aerospike-server-community-4.7.0.2-ubuntu18.04

Agora, para realizar a instalação:
* Entre na pasta criada com o seguinte comando: **cd aerospike-server-community-4.7.0.2-ubuntu18.04**
* Para continuar a instalação é necessário possuir o Python instalado, com um número de versão maior que 2.7 e menor que 3. Com o Python instalado, execute com o comando: **sudo ./asinstall**

Agora tudo deve estar instalado corretamente, para conferir isso utilize os seguintes comandos:
* Para inicializar o banco utilize: **sudo service aerospike start**
* Para se certificar que o banco está rodando utilize: **sudo service aerospike status**

#### Instalação do AMC
O AMC (*Aerospike Management Console*) é uma ferramenta para manipular e monitorar *cluster* de maneira rápida, com updates automáticos dos status dos *clusters*.
Para a instalação é necessário se certificar de que possui os seguintes pacotes:
* python (2.6+)
* gcc
* python-dev

Assim que todas as dependências forem instaladas, para baixar o AMC você deverá entrar no seguinte link: https://www.aerospike.com/download/amc/4.0.27/ e selecionar o pacote para Ubuntu 12.04+.
Quando o *download* do arquivo terminar é necessário fazer a instalação com o seguinte comando no terminal: **sudo dpkg -i aerospike-amc-community-4.0.27\_amd64.deb**
Assim que o ACM for instalado os comandos básicos para utilizá-lo são:
*  Inicialização: **sudo /etc/init.d/amc start**
*  Parada: **sudo /etc/init.d/amc stop**
*  Verificar status: **sudo systemctl status amc**

### Instalação e configuração no MAC OS X
Nesta parte do tutorial ensinaremos como fazer a instalação do Aerospike e configuração para MAC OS X.

#### Aerospike Server e AMC  
No MAC OS, o Aerospike funciona dentro de um ambiente virtual, por isso, é necessária a instalação do Vagrant (ambiente de desenvolvimento virtual) e do VMWare (máquina virtual).
* Faça o download e instale o Vagrant : **https://www.vagrantup.com/downloads.html**
* Faça o download e instale o VMWare: **https://www.virtualbox.org/wiki/Downloads**

 Agora, já podemos começar a instalação do Aerospike:
 
* Crie e acesse o diretório de trabalho do Aerospike: **mkdir ~/aerospike-vm && cd ~/aerospike-vm**
* Inicialize a máquina virtual do Aerospike: **vagrant init aerospike/aerospike-ce**
* Inicialize o Vagrant:
    * vagrant up
    * vagrant ssh
    * sudo service aerospike start
    * sudo service amc start

### Como interagir com o sistema?
Conforme citado acima, uma das maneiras de se interagir com o Aerospike é através do [AMC](https://www.aerospike.com/docs/amc/) (*Aerospike Management Console*), que fornece uma interface gráfica onde é possível analizar os nós do cluster e algumas métrias.


### Comandos Básicos: Aerospike Query Language (AQL)
O Aerospike possui sua própria linguagem para realizar manipulações no banco e efetuar pesquisas. 

## Arquitetura
O Aerospike utiliza o *Shared-Nothing* (SN), arquitetura de computação distribuída onde cada requisição é satisfeita por um único nó, que armazena e é o responsável (mestre) de uma parte do total de dados. Essa arquitetura cria um sistema sem um ponto único de falha, e permite a escalabilidade horizontal. Para aumentar a disponibilidade e a  confiabilidade, o Aerospike também replica os dados em diferentes nós.

### Terminologia
A seguir, algumas terminologias usadas em Aerospike que se referem a termos similares de RDBMS:

Aerospike   | RDBMS
--------- | ------
namespace | tablespace
record    | row
set       | table
bin       | column
index     | index


### Distribuição
No Aerospike, um *namespace* é uma coleção de dados que segue uma "política" comum, como o número de réplicas, a maneira como o dado é armazenado, e quando ele expira. É o equivalente deste sistema aos *databases* dos sistemas relacionais.
Cada *namespace* é dividido em 4096 partições lógicas, que são divididas igualmente entre os n nós do *cluster*. Ou seja, quando não há réplicas dos dados (fator de replicação igual a 1), cada nó armazena aproximadamente 
1/n dos dados.
Essa divisão se dá da seguinte maneira: a chave de cada registro (independente do tamanho) é transformado em uma *hashtag* de 20 *bytes*, utilizando o RIPEMD160 (uma função hash de distribuição aleatória). Utilizando os 12 primeiros *bits* da *hashtag*, o ID da partição deste registro é determinado. Graças às características desta função, as partições terão uma distribuição normal dentro dos diferentes nós do cluster, não havendo necessidade de *sharding* manual.
Quando nós são adicionados ou removidos do *cluster*, um novo *cluster* se formará e seus nós se coordenarão para dividir as partições entre eles. Então, o *cluster* automaticamente se rebalanceará.
Como os dados se distribuem igualmente (e aleatoriamente) nos nós, não existem *hot spots* ou gargalos onde aconteçam mais requisições de dados que em outros nós.

### Replicação
O Aerospike replica as partições em um ou mais nós. Um nó se torna o mestre, para escritas e leituras, para uma partição, enquanto outros nós guardam uma réplica dessa partição.
O fator de replicação é configurável; no entanto, ele não pode exceder o número de nós existentes. Mais réplicas significa maior confiabilidade dos dados, mas aumenta a demanda do cluster, já que as escritas tem de ser propagadas para todas as réplicas. Por padrão (e na maior parte dos casos), é utilizado fator de replicação 2 (o mestre e uma réplica).
Para o caso de fator de replicação 2, por exemplo, além das 1/n partições da qual é mestre, cada nó também terá réplica de 1/n dos dados do *cluster*.

| ![Replicação](https://aerospike.com/docs/architecture/assets/ARCH_shared_nothing_small.png) | 
|:--:| 
| *Replicação fator 2. Dois nós em um cluster que são master e réplica.* |


## Implementação de propriedades
 Conforme o teorema CAP, o Aerospike, até a versão 3.0, é um banco de dados AP - isto é, oferece disponibilidade ao invés de consistência, em casos de partição de rede. O Aerospike 3.0 não fornece diversas funcionalidades que são necessárias para consistência de replicação durante uma transação. Em vez disso, permite que o dado esteja disponível e aceitando escritas - criando, eventualmente, conflitos de escrita. O processo de escolher uma versão do dado para ser persistida, em caso de versões conflitantes, pode resultar em perda de dado.

À partir da versão 4.0, o Aerospike suporta tanto o modo AP (Disponível e Tolerante à partição), quanto o modo CP (Consistente e Tolerante à partição). Sendo o modo configurado através das políticas do “namespace”.

### Transações
Como citado anteriormente, até sua versão 3.0, era suportado apenas o modo AP (disponibilidade sobre consistência). Com isso, em caso de partição de rede, qualquer partição se colocaria como dono de todos os dados (que estivessem disponíveis nos nós do "sub-cluster"). Quando o problema da partição fosse resolvido, conflitos aconteceriam, causando perda de dados.
A partir do Aerospike 4, a "consistência forte" foi introduzida. Com esse algoritmo, as partes do cluster e as partições gerenciam cuidadosamente quais partes do cluster ainda estão disponíveis, não permitindo qualquer potencial conflito de escrita.
Ele passou a manter, também, consistência com base em chave-primária, promovendo durabilidade ao *commitar* a escrita para múltiplos servidores físicos, com componentes de *hardware* independentes.
Para durabilidade ainda maior, pode ser necessário que uma escrita seja feita em memória persistente.
No entanto, ser *ACID* também implica em outras duas características: isolamento e consistência *Multi-record* e isolamento de consulta, para que uma consulta *multi-record* seja consistente em relação à escritas. O Aerospike não provê essas duas caracteristicas.

### Consistência
O Aerospike oferece consistência forte para transações *single-record*, e todas as operações podem ser executadas de modo linear. Em adição, o Aerospike permite configuração para relaxar a consistência, em favor de maior performance e disponibilidade, conforme necessário.
A consistência forte garante que todas as escritas para um determinado arquivo irão ser aplicadas em uma ordem específica (sequencialmente), e escritas não serão reordenadas ou puladas (ou seja, perdidas).
Em particular, escritas que são reconhecidas como enviadas (*committed*) foram aplicadas e existem na linha do tempo, em contraste com outras escritas ao mesmo arquivo. Essa garantia se mantém mesmo em face à falhas na rede e partições do cluster. Escritas marcadas como *"timeouts"* ou *"InDoubt"* podem não ser aplicadas.
A consistência forte do Aerospike é *per-record*, e não envolve transações *multi-record*. Cada escrita ou atualização dentro de um mesmo arquivo será atômica e isolada, e a ordenação é garantida ao se utilizar relógio híbrido.
São oferecidos, também, tanto o modo *Full Linearizable*, que provê uma única *view* dentre todos os clientes que podem ler dados, como o modo de *Consistência de Sessão*, que garante que um processo individual veja a sequencial de atualizações. Qual modo será utilizado pelo cluster é configurável.
Na replicação síncrona, o dado é escrito no mestre e nas réplicas “simultaneamente”. Desta maneira, todas as cópias dos dados estão sempre sincronizadas. Isto é alcançado ao se esperar que todos os nós com uma cópia de um dado X processem uma transação feita sobre este dado, enviando a confirmação da transação ao usuário apenas após a confirmação de todas as cópias. Na replicação assíncrona, as réplicas geralmente ficam algumas transações para trás do mestre.
Em um exemplo, onde um usuário efetua uma transação sobre um dado com o mestre no nó A e réplica no nó B, o que ocorre é:
* O nó A guarda a transação em cache e imediatamente envia a atualização à cópia do dado no nó B.
* A cópia em B envia uma mensagem de confirmação de volta ao nó A.
* O nó A envia a confirmação de escrita ao usuário.

Enquanto na replicação assíncrona, uma das formas de se relaxar a consistência do cluster em troca de disponibilidade. O que seria feito seria:
* O nó A escreve a atualização e envia a confirmação ao usuário.
* O nó A envia a atualização do dado à cópia no nó B.
* O nó B, apenas então, envia uma mensagem de confirmação de escrita ao nó A.

Quando o banco está se recuperando de um reparticionamento, podem haver escritas perdidas, na replicação assíncrona.


### Disponibilidade
Como o sistema é configurado como AP, do teorema CAP, a disponibilidade é um dos fatores mais importantes no banco de dados, perdendo um pouco da consistência. O Aerospike diz ter disponibilidade perfeita, sendo que sempre que um nó cai, outros vão estar disponíveis com a informação necessária para ser utilizada.
O sistema implementa isso utilizando o algoritmo de *Smart Partitions*, onde quando um nó de uma partição para de funcionar, as requisições que este nó aceitaria são distribuídas aleatoriamente entre os outros nós da partição. Isso é um diferencial do Aerospike entre os outros bancos de dados, que neste caso iriam alocar todas as requisições para um único ou poucos outros nós, causando um possível overflow de requisições.
Além disso, vale ressaltar algumas tecnologias utilizadas pelo Aerospike para garantir o máximo de tempo no ar, que são:
* O sistema de monitoramento de *cluster* dinâmico, que consiste em automaticamente detectar nós entrando e saindo do *cluster*, e assim que essa movimentação ocorre, os dados são re-replicados. O sistema distribui as responsabilidades entre os nós principais e os nós réplica igualmente, garantindo a disponibilidade mencionada acima.
* O *Aerospike Smart Client*, que serve para monitorar o *cluster* durante sua adaptação em uma nova rede. Ele guarda informações como a localização mais recente dos dados, além de reconfigurar as conexões do cliente automaticamente e inicializá-las, independentemente do hardware. 
*  A replicação geográfica, que se preocupa em encontrar *datacenters* mais adequados para hospedar os dados com uma distribuição favorável para a velocidade de conexão.

Além de tudo isso, os *updates* do *software* são realizados de maneira que não ocorram problemas de compatibilidade entre versões. Assim não é necessário um *downtime* para que ocorra a atualização.

### Escalabilidade
Devido a arquitetura *shared-nothing*, basta apenas adicionar novos nós ao cluster. Ela permite escalabilidade horizontal em todas as camadas da aplicação. Esse é um dos maiores diferenciais de Aerospike sobre outros bancos, que não se preocupam tanto com a escalabilidade, e quando chegam em seu limite causam danos aos clientes de seus serviços, e no fim a equipe desenvolvedora vai gastar mais tempo e dinheiro para atualizar a aplicação. E ainda assim é só uma questão de tempo até o mesmo problema acontecer e o processo se iniciar de novo.

### Quando usar?
O Aerospike foi desenhado para aplicações que devem estar sempre disponíveis e lidar com big data de maneira rápida e confiável. A seguir veremos alguns [casos de uso] e exemplos de empresas que o utilizam no dia a dia de suas operações.

* Substituição de cache
    Latência baixa e alto rendimento são características do Aerospike e fazem com que ele seja um ótimo candidato para substitui o cache. Quando existe um grande número de dados dinâmicos é necessária a decisão entre guardar dados em cache, e assim abrir espaço para eventuais inconsistências de dados, ou fazer muitas consultas lentas. O Aerospike funciona com o sistema chave-valor que possui grande foco na rapidez e simplicidade, com caminho mais direto para o disco ou memória na busca dos dados, e por isso o seu uso pode substituir o uso de cache. Empresas como a KAYAK, uma ferramenta de busca especializada em viagens, utilizam o Aerospike desta forma em suas buscas.
* Armazenamento de perfil de usuários
    O armazenamento de perfis de usuário é muito importante em alguns tipos de sistemas e muitas vezes os dados são variados e de tamanho pequeno. O Aerospike funciona muito bem nessas condições e por conta disso possui muitos clientes do ramo de publicidade, como as empresas AppNexus e Nielsen.
* Mecanismo de recomendação
    Mecanismos de recomendação exigem uma camada de dados rápida, para suportar muitas requisições sem grande impacto para o usuário, e flexível, já que a base tende a crescer rapidamente. O Aerospike funciona muito bem neste contexto e a utilização dele no mercado acontece em diversas empresas de publicidade e e-commerces. 0 Aerospike possui até mesmo um modelo open source de criação de um mecanismo de recomendação usando suas ferramentas, RESTful Web Service e Spring Boot.

* [Teste do link] 

[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen)
[Teste do link]: <https://twitter.com/littleis13>
[casos de uso]: <https://www.aerospike.com/solutions/technology/use-cases/>

