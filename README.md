# Descomplicando o Docker - LinuxTips

## Iniciando com Docker
### Instalação do Docker
<https://docs.docker.com/install>

### Melhores práticas para o Dockerfile
<https://docs.docker.com/develop/develop-images/dockerfile_best-practices/>

### Entenda recursos do kernel como namespace e cgroups
<http://man7.org/linux/man-pages/man7/namespaces.7.html><br>
<https://en.wikipedia.org/wiki/Cgroups>

### Comandos básicos
``` sh
docker container run -d IMAGENAME
docker container attach CONTAINERID
docker container exec -ti CONTAINERID <comando>

docker container stop CONTAINERID
docker container rm CONTAINERID
docker container start CONTAINERID
docker container restart CONTAINERID
docker container pause CONTAINERID
docker container unpause CONTAINERID

docker container inspect CONTAINERID
docker container logs -f CONTAINERID

docker image inspect IMAGEMID
docker image history IMAGEMID
```

### Uso de Recursos

* Informações de CPU, MEM, IO rede, IO de bloco
``` sh
docker container stat CONTAINERID
```

* TOP igual ao do Linux, mostra os processo
``` sh
docker container top CONTAINERID
```

* Limitando Memória e CPU
_A CPU é especificada em percent, logo o range de valor vai de 0.01 até 1.00
O valor acima "0.25" está limitando o uso do container em até 25% de CPU._
``` sh
docker container run -d -m 128M IMAGENAME
docker container run -d --cpu 0.25 -m 256M IMAGENAME

docker container update --cpus 0.5 CONTAINERID
docker container update --cpus 1 --memory 64M CONTAINERID
```

### Dockerfile
*Exemplo*
``` yaml
FROM debian

LABEL app="Certificado"
ENV TREINAMENTO="Docker"

RUN apt-get update && apt-get install -y stress && apt-get clean

CMD stress --cpu 1 --vm-bytes 64M --vm 1
```

*Build*
``` sh
docker image build -t IMAGENAME:<imageversion> .
```

## Volumes
### Tipo BIND
* Montar um diretório específico no container
``` sh
docker container run -ti --mount type=bind,src=/diretorio/local,dst=/meuVolume IMAGENAME
docker container run -ti --mount type=bind,src=/diretorio/local,dst=/meuVolume,ro IMAGENAME
```

> Se você colocar um Volume no dockerfile e não especifícá-lo no "docker run", o Docker irá criar um volume com um nome aleatório. Isso é ruim para a gestão de volumes que o administrador deverá ser responsável.

### Tipo VOLUME
``` sh
docker volume create MEUVOLUME
docker volume ls
docker volume inspect MEUVOLUME
docker volume rm MEUVOLUME
docker volume prune
```

> Os dados do volume serão armazenados no diretório */var/lib/docker/volumes/meuVolume/_data*

* Montar o volume no container
``` sh
docker container run -ti --mount type=volume,src=MEUVOLUME,dst=/meuVolume IMAGENAME
```

## Dockerfile
### Exemplo de Dockerfile
``` yaml
FROM debian

RUN apt-get update && apt-get install -y apache2 && apt-get clean

ENV APACHE_LOCK_DIR="/var/lock"
ENV APACHE_PID_FILE="/var/run/apache2.pid"
ENV APACHE_RUN_USER="www-data"
ENV APACHE_RUN_GROUP="www-data"
ENV APACHE_LOG_DIR="/var/log/apache2"

LABEL description="Webserver"

VOLUME /var/www/html
EXPOSE 80

ENTRYPOINT ["/usr/sbin/apachectl"]
CMD ["-D", "FOREGROUND"]
```

> ENTRYPOINT: Principal processo do container, é um INIT do container.<br>
> CMD: O CMD apenas passa parâmetros para o processo principal. No exemplo o ENTRYPOINT é o *apachectl*, então passamos o parâmetro para iniciar o apache em FOREGROUND.

### Constuindo a image a partir do Dockerfile
``` sh
docker image build -t meu-apache:1.0
docker image build -t meu-apache:1.0 -no-cache
```

