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

## DOCKERFILES
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

> ENTRYPOINT: Principal processo do container, é um INIT do container.
> CMD: O CMD apenas passa parâmetros para o processo principal. No exemplo o ENTRYPOINT é o *apachectl*, então passamos o parâmetro para iniciar o apache em FOREGROUND.

### Constuindo a image a partir do Dockerfile
``` sh
docker image build -t meu-apache:1.0
docker image build -t meu-apache:1.0 -no-cache
```

### Outras instruções do Dockerfile
*COPY*: Copia arquivo para dentro do container durante o build
*ADD*: Copiar arquivo para dentro do container durante o build, porém pode especificar uma URL para download e também arquivos compactados serão descompactados no container
*RUN*: Executa comandos dentro do container
*ENV*: Cria uma variável de ambiente
*USER*: Especifica um usuário dentro do container para executar os processos
*WORKDIR*: Diretório padrão do container

## Imagens
``` sh
docker image ls
docker image insepect IMAGENAME
docker image prune --all --filter until=240h
docker image prune -a
```


