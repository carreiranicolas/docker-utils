# Guia de Docker — Comandos e Conceitos

Anotações do curso de Docker, organizadas e com exemplos de uso.

> **Dica geral:** todo comando do Docker tem um `--help` que mostra as flags disponíveis.
> Ex: `docker run --help`, `docker network create --help`

## Sumário

1. [Containers](#1-containers)
2. [Dockerfile](#2-dockerfile)
3. [Imagens](#3-imagens)
4. [Comandos úteis para inspecionar containers](#4-comandos-úteis-para-inspecionar-containers)
5. [Docker Hub](#5-docker-hub)
6. [Volumes](#6-volumes)
7. [Networks](#7-networks)
8. [Docker Compose](#8-docker-compose)
9. [Limpeza do ambiente](#9-limpeza-do-ambiente)
10. [Resumão (cola rápida)](#10-resumão-cola-rápida)

---

## 1. Containers

Um **container** é uma instância em execução de uma **imagem**. Pensa na imagem como uma "receita" e no container como o "bolo" feito a partir dela. Dá pra ter vários containers da mesma imagem rodando ao mesmo tempo.

### `docker run` — criar e rodar um container

```bash
docker run [IMAGEM]
```

Cria e roda um container a partir da imagem. Se a imagem não estiver na sua máquina (em cache), o Docker baixa ela automaticamente do Docker Hub antes de rodar.

```bash
docker run hello-world
docker run nginx
```

Se quiser só baixar a imagem sem rodar nada, usa o `docker pull` (ver [Imagens](#3-imagens)).

#### Principais flags do `docker run`

| Flag | O que faz |
|------|-----------|
| `-it` | Modo interativo: dá acesso ao terminal do container pra rodar comandos lá dentro |
| `-d` | Modo *detached*: o container roda em background, sem travar o terminal |
| `-p <porta_host>:<porta_container>` | Expõe uma porta do container numa porta do seu computador |
| `--name <nome>` | Define o nome do container (se não passar, o Docker gera um aleatório) |
| `--rm` | Remove o container automaticamente quando ele parar |
| `-v` | Monta um volume (ver [Volumes](#6-volumes)) |
| `-e <VAR>=<valor>` | Define uma variável de ambiente no container |
| `--network <rede>` | Conecta o container a uma rede (ver [Networks](#7-networks)) |

**Exemplos:**

```bash
# Modo interativo: abre o bash dentro de um container Ubuntu
docker run -it ubuntu bash

# Roda o nginx em background
docker run -d nginx

# Porta 80 do meu computador -> porta 80 do container
# Depois é só acessar http://localhost no navegador
docker run -d -p 80:80 nginx

# Porta 8080 do meu computador -> porta 80 do container
# Acessa em http://localhost:8080
docker run -d -p 8080:80 nginx

# Dando nome ao container
docker run -d -p 80:80 --name meu_nginx nginx

# Container "descartável": some depois que parar
docker run --rm -it ubuntu bash
```

> As flags podem ser combinadas: `docker run -d -p 3000:3000 --name minha_api --rm minha_imagem`
> Repara que as flags vêm **antes** do nome da imagem.

### Vários containers da mesma imagem

Dá pra rodar vários containers da mesma imagem ao mesmo tempo, basta trocar a porta do host (e o nome, que precisa ser único):

```bash
docker run -d -p 3000:3000 --name meu_node1 minha_imagem
docker run -d -p 4000:3000 --name meu_node2 minha_imagem
docker run -d -p 5000:3000 --name meu_node3 minha_imagem
```

Aqui os três containers usam a porta 3000 **dentro** deles, mas no seu computador cada um fica acessível numa porta diferente (3000, 4000 e 5000).

### `docker ps` — listar containers

```bash
docker ps        # mostra só os containers rodando
docker ps -a     # mostra todos, inclusive os que já foram executados e estão parados
```

A saída mostra o ID, a imagem, o comando, há quanto tempo foi criado, o status, as portas e o nome de cada container.

### `docker stop` e `docker start` — parar e iniciar

```bash
docker stop <id ou nome>    # para a execução do container
docker start <id ou nome>   # inicia de novo um container que já existe
```

> **Importante:** parar um container **não** quer dizer que ele foi removido! Ele continua aparecendo no `docker ps -a` e pode ser iniciado de novo com `docker start`.

```bash
docker stop meu_nginx
docker start meu_nginx

# Pra entrar no terminal ao iniciar um container que foi criado com -it
docker start -i meu_ubuntu
```

> **Diferença entre `run` e `start`:** o `docker run` sempre cria um container **novo**. O `docker start` reaproveita um container que já existe.

### `docker logs` — ver os logs

```bash
docker logs <id ou nome>
docker logs -f <id ou nome>   # -f fica acompanhando os logs em tempo real
```

Muito útil quando o container está rodando em background (`-d`) e você quer ver o que a aplicação está printando ou se deu algum erro.

### `docker rm` — remover containers

```bash
docker rm <id ou nome>
docker rm -f <id ou nome>    # força a remoção mesmo se o container estiver rodando
```

Containers removidos não aparecem mais no `docker ps -a`.

```bash
docker rm meu_nginx
docker rm -f meu_node1 meu_node2 meu_node3   # dá pra remover vários de uma vez
```

> Dica: não precisa digitar o ID inteiro, só os primeiros caracteres já bastam (ex: `docker stop 3f2a`).

---

## 2. Dockerfile

O **Dockerfile** é o arquivo com as instruções pra construir uma imagem. Ele fica geralmente na raiz do projeto e se chama exatamente `Dockerfile` (sem extensão).

### Principais instruções

| Instrução | O que faz |
|-----------|-----------|
| `FROM` | Imagem base que vamos usar (ex: `node`, `python`, `ubuntu`) |
| `WORKDIR` | Define o diretório de trabalho dentro do container. Os comandos seguintes (`COPY`, `RUN`, `CMD`) vão rodar a partir dele |
| `COPY` | Quais arquivos precisam ser copiados e para onde (do seu computador pra imagem) |
| `RUN` | Comando que roda quando a imagem é **construída** (build). Ex: instalar dependências |
| `EXPOSE` | Porta que a aplicação usa (é mais uma documentação; quem publica a porta de fato é o `-p` no `docker run`) |
| `CMD` | Comando executado quando o **container inicia**. Aceita uma lista como argumento |

> **`RUN` x `CMD`:** o `RUN` roda na hora do build (preparando a imagem). O `CMD` roda toda vez que um container é iniciado a partir da imagem.

### Exemplo — aplicação Node

```dockerfile
# Imagem base
FROM node:20

# Diretório de trabalho dentro do container
WORKDIR /app

# Copia primeiro só o package.json pra aproveitar o cache das dependências
COPY package*.json .

# Instala as dependências (roda no build)
RUN npm install

# Copia o resto do projeto
COPY . .

# Porta da aplicação
EXPOSE 3000

# Comando que roda quando o container inicia
CMD ["node", "app.js"]
```

### Exemplo — aplicação Python (Flask)

```dockerfile
FROM python:3.12

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

---

## 3. Imagens

### `docker build` — criar uma imagem

```bash
docker build <diretório do Dockerfile>
```

```bash
# O "." significa "o diretório atual" (onde está o Dockerfile)
docker build .
```

> **Sempre que alterar o código do projeto, precisa fazer o build de novo** pra que a imagem tenha as mudanças. (Ou usar bind mount pra atualização em tempo real, ver [Volumes](#6-volumes).)

#### Já dando nome no build (`-t`)

```bash
docker build -t <nome>:<tag> <diretório>
```

```bash
docker build -t meu_node .          # sem tag, o Docker usa "latest"
docker build -t meu_node:1.0 .      # com tag de versão
```

### `docker image ls` — listar imagens

```bash
docker image ls
docker images      # faz a mesma coisa
```

Mostra o nome (repository), a tag, o ID, quando foi criada e o tamanho de cada imagem.

### `docker pull` — baixar uma imagem

```bash
docker pull <imagem>
```

Faz o download da imagem e deixa ela disponível no ambiente, sem rodar nada.

```bash
docker pull node
docker pull mysql:8.0     # baixando uma versão específica
```

### `docker tag` — nomear/renomear uma imagem

Usado quando a imagem já existe (por isso usamos o ID dela):

```bash
docker tag <id_imagem> <nome>
docker tag <id_imagem> <nome>:<tag>    # boa prática: sempre colocar a tag
```

A **tag** funciona como um versionamento da imagem.

```bash
docker tag 3f2a9b1c meu_node:1.0
docker tag meu_node:1.0 meu_node:latest   # dá pra usar o nome em vez do ID também
```

### `docker rmi` — remover imagens

```bash
docker rmi <imagem ou id>
docker rmi -f <imagem ou id>   # força a remoção mesmo se tiver container usando a imagem
```

```bash
docker rmi meu_node:1.0
```

---

## 4. Comandos úteis para inspecionar containers

### `docker cp` — copiar arquivos

Copia arquivos de um diretório do seu computador para um container, ou do container para o seu computador.

```bash
# Do container -> para o seu computador
docker cp <container>:<caminho_no_container> <destino_local>

# Do seu computador -> para o container
docker cp <arquivo_local> <container>:<caminho_no_container>
```

```bash
docker cp meu_node:/app/app.js ./copia/
docker cp ./config.json meu_node:/app/config.json
```

### `docker top` — processos do container

```bash
docker top <container>
```

Mostra os processos rodando dentro do container: ID do processo (PID), usuário, quando foi iniciado e o comando.

### `docker inspect` — configuração do container

```bash
docker inspect <container>
```

Mostra (em JSON) como o container está configurado: ID, data de criação, imagem usada, volumes, redes, IP, variáveis de ambiente etc.

```bash
docker inspect meu_node

# Pegando só o IP do container
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' meu_node
```

### `docker stats` — uso de recursos

```bash
docker stats
docker stats <container>
```

Mostra em tempo real o consumo de recursos dos containers em execução: CPU, memória, rede e disco. Pra sair, `Ctrl + C`.

### `docker exec` — rodar comandos em um container que já está rodando

Não estava nas anotações, mas é muito útil:

```bash
docker exec -it <container> bash    # abre um terminal dentro do container
docker exec <container> ls /app     # roda um comando só e volta
```

---

## 5. Docker Hub

O **Docker Hub** é o repositório online de imagens (tipo um GitHub, mas de imagens). É de lá que vêm as imagens quando damos `docker pull` ou `docker run`.

### Login e logout

```bash
docker login     # pede usuário e senha (ou token)
docker logout
```

Depois de logado, você pode enviar suas imagens pro Hub.

### `docker push` — enviar uma imagem

```bash
docker push <usuario>/<imagem>
```

Antes de enviar:

1. Crie o repositório no Docker Hub (com **exatamente** o mesmo nome da imagem).
2. A imagem precisa estar nomeada no formato `usuario/imagem`.

```bash
docker build -t carreiranicolas/meu_node .
docker push carreiranicolas/meu_node
```

### Atualizando uma imagem no Hub

1. Faz o build trocando a tag pra uma versão nova:
   ```bash
   docker build -t carreiranicolas/meu_node:2.0 .
   ```
2. Envia:
   ```bash
   docker push carreiranicolas/meu_node:2.0
   ```

Pra baixar a imagem em outra máquina:

```bash
docker pull carreiranicolas/meu_node:2.0
```

---

## 6. Volumes

Volumes são uma forma de **salvar dados fora dos containers**. Todo dado criado por um container fica salvo nele, então quando o container é removido, perdemos os dados. Com volumes, os dados continuam existindo mesmo depois de remover o container.

Exemplo clássico: um banco de dados num container. Sem volume, se remover o container, perde o banco inteiro.

### Tipos de volume

| Tipo | Descrição |
|------|-----------|
| **Anônimo** | Diretório criado pela flag `-v`, mas com um nome aleatório |
| **Nomeado** | Volume com nome. Fica fácil de referenciar e saber pra que serve no ambiente |
| **Bind mount** | Salva os dados num diretório da **sua máquina**, sem o gerenciamento do Docker. A gente informa qual diretório usar |

### Criando cada tipo

```bash
# Anônimo: só passa o caminho dentro do container
docker run -v /data <imagem>

# Nomeado: nome_do_volume:caminho_dentro_do_container
docker run -v <nome_do_volume>:/caminho/no/container <imagem>

# Bind mount: caminho_na_sua_maquina:caminho_dentro_do_container
docker run -v /dir/data:/data <imagem>
```

No bind mount, `/dir/data` é onde os dados vão ser salvos na sua máquina e `/data` é o diretório dentro do container.

**Exemplos:**

```bash
# Volume nomeado pra guardar os dados do MySQL
docker run -d --name meu_mysql -e MYSQL_ALLOW_EMPTY_PASSWORD=True \
  -v dados_mysql:/var/lib/mysql mysql

# Bind mount com o diretório atual (usando $(pwd) pra pegar o caminho completo)
docker run -d -p 3000:3000 -v $(pwd):/app meu_node
```

> **Bind mount pra atualização em tempo real:** o bind mount não serve só pra guardar dados. Dá pra usar como técnica pra atualizar o projeto em tempo real: como o diretório do projeto fica "espelhado" dentro do container, qualquer alteração no código aparece direto no container, sem precisar ficar dando build toda hora. Ótimo pra desenvolvimento.

### Volume só de leitura (`:ro`)

Útil quando o volume é só uma base de consulta e o container não deve alterar nada:

```bash
docker run -v <volume>:/data:ro <imagem>
```

### Gerenciando volumes manualmente

```bash
docker volume create <nome>     # cria um volume nomeado
docker volume ls                # lista os volumes (só anônimos e nomeados, bind mount não aparece)
docker volume inspect <nome>    # mostra detalhes, como onde o volume fica salvo na máquina
docker volume rm <nome>         # remove o volume
docker volume prune             # remove todos os volumes que não estão sendo usados
```

```bash
docker volume create dados_app
docker run -d -v dados_app:/app/data meu_node
docker volume inspect dados_app
```

> Cuidado: remover um volume apaga os dados dele de vez.

---

## 7. Networks

Networks são uma forma de gerenciar a conexão do Docker com outras plataformas ou até com outros containers. O uso mais comum é conectar uma aplicação que roda num container com um banco de dados que roda em outro. As redes são criadas separadas dos containers.

### Tipos de conexão

| Tipo | Descrição |
|------|-----------|
| **Externa** | O container se conecta a um serviço que já existe na web (uma API, por exemplo) |
| **Com o host** | Comunicação com a máquina que está rodando o Docker |
| **Entre containers** (mais usado) | Usa o driver `bridge` e permite a comunicação entre 2 ou mais containers |

### Drivers de rede

| Driver | Descrição |
|--------|-----------|
| `bridge` | O mais comum e o padrão do Docker. Usado quando precisamos conectar um container no outro |
| `host` | O container usa a rede da própria máquina que está rodando o Docker |
| `macvlan` | Permite conectar um container a um MAC address, como se fosse um dispositivo físico na rede |
| `none` | Remove todas as conexões de rede do container (ele fica isolado) |
| `plugins` | Permite instalar código de terceiros que simulam um driver ou uma rede |

### Comandos de rede

```bash
docker network ls                          # lista todas as redes do ambiente
docker network create <nome>               # cria uma rede (do tipo bridge, por padrão)
docker network create -d <driver> <nome>   # cria a rede com o driver especificado
docker network rm <nome>                   # remove uma rede
docker network prune                       # remove as redes que não estão sendo usadas
docker network inspect <nome>              # mostra informações da rede (containers conectados, IPs etc.)
docker network connect <rede> <container>     # conecta um container (já criado) a uma rede
docker network disconnect <rede> <container>  # desconecta um container de uma rede
```

> Algumas redes já vêm criadas quando você roda `docker network ls` (`bridge`, `host` e `none`). Elas fazem parte da configuração inicial do Docker.

```bash
docker network create minha_rede
docker network create -d macvlan minha_macvlan
docker network connect minha_rede meu_node
docker network inspect minha_rede
```

### Exemplos práticos (os do curso)

#### Externa
Um container rodando uma aplicação Flask que faz requisições pra uma API na internet. Não precisa configurar nada: por padrão o container já tem acesso à internet.

#### Com o host
A aplicação Flask (rodando num container) se conecta e insere dados num MySQL instalado **na sua máquina** (fora do Docker). Dentro do código, em vez de `localhost`, usamos o endereço especial `host.docker.internal`, que aponta pra máquina host:

```python
# No código do Flask
app.config['MYSQL_HOST'] = 'host.docker.internal'
```

> No Linux, pode ser necessário adicionar a flag `--add-host=host.docker.internal:host-gateway` no `docker run`.

#### Entre containers
Duas imagens diferentes rodando em containers separados: a aplicação Flask se conecta e insere dados num MySQL que também roda num container.

Passo a passo:

```bash
# 1. Build das imagens
docker build -t meu_mysql ./mysql
docker build -t meu_flask ./flask

# 2. Cria a rede
docker network create flasknetwork

# 3. Sobe o MySQL na rede (MYSQL_ALLOW_EMPTY_PASSWORD pra não precisar de senha)
docker run -d -p 3306:3306 --name mysql_api --rm \
  --network flasknetwork \
  -e MYSQL_ALLOW_EMPTY_PASSWORD=True \
  meu_mysql

# 4. Sobe o Flask na mesma rede
docker run -d -p 5000:5000 --name flask_api --rm \
  --network flasknetwork \
  meu_flask
```

No código do Flask, o host do banco é o **nome do container** do MySQL:

```python
app.config['MYSQL_HOST'] = 'mysql_api'
```

> Isso funciona porque, numa rede bridge criada por nós, o Docker resolve o nome do container como se fosse um endereço. Então um container acha o outro pelo nome.

---

## 8. Docker Compose

O **Docker Compose** é uma ferramenta pra rodar **múltiplos containers** de forma centralizada. Surgiu pra gente não ter que ficar rodando vários `build` e `run` na mão: com um comando só, sobe tudo.

### O arquivo `docker-compose.yaml`

É ele que coordena os containers e imagens. Principais chaves:

| Chave | O que faz |
|-------|-----------|
| `version` | Versão do compose. Nas versões mais novas do Docker é opcional/obsoleta, pode deixar de fora |
| `services` | Os containers/serviços que vão rodar na aplicação |
| `volumes` | Declaração dos volumes (nomeados) usados pelos serviços |
| `networks` | Declaração das redes usadas pelos serviços |

Dentro de cada serviço, as chaves mais usadas:

| Chave | O que faz | Equivale no `docker run` a |
|-------|-----------|----------------------------|
| `image` | Imagem usada | `docker run <imagem>` |
| `build` | Pasta onde está o Dockerfile (gera a imagem no compose) | `docker build` |
| `ports` | Mapeamento de portas | `-p` |
| `volumes` | Volumes / bind mounts | `-v` |
| `environment` | Variáveis de ambiente | `-e` |
| `env_file` | Arquivo com as variáveis de ambiente | `--env-file` |
| `networks` | Redes do serviço | `--network` |
| `depends_on` | Serviços que precisam subir antes | — |
| `restart` | Política de reinício (ex: `always`) | `--restart` |

### Exemplo básico

```yaml
services:
  db:
    image: mysql
    environment:
      MYSQL_ALLOW_EMPTY_PASSWORD: "True"
    volumes:
      - dados_mysql:/var/lib/mysql

  app:
    image: meu_flask
    ports:
      - "5000:5000"
    depends_on:
      - db

volumes:
  dados_mysql:
```

### Comandos do Compose

```bash
docker compose up        # faz os builds (se tiver) e roda os containers
docker compose up -d     # mesma coisa, mas em modo detached (background)
docker compose down      # para e remove os containers (e a rede criada pelo compose)
docker compose ps        # mostra os serviços que subiram com o compose up
docker compose logs      # logs de todos os serviços
docker compose up --build   # força o build das imagens de novo
```

> Os comandos têm que ser rodados na pasta onde está o `docker-compose.yaml`.

### Variáveis de ambiente com `env_file`

Em vez de colocar as variáveis direto no compose com a chave `environment`, dá pra colocar elas num arquivo separado e usar `env_file`. Assim não precisamos da chave `environment` (e evitamos deixar senha exposta no compose).

```yaml
services:
  db:
    image: mysql
    env_file:
      - ./config/db.env
```

Arquivo `./config/db.env`:

```env
MYSQL_ROOT_PASSWORD=minhasenha
MYSQL_DATABASE=meubanco
```

> Dica: coloque os arquivos `.env` no `.gitignore` pra não subir senha pro GitHub.

### Redes no Compose

O compose já cria uma rede básica do tipo bridge entre todos os containers da aplicação, então eles já conseguem se comunicar pelo nome do serviço (ex: o Flask acessa o banco pelo host `db`).

Mas dá pra **isolar as redes** com a chave `networks`, conectando só os containers que a gente escolher. E dá pra definir o driver:

```yaml
services:
  db:
    image: mysql
    networks:
      - backend

  app:
    build: ./flask
    networks:
      - backend
      - frontend

  web:
    image: nginx
    networks:
      - frontend

networks:
  backend:
    driver: bridge
  frontend:
    driver: bridge
```

Aqui o `web` não enxerga o `db`, só o `app` consegue falar com os dois.

### Build de imagens no Compose

Dá pra gerar o build das imagens durante o compose. É só usar a chave `build` com a pasta onde está o Dockerfile (no lugar de `image`):

```yaml
services:
  app:
    build: ./flask        # pasta com o Dockerfile
    ports:
      - "5000:5000"
```

### Bind mount no Compose

O bind mount permite a atualização em tempo real dos arquivos do container. No compose é só usar a chave `volumes` dentro do serviço que a gente quer. Diferente dos volumes nomeados, **não precisa declarar** o bind mount na chave `volumes` lá embaixo:

```yaml
services:
  app:
    build: ./flask
    ports:
      - "5000:5000"
    volumes:
      - ./flask:/app      # pasta do projeto na máquina -> /app no container
```

### Exemplo completo (Flask + MySQL)

```yaml
services:
  db:
    build: ./mysql
    restart: always
    env_file:
      - ./config/db.env
    ports:
      - "3306:3306"
    volumes:
      - dados_mysql:/var/lib/mysql
    networks:
      - backend

  backend:
    build: ./flask
    restart: always
    ports:
      - "5000:5000"
    volumes:
      - ./flask:/app
    depends_on:
      - db
    networks:
      - backend

volumes:
  dados_mysql:

networks:
  backend:
    driver: bridge
```

---

## 9. Limpeza do ambiente

Com o tempo, vai acumulando container parado, imagem velha e volume sem uso. Pra limpar:

```bash
docker system prune        # remove containers parados, redes sem uso, imagens sem nome e cache de build
docker system prune -a     # remove também todas as imagens que não estão sendo usadas por nenhum container
docker container prune     # remove só os containers parados
docker image prune         # remove só as imagens sem uso
docker volume prune        # remove só os volumes sem uso
docker network prune       # remove só as redes sem uso
```

> O `docker system prune` **não** remove volumes por padrão. Pra incluir, usa `docker system prune --volumes`.

---

## 10. Resumão (cola rápida)

### Containers
| Comando | O que faz |
|---------|-----------|
| `docker run <imagem>` | Cria e roda um container |
| `docker run -it <imagem>` | Modo interativo |
| `docker run -d <imagem>` | Background |
| `docker run -p 80:80 <imagem>` | Expõe porta |
| `docker run --name <nome> <imagem>` | Dá nome ao container |
| `docker run --rm <imagem>` | Remove ao parar |
| `docker ps` / `docker ps -a` | Lista containers rodando / todos |
| `docker stop <container>` | Para |
| `docker start <container>` | Inicia de novo |
| `docker logs <container>` | Logs |
| `docker rm <container>` | Remove (`-f` força) |
| `docker exec -it <container> bash` | Abre terminal num container rodando |
| `docker cp` | Copia arquivos entre máquina e container |
| `docker top <container>` | Processos do container |
| `docker inspect <container>` | Configuração do container |
| `docker stats` | Uso de CPU/memória |

### Imagens
| Comando | O que faz |
|---------|-----------|
| `docker build -t <nome>:<tag> .` | Cria a imagem |
| `docker image ls` | Lista imagens |
| `docker pull <imagem>` | Baixa imagem |
| `docker tag <id> <nome>:<tag>` | Nomeia imagem |
| `docker rmi <imagem>` | Remove imagem (`-f` força) |
| `docker login` / `docker logout` | Login no Docker Hub |
| `docker push <usuario>/<imagem>` | Envia pro Docker Hub |

### Volumes
| Comando | O que faz |
|---------|-----------|
| `docker run -v /data` | Volume anônimo |
| `docker run -v nome:/data` | Volume nomeado |
| `docker run -v /dir/data:/data` | Bind mount |
| `docker run -v nome:/data:ro` | Somente leitura |
| `docker volume create/ls/inspect/rm/prune` | Gerenciar volumes |

### Networks
| Comando | O que faz |
|---------|-----------|
| `docker network ls` | Lista redes |
| `docker network create <nome>` | Cria rede bridge |
| `docker network create -d <driver> <nome>` | Cria rede com driver |
| `docker network connect/disconnect <rede> <container>` | Conecta/desconecta container |
| `docker network inspect/rm/prune` | Gerenciar redes |

### Compose
| Comando | O que faz |
|---------|-----------|
| `docker compose up` | Sobe tudo (`-d` em background) |
| `docker compose down` | Derruba tudo |
| `docker compose ps` | Serviços rodando |
| `docker compose logs` | Logs dos serviços |