### Outras instruções do Dockerfile
|COMANDO	|DESCRIÇÃO		|
|---------------|-----------------------|
|COPY   	|Copia arquivo para dentro do container durante o build|
|ADD    	|Copiar arquivo para dentro do container durante o build, porém pode especificar uma URL para download e também arquivos compactados serão descompactados no container|
|RUN   	 	|Executa comandos dentro do container|
|ENV    	|Cria uma variável de ambiente|
|USER   	|Especifica um usuário dentro do container para executar os processos|
|WORKDIR	|Diretório padrão do container|
|CMD		|Executa um comando, diferente do RUN que executa o comando no momento em que está "buildando" a imagem, o CMD executa no início da execução do container|
|LABEL		|Adiciona metadados a imagem como versão, descrição e fabricante|
|COPY		|Copia novos arquivos e diretórios e os adicionam ao filesystem do container|
|ENTRYPOINT	|Permite você configurar um container para rodar um executável, e quando esse executável for finalizado, o container também será|
|ENV		|Informa variáveis de ambiente ao container|
|EXPOSE		|Informa qual porta o container estará ouvindo|
|FROM		|Indica qual imagem será utilizada como base, ela precisa ser a primeira linha do Dockerfile|
|MAINTAINER	|Autor da imagem|
|RUN		|Executa qualquer comando em uma nova camada no topo da imagem e "commita" as alterações. Essas alterações você poderá utilizar nas próximas instruções de seu Dockerfile|
|USER		|Determina qual o usuário será utilizado na imagem. Por default é o root|
|VOLUME		|Permite a criação de um ponto de montagem no container|
|WORKDIR	|Responsável por mudar do diretório / (raiz) para o especificado nele|


### Argumentos (ARG)

``` yaml
FROM ubuntu
ARG CONT_IMG_VER
ENV CONT_IMG_VER v1.0.0
RUN echo $CONT_IMG_VER
```

Nesse exemplo o container quando executado irá printar a string v1.0.0.
Essa string pode ser alterada durante o build da imagem como mostrado abaixo:
```
docker build --build-arg CONT_IMG_VER=2.0.0 -f Dockerfile .
```

Agora quando o container dessa imagem for executado, será mostrada a string v2.0.0.

### HEALTHCHECK

``` yaml
HEALTCHECK --interval 5m --timeout=3s \
    CMD curl --retry-max-time 2 -f http://localhost/ || exit 1
```

Adicionando a linha acima em uma imagem com o Apache, o healtcheck checará a cada 5 minutos se a URL localhost está respondendo, caso ela falhe "||" então o container será encerrado com o código 1.
> Atenção: o comando utilizado para o healthcheck deve existir na imagem.

## Imagens
``` sh
docker image pull http
docker image ls
docker image insepect IMAGENAME
docker image prune --all --filter until=240h
docker image prune -a
docker image tag IMAGEID NOMEDAIMAGEM:VERSAOIMAGEM
```

### EDITANDO UMA IMAGEM "COMMIT"

``` sh
docker container run -it ubuntu
```

Altere a image conforme sua necessidade, saia do container com *Ctrl + P + Q* para que o mesmo continue em execução e commite as alterações na imagem:

``` sh
docker commit -m "Descrição da alteração" CONTAINERID
docker image tag <image_id> <NovoNome>:<NovaVersao>
```

> Prefira sempre realizar alterações via Dockerfile, pois com o Dockerfile é possível recriar a imagem novamente em caso de perda, e todas as alterações ficarão registradas.


## MultiStage
A utilização do multistage é utilizado para reduzir o tamanho da imagem final.
Por exemplo, para uma aplicação GO, precisaríamos utilizar a imagem GOLANG para realizar o build da imagem, essa imagem possui em torno de 800MB, logo minha imagem final teria os 800MB mais o tamanho da minha aplicação GO.
Para contornar isso, poderíamos utilizar a imagem GOLANG apenas para gerar o executável da aplicação GO e depois copia-lo para uma imagem menor como a imagem ALPINE.
OBS: A utilização do MULTISTAGE depende muito de como funciona sua aplicação, pois é necessário copiar os arquivos para a outra imagem, com exemplo o APACHE seria bem difícil de fazer.
Apenas como comparação, utilizando o multistage a imagem final ficou com 7MB e sem o multistage a imagem final ficou com 800MB.

