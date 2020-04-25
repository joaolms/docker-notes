# Descomplicando o Docker - LinuxTips

## Docker Básico
### Instalação do Docker
<https://docs.docker.com/install>

### Comandos básicos
``` sh
docker container run -d nginx
docker container attach <container_id>
docker container exec -ti <container_id> <comando>

docker container stop <container_id>
docker container rm <container_id>
docker container start <container_id>
docker container restart <container_id>
docker container pause <container_id>
docker container unpause <container_id>

docker container inspect <container_id>
docker container logs -f <container_id>

docker image inspect IMAGEMID
docker image history IMAGEMID
```

### Uso de Recursos

* Informações de CPU, MEM, IO rede, IO de bloco
``` sh
docker container stat <container_id>
```

* TOP igual ao do Linux, mostra os processo
``` sh
docker container top <container_id>
```

* Limitando Memória e CPU
_A CPU é especificada em percent, logo o range de valor vai de 0.01 até 1.00
O valor acima "0.25" está limitando o uso do container em até 25% de CPU._
``` sh
docker container run -d -m 128M IMAGENAME
docker container run -d --cpu 0.25 -m 256M IMAGENAME

docker container update --cpus 0.5 <container_id>
docker container update --cpus 1 --memory 64M <container_id>
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
|COMANDO|DESCRIÇÃO|
|-------|---------|
|COPY   |Copia arquivo para dentro do container durante o build|
|ADD    |Copiar arquivo para dentro do container durante o build, porém pode especificar uma URL para download e também arquivos compactados serão descompactados no container|
|RUN    |Executa comandos dentro do container|
|ENV    |Cria uma variável de ambiente|
|USER   |Especifica um usuário dentro do container para executar os processos|
|WORKDIR|Diretório padrão do container|
|CMD	|Executa um comando, diferente do RUN que executa o comando no momento em que está "buildando" a imagem, o CMD executa no início da execução do container|
|LABEL	|Adiciona metadados a imagem como versão, descrição e fabricante|
|COPY	|Copia novos arquivos e diretórios e os adicionam ao filesystem do container|
|ENTRYPOINT|Permite você configurar um container para rodar um executável, e quando esse executável for finalizado, o container também será|
|ENV	|Informa variáveis de ambiente ao container|
|EXPOSE	|Informa qual porta o container estará ouvindo|
|FROM	|Indica qual imagem será utilizada como base, ela precisa ser a primeira linha do Dockerfile|
|MAINTAINER|Autor da imagem|
|RUN	|Executa qualquer comando em uma nova camada no topo da imagem e "commita" as alterações. Essas alterações você poderá utilizar nas próximas instruções de seu Dockerfile|
|USER	|Determina qual o usuário será utilizado na imagem. Por default é o root|
|VOLUME	|Permite a criação de um ponto de montagem no container|
|WORKDIR|Responsável por mudar do diretório / (raiz) para o especificado nele|

## Imagens
``` sh
docker image ls
docker image insepect IMAGENAME
docker image prune --all --filter until=240h
docker image prune -a
```

## MultiStage
A utilização do multistage é utilizado para reduzir o tamanho da imagem final.
Por exemplo, para uma aplicação GO, precisaríamos utilizar a imagem GOLANG para realizar o build da imagem, essa imagem possui em torno de 800MB, logo minha imagem final teria os 800MB mais o tamanho da minha aplicação GO.
Para contornar isso, poderíamos utilizar a imagem GOLANG apenas para gerar o executável da aplicação GO e depois copia-lo para uma imagem menor como a imagem ALPINE.
OBS: A utilização do MULTISTAGE depende muito de como funciona sua aplicação, pois é necessário copiar os arquivos para a outra imagem, com exemplo o APACHE seria bem difícil de fazer.
Apenas como comparação, utilizando o multistage a imagem final ficou com 7MB e sem o multistage a imagem final ficou com 800MB.

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
- Remova do contêiner tudo que não será usado, lembrando que deve limpar na camada de escrita
- Reduza o número de camadas da imagem
- Utilize o .Dockerignore
- Adicione o non-root user, Utilize outro usuário para executar sua aplicação
- As vezes o cache é problemático, pois se houve uma atualização na versão da aplicação que está em repositório APT, ela pode não ser atualizada por estar em cache, esse é um exemplo de problema ao usar o cache, porém ele é muito útil
- Contêineres são imutáveis e efêmeros, sendo assim ele nasceu pra morrer, não confunda este com uma VM.
- As variáveis declaradas na instrução ENV do dockerfile pode ser utilizada dentro do próprio dockerfile
- SHELL ["powershell", "-command"] essa instrução muda o shell do container.
- ARGS Argumento que posso utilizar durante o build da imagem, exemplo após essa lista de dicas
- HEALTHCHECK, uma forma de verificar se o container está íntegro. Exemplo mais á frente

ADDCopia novos arquivos, diretórios, arquivos TAR ou arquivos remotos e os adicionam ao filesystem do container;
CMD => Executa um comando, diferente do RUN que executa o comando no momento em que está "buildando" a imagem, o CMD executa no início da execução do container;
LABEL => Adiciona metadados a imagem como versão, descrição e fabricante;
COPY => Copia novos arquivos e diretórios e os adicionam ao filesystem do container;
ENTRYPOINT => Permite você configurar um container para rodar um executável, e quando esse executável for finalizado, o container também será;
ENV => Informa variáveis de ambiente ao container;
EXPOSE => Informa qual porta o container estará ouvindo;
FROM => Indica qual imagem será utilizada como base, ela precisa ser a primeira linha do Dockerfile;
MAINTAINER => Autor da imagem;
RUN => Executa qualquer comando em uma nova camada no topo da imagem e "commita" as alterações. Essas alterações você poderá utilizar nas próximas instruções de seu Dockerfile;
USER => Determina qual o usuário será utilizado na imagem. Por default é o root;
VOLUME => Permite a criação de um ponto de montagem no container;
WORKDIR => Responsável por mudar do diretório / (raiz) para o especificado nele;