Aplicação e Go que será usada na imagem
``` go
package main
import "fmt"

func main() {
        fmt.Println("Giropops Strigus Girus")
}
```

Imagem sem o MultiStage -> 700MB
``` yaml
FROM golang

WORKDIR /app
ADD . /app
RUN go build -o meugo

ENTRYPOINT ./meugo
```

Imagem usando o MultiStage -> 7MB
``` yaml
FROM golang AS buildando

WORKDIR /app
ADD . /app
RUN go build -o meugo

FROM alpine
WORKDIR /app
COPY --from=buildando /app/meugo /app

ENTRYPOINT ./meugo
```

## Dicas
- Remova do contêiner tudo que não será usado, lembrando que deve limpar na camada de escrita.
- Reduza o número de camadas da imagem.
- Utilize o MultiStage sempre que possível.
- Utilize o .Dockerignore.
- Adicione o non-root user, Utilize outro usuário para executar sua aplicação.
- As vezes o cache é problemático, pois se houve uma atualização na versão da aplicação que está em repositório APT, ela pode não ser atualizada por estar em cache, esse é um exemplo de problema ao usar o cache, porém ele é muito útil.
- Contêineres são imutáveis e efêmeros, sendo assim ele nasceu pra morrer, não confunda este com uma VM.
- As variáveis declaradas na instrução ENV do dockerfile pode ser utilizada dentro do próprio dockerfile.
- SHELL ["powershell", "-command"] essa instrução muda o shell do container.
- ARGS Argumento que posso utilizar durante o build da imagem.
- HEALTHCHECK, uma forma de verificar se o container está íntegro. 


## Container Registry (CR)
### Login
Login no Docker Hub
``` sh
docker login
```

Login em um CR que não seja o Docker Hub
``` sh
docker login URL-CR
```

### Upload de imagem
``` sh
docker push IMAGENAME:IMAGEVERSION
```

### Um container para um CR
``` sh
docker container run -d -p 5000:5000 --restart=always --name registry registry:2
```

As imagens do CR ficam armazenadas em:
```/var/lib/registry/docker/registry/v2/repositories/meu_apache```

### Verificando imagens de um CR
``` sh
curl localhost:5000/v2/_catalog
curl localhost:5000/v2/meu_apache/tags/list
```

## Docker-machine
*docker-machine* é utilizado criar uma VM remota para rodar o docker, pode ser utilizado para criar a VM na AWS, Digital Ocean, Virtual Box, Hyper-V entre outros.

Utilize o docker-machine para gerenciar um host remoto com o Docker

### Instalação
``` sh
curl -L https://github.com/docker/machine/releases/download/v0.16.1
/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine
chmod +x /tmp/docker-machine
sudo cp /tmp/docker-machine /usr/local/bin/docker-machine
```

### Criando um host para docker no VirtualBox e Hyper-V
``` sh
docker-machine create --driver virtualbox NOMEDAVM
docker-machine create --driver hyperv --hyperv-virtual-switch "DockerMachineNetwork" NOMEDAVM
```

### Comandos úteis do docker-machine
``` sh
docker-machine ls
docker-machine start NOMEDAVM
docker-machine stop NOMEDAVM
docker-machine ip NOMEDAVM
docker-machine ssh NOMEDAVM
docker-machine inspect NOMEDAVM
docker-machine status NOMEDAVM
docker-machine rm NOMEDAVM
```

### Conectando o Docker client local em um Host remoto docker.
``` sh
eval $(docker-machine env NOMEDAVM)
```
Feito isso, os comando *docker* serão executados no host remoto e não no local. Utilize para gerenciar hosts remotos com o docker da sua estação de trabalho.

Para voltar o cliente para o host local
``` sh
eval $(docker-machine env -u)
```

## SWARM
Swarm é um orquestrador de container oficial do Docker, é como um Kubernetes.

Gerenciamento de Container (Orquestrador)
- Worker: responsável apenas por carregar os container
- Manager: sabe todos os detalhes do cluster

Para um cluster saudável, precisamos ter no mínimo 51% de manager online.
pensando assim se temos 5 nós, devemos ter 3 managers, pois se houver apenas 2 managers e um deles cair, terei 50% nos manager online, com isso meu cluster vai pro saco.

Por que não usar todos os nós como managers então?
qdo temos um cluster swarm e um manager apresenta problema, ocorre um processo chamado de eleição para decidir qual dos managers será o principal novamente, logo quanto mais managers exisitir maior o tempo para um nó ser eleito e maior o tempo de downtime do ambiente.

### Antes precisa da seguinte liberação de portas TCP/UDP

 - 2377 TCP       Utilizada para o gerenciamento do Cluster
 - 7946 TCP/UDP   Comunicação entre os Nós do Cluster
 - 4789 UDP       Porta de comunicação entre a rede Overlay

### Iniciar um cluster
``` sh
docker swarm init --advertise-addr IpDaInterfaceDeRede

#    (command output)
#Swarm initialized: current node (taxeaef969y2bm5attuxtxxug) is now a manager.
#
#To add a worker to this swarm, run the following command:
#
#    docker swarm join --token SWMTKN-1-2b77msrwdciljb8vivzusy01sgf9x2kghkqdzucyostquj3o5e-b9swdjndhbqw9yj9vy5h6ec7s 172.29.241.39:2377
#
#To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

### Listar nós de um cluster
``` sh
docker node ls
```

### Adicionar um novo worker ao Cluster Swarm
1. No nó "manager" execute o comando abaixo para conseguir o token e o comando para adicionar o worker
``` sh
docker swarm join-token worker
```

2. Execute o comando o token informado
``` sh
docker swarm join --token SWMTKN-1-2b77msrwdciljb8vivzusy01sgf9x2kghkqdzucyostquj3o5e-b9swdjndhbqw9yj9vy5h6ec7s 172.29.241.39:2377
```

### Adicionar um novo manager ao Cluster Swarm
1. No nó "manager" execute o comando abaixo para conseguir o token e o comando para adicionar o manager
``` sh
docker swarm join-token manager
```

2. Execute o comando o token informado
``` sh
docker swarm join --token SWMTKN-1-2b77msrwdciljb8vivzusy01sgf9x2kghkqdzucyostquj3o5e-di7qyk01kdyig3ajww6nadr6m 172.29.241.39:2377
```

### Promover / Despromover um nó à Manager
``` sh
docker node promote HOSTNAME
docker node demote HOSTNAME
```

### Remover um nó do cluster apartir do manager
``` sh
docker swarm demote HOSTNAME
docker swarm rm -f HOSTNAME
```

### Remover pelo próprio nó
``` sh
docker swarm leave
docker swarm leave -f
```

> Recomenda-se alterar os tokens periodicamente por segurança.
``` sh
docker swarm join-token --rotate manager
docker swarm join-token --rotate worker
```

### Docker SWARM Node
Opção "Availability"
``` sh
docker node update --availability [ drain | pause | active ] NOMEDAVM
```
|OPÇÃO    |DESCRIÇÃO                                                                                                          |
|---------|-------------------------------------------------------------------------------------------------------------------|
|*Drain*  |O nó não receberá nenhum novo container e os container atuais do nó são desligados                                 |
|*Pause*  |Nenhum novo container é adicionado ao container,<br>porém os atuais containers do nó continuam rodando normalmente |
|*Active* |O nó recebe os containers normalmente                                                                              |

``` sh
docker node inspect NOMEDAVM
```

### Docker SWARM Service

- Swarm é a resiliência dos nós enquanto o Service é uma forma de ter resiliência nos containers.
- A comunicação interna dos container não são realizadas por IP e sim por nome de serviço, ou seja, o nome do serviço responde como um VIP ou DNS.
- SERVICE é um conjunto de container respondendo para o mesmo serviço, como se fosse um único servidor.
- Quando se cria um service, informa a quantidade de replicas (tasks) desejadas, isso é a quantidade de container que será criado.
- O Service faz o balanceamento de carga no estilo Round Robin.
- Quando uma porta é publicada para um container, mesmo que houver apenas uma réplica do container no cluster, esta porta estará publicada em todos os nós do cluster e todos eles redirecionaram para o container, ou seja, todos os nós sabem onde os container estão sendo executados, logo é possível se conectar ao serviço utilizando qualquer IP de um dos nós do cluster.

#### Exemplo de uso:
``` sh
docker service create --name <service_name> --replicas 3 -p 8080:80 nginx

docker service create --name <service_name> --hostname <container_hostname> -- limit-cpu 0.25 --limit-memory 64M --replicas 3 -p 8080:80 nginx

docker service create --name <service_name> --env VARIAVEL=VALOR --replicas 3 -p 8080:80 nginx

docker service create --dns 8.8.8.8 --name <service_name> --replicas 3 -p 8080:80 nginx

docker service create --network rededev --name <service_name> --replicas 3 -p 8080:80 nginx
```

#### Inspecionar
``` sh
docker service inspect <service_name>
docker service inspect <service_name> --pretty
```

#### Verificar onde os container estão sendo executados
``` sh
docker service ps <service_name>
```

#### Verificar o logs de todos os containers:
``` sh
docker service logs -f <service_name>
```

#### Escalar containers
``` sh
docker service scale giropops=9
```

#### Remover o serviço
``` sh
docker service rm <service_name>
```

#### Cuidado com volumes

O *volume* criado em um Nó do cluster não é compartilhado com os outros nós.
Desta forma o serviço do nginx criado no comando abaixo teria problemas de sincronismo de dados.
``` sh
docker volume create webindex
docker service create --name webserver --replicas 3 --publish 8080:80 --mount type=volume,src=webindex,dst=/usr/share/nginx/html nginx
```

Este volume será criado em todos os nós porém não haverá replicação dos dados, ou seja, alterando o index.html de um dos nós, a alteração estará apenas naquele nó do cluster

Para  utilizar o volume em todos os nós é necessário utilizar plugins como o *Flocker, GlusterFS, Ceph ou NFS*.
Já indo para o mundo de Cloud, a AWS e Azure com Cloud Store fornecem soluções melhores para esse compartilhamento.

Algumas fontes de consulta:
<https://github.com/ClusterHQ/flocker>
<https://www.gluster.org/>
<https://www.ibm.com/developerworks/br/linux/library/l-ceph/index.html>

<https://www.mundodocker.com.br/persistindo-dados-flocker/>
<https://stato.blog.br/wordpress/replicacao-de-dados-com-glusterfs/>


## Secrets
Quando está usando um cluster Swarm, é possível utilizar o recurso de secret do docker.

### Criar Secret
- Por Arquivo
``` sh
echo -n "MySecret" > ~/secretFile.txt
docker secret create SecretName SecretFile
```

- Direto pelo comando echo
``` sh
echo -n "MySecret" | docker secret create SecretName -
```

- As Secret ficam dentro de cada container no arquivo listado abaixo como texto puro, assim pode utilizá-la como achar melhor
  - /run/secrets/SecretName

> Importante colocar a opção "-n" no echo para evitar a quebra de linha.

### Verificando e inspecionando secrets disponíveis
``` sh
docker secret ls
docker secret inspect Secretname
```

### Removendo uma secret
``` sh
docker secret rm SecretName
```

### Subindo um serviço com uma secret
``` sh
docker service create --name webserver -p 8080:80 --replicas 10 --secret SecretName nginx
```

### Definindo permissões para o arquivo de secret dentro do container e usando source e target para definir o nome do secret dentro do container
``` sh
docker service create --name nginx2 -p 8080:80 --secret src=SecretName,target=meu-secret,uid=200,gid=200,mode=0400 nginx
```

### Adicionando uma secret a serviço já existente
``` sh
docker service update --secret-add SecretName ServiceName
```

### Removendo uma secret de um serviço existente
``` sh
docker service update --secret-rm SecretName ServiceName
```

### Dica do especialista Jeferson Noronha
>Ele realiza um mix entre o vault do Ansible que é encriptado com o Secret do Docker, assim não expoẽ a senha ou a informação em nenhum lugar.

## Compose, Stack e Services
[Instalando o docker-compose]<https://docs.docker.com/compose/install/>
[Overview do docker-compose]<https://docs.docker.com/compose/>

_Compose é um ferramenta para definir aplicações Docker multi-container. Com o Compose, você usar um arquio YAML para configurar seus serviços. Então, com um simples comando, você cria e inicia todos os serviços dos seu arquivo de configuração._

Services = conjunto de _container_
Stack = conjunto de _services_

A diferença é que quando eu crio um service por linha de comando, eu posso ter vários container para um mesmo serviço, um exemplo, criar um service com 5 réplicas de nginx.
Já quando eu uso o docker-compose e o stack, eu posso definir em um único arquivos, vários serviços e suas réplicas, e apenas usando o _docker stack_ eu consigo subir todos os serviços de uma única vez.

### docker-compose file [Primeiro Exemplo]
``` yaml
version: "3.5"
services:
  web:
    image: nginx
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
    - "8080:80"
    networks:
    - webserver
networks:
  webserver:
```

### Deploy de um docker-compose em um cluster swarm
``` sh
docker stack deploy -c docker-compose.yaml STACKNAME
```

### Análise da STACK
```
docker network ls
docker network inspect STACKNAME_NETWORKNAME
docker service ls
docker service inspect STACKNAME_SERVICENAME
docker stack ls
docker stack ps STACKNAME
docker stack services STACKNAME
```

### Scale
``` sh
docker service scale STACKNAME_SERVICENAME=10
```

### Remover a STACK
``` sh
docker stack rm STACKNAME
```

### docker-compose file [Segundo Exemplo]
``` yaml
version: '3'
services:
  db:
    image: mysql:5.7
    volumes:
    - db_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress

  wordpress:
    depends_on:
    - db
    image: wordpress:latest
    ports:
    - "8080:80"
    environment: 
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress

volumes:
  db_data:
```

### docker-compose file [Terceiro Exemplo]
``` yaml
version: "3.5"
services:
  web:
    image: nginx
    deploy:
      placement:
        constraints:
        - node.labels.dc == UK
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
    - "8080:80"
    networks:
    - webserver
         
  visualizer:
    image: dockersamples/visualizer:stable 
    ports:  
    - "8888:8080"
    volumes:
    - "/var/run/docker.sock:/var/run/docker.sock"
    deploy: 
      placement:
        constraints: [node.role == manager]
    networks:
    - webserver
            
networks:
  webserver:
```

> Neste terceiro exemplo, usamos o *_placement: constraints_*, onde indicamos que os containers do serviço *WEB* só podem subir em nodes onde há uma label chamada *DC* e que o seu valor seja *UK*, assim como o serviço *VISUALIZER* irá verificar pela role *MANAGER*.

* Visualizer:* Subirá apenas nos nodes magagers.
* Web:* Subirá apenas em nodes onde exista uma TAG dc=UK.

### Adicionando uma TAG aos nodes
``` sh
docker node update --label-add dc=UK elliot-01
```

### docker-compose file [Quarto Exemplo]
<https://github.com/dockersamples/example-voting-app/blob/master/docker-stack.yml>

``` yaml
version: "3.7"
services:

  redis:
    image: redis:alpine
    networks:
      - frontend
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
        delay: 10s
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 10s
        failure_action: continue
        monitor: 60s
        order: stop-first
      restart_policy:
        condition: on-failure
  db:
    image: postgres:9.4
    environment:
      POSTGRES_USER: "postgres"
      POSTGRES_PASSWORD: "postgres"
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    deploy:
      placement:
        constraints: [node.role == manager]
  vote:
    image: dockersamples/examplevotingapp_vote:before
    ports:
      - 5000:80
    networks:
      - frontend
    depends_on:
      - redis
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
      restart_policy:
        condition: on-failure
  result:
    image: dockersamples/examplevotingapp_result:before
    ports:
      - 5001:80
    networks:
      - backend
    depends_on:
      - db
    deploy:
      replicas: 1
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure

  worker:
    image: dockersamples/examplevotingapp_worker
    networks:
      - frontend
      - backend
    depends_on:
      - db
      - redis
    deploy:
      mode: replicated
      replicas: 1
      labels: [APP=VOTING]
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
      placement:
        constraints: [node.role == manager]

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    stop_grace_period: 1m30s
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]

networks:
  frontend:
  backend:

volumes: 
  db-data:
```

#### O que temos de novo no docker-compose acima:
``` yaml
...
  deploy:
    mode: replicated | global
    replicas: 2
    update_config:
      parallelism: 2
      delay: 10s
      order: start-first
```

|CHAVE|DESCRIÇÃO|
|-----|---------|
|deploy: replicas|Quantidade de containers que a stack irá subir|
|deploy: mode: replicated|Será criado em toda a stack a quantidade de container específicado na opção *replicas*|
|mode: global|Haverá um container por NODE do Swarm, caso um novo node seja adicionado ao cluster, um novo container também será criado neste node. Opção muito utilizada para containers de monitoramento do host|
|updateconfig: parallelism:|Em caso de um update, esta opção indicará quantos containers por vez serão afetados. Neste caso temos duas replicas e elas seriam afetadas ao mesmo tempo, causando indisponibilidade, o melhor seria deixar paralleslism como 1, assim apenas um container por vez seria atualizado.|
| update_config: delay|Considerando o parallelism, o delay indica o tempo entre executar o update no próximo grupo de container|
|update_config: order|valores que podemos usar é *start-first* e *stop_first*. Assim você decide se quer primeiro parar o que está rolando e iniciar os novos containers ou se quer primeiro iniciar os novos container e depois para os antigos|


``` yaml
...
  deploy:
    rollback_config:
      parallelism: 1
      delay: 10s
      failure_action: continue
      monitor: 60s
      order: stop-first
```

|CHAVE|DESCRIÇÃO|
|-----|---------|
|rollback_config|Possui as mesmas opções do *update_config*, porém é claro, para fazer o rollback|
|failure_action: pause|Em caso de falha no rollback de algum container, o sistema irá pause o processo de rollback para que seja analisado o problema|
|failure_action: continue|Em caso de falha no rollback de algum container, o sistema irá continuar executando o processo de rollback, de qualquer forma deve ser analisado o motivo da falha|
|monitor|Tempo que o sistema irá monitorar pra saber se o rollback foi realizado com sucesso|

### docker-compose file [Quinto Exemplo]

``` yaml
version: "3.7"
services:
  prometheus:
    image: linuxtips/prometheus_alpine
    volumes:
      - ./conf/prometheus/:/etc/prometheus/
      - prometheus_data:/var/lib/prometheus
    networks: 
      - backend
    ports:
      - 9090:9090
  node-exporter:
    image: linuxtips/node-exporter_alpine
    hostname: '{{.Node.ID}}'
    volumes:
      - /proc:/usr/proc
      - /sys:/usr/sys
      -/:/rootfs
    deploy:
      mode: global
    networks: 
      - backend
    ports:
      - 9100:9093
  alertmanager:
    image: linuxtips/alertmanager_alpine
    volumes:
      - ./conf/alertmanager/:/etc/alertmanager/
    networks: 
      - backend
    ports:
      - 9093:9093
  cadvisor:
    image: google/cadvisor
    hostname: '{{.Node.ID}}'
    volumes: 
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - backend
    deploy:
      mode: global
    ports:
      - 8080:8080
  grafana:
    image: nopp/grafana_alpine
    depends_on:
      - prometheus
    volumes:
      - ./conf/grafana/grafana.db:/grafana/data/grafana.db
    env_file:
      - grafana.config
    networks:
      - backend
      - frontend
    ports:
      - 3000:3000
networks:
  backend:
  frontend:
volumes:
  prometheus_data:
  grafana_data:
```

```
# docker stack deploy -c docker-compose.yml giropops
# docker service ls 
# docker stack ls

Prometheus:
http://SEU_IP:9090

AlertManager:
http://SEU_IP:9093

Grafana:
http://SEU_IP:3000

Node_Exporter:
http://SEU_IP:9100

Rocket.Chat:
http://SEU_IP:3080

cAdivisor:
http://SEU_IP:8080

# docker stack rm giropops


Lembrando, para conhecer mais sobre o giropops-monitoring acesse o repositório no GitHub e assista a série de vídeos em que  falo detalhadamente como montei essa solução:
Repo: https://github.com/badtuxx/giropops-monitoring
Vídeos: https://www.youtube.com/playlist?list=PLf-O3X2-mxDls9uH8gyCQTnyXNMe10iml
```