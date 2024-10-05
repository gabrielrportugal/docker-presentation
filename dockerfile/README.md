# Build de uma nova imagem

Vamos imaginar que temos uma aplicação HTML simples que será disponibilizada em um servidor web Nginx através de um container. Para isso, precisamos criar uma imagem que contenha o Nginx e o código da aplicação.

Para isso, vamos criar um arquivo chamado `Dockerfile` na raiz do projeto com o seguinte conteúdo:

```dockerfile
FROM nginx:latest
COPY . /usr/share/nginx/html
```

A primeira linha indica qual imagem será utilizada como base para a nossa imagem. Nesse caso, estamos utilizando a imagem oficial do Nginx, que está disponível no Docker Hub.

A segunda linha indica que o conteúdo do diretório atual será copiado para o diretório `/usr/share/nginx/html` dentro do container.

Agora, vamos criar a imagem a partir do Dockerfile:

```bash
docker build -t meu-site .
```

O parâmetro `-t` indica o nome da imagem que será criada. Nesse caso, o nome da imagem será `meu-site`. O ponto indica que o Dockerfile está no diretório atual.

Após a execução do comando, podemos verificar que a imagem foi criada através do comando `docker images`:

```bash
docker images ls
```

Agora, podemos executar um container a partir da imagem que acabamos de criar:

```bash
docker run -d -p 8080:80 meu-site
```

Alterar o conteudo de index.html da imagem ngix não irá gerar outra imagem, lembre-se sempre que a imagem é imutável. Se você deseja publicar uma nova versão do seu site, você precisa criar uma nova imagem.

Para isso, vamos alterar o conteúdo do index.html e criar uma nova imagem:

```bash
docker build -t meu-site .
```

Agora, vamos executar um novo container a partir da nova imagem:

```bash
docker run -d -p 8080:80 meu-site
```

Quando geramos imagens sem passar uma tag, o Docker automaticamente cria uma tag `latest`. Por isso, podemos executar o container a partir da imagem `meu-site:latest`:

```bash
docker run -d -p 8080:80 meu-site:latest
```

Se quisermos ter mais controle sobre as versões da nossa imagem, podemos utilizar tags. Por exemplo, podemos criar uma tag `v1` para a primeira versão da nossa imagem:

```bash
docker build -t meu-site:v1 .
```

E podemos executar o container a partir da imagem `meu-site:v1`:

```bash
docker run -d -p 8080:80 meu-site:v1
```

Agora, podemos criar uma nova versão da nossa imagem e atribuir a tag `v2`:

```bash
docker build -t meu-site:v2 .
```

E podemos executar o container a partir da imagem `meu-site:v2`:

```bash
docker run -d -p 8080:80 meu-site:v2
```

Podemos verificar todas as imagens que temos no nosso host através do comando:

```bash
docker images ls
```

Podemos verificar todos os containers que estão em execução através do comando:

```bash
docker ps
```

# 03 - As instruções RUN, COPY e CMD

O Dockerfile suporta várias instruções, nesta aula vamos aprender sobre as instruções `RUN`, `COPY` e `EXPOSE`.

## Instrução RUN

A instrução `RUN` executa um comando durante o build da imagem. Por exemplo, se você precisa instalar um pacote, você pode utilizar a instrução `RUN` para executar o comando de instalação durante o build da imagem.

Vamos imaginar um exemplo que temos que publicar uma aplicação Node.js + Nginx. Para isto, temos 2 opções mais práticas, ou pegamos uma imagem Nginx e instalamos o Node.js, ou pegamos uma imagem Node.js e instalamos o Nginx. Vamos utilizar a segunda opção, pois a imagem Node.js é menor que a imagem Nginx.

Isto seria muito comum, apesar do Node.js ter um servidor web, o Nginx facilitaria configuração de cache, load balance, https e etc.

Para isto, vamos criar um Dockerfile com a seguinte instrução:

```dockerfile
FROM node:20-slim

RUN apt-get update && apt-get install -y nginx
```

A instrução `RUN` executa o comando `apt-get update && apt-get install -y nginx` durante o build da imagem. O comando `apt-get update` atualiza a lista de pacotes disponíveis e o comando `apt-get install -y nginx` instala o pacote `nginx`.

## Instrução COPY

A instrução `COPY` copia arquivos e diretórios para dentro da imagem. Por exemplo, se você precisa copiar um arquivo de configuração para dentro da imagem, você pode utilizar a instrução `COPY` para copiar o arquivo durante o build da imagem.

Não existe necessariamente uma ordem para posicionar as instruções no Dockerfile, mas normalmente nas primeiras linhas colocamos as instruções que não mudam com frequência, como a instrução `FROM`, `RUN` e `COPY`. Nas últimas linhas colocamos as instruções que mudam com frequência, como a instrução `CMD` e `ENTRYPOINT`.

O próximo passo seria copiar o arquivo de configuração do Nginx para dentro da imagem. O `nginx.conf` será necessário para informa-lo que ele será um proxy reverso para o Node.js.

Para isto, vamos criar um arquivo `nginx.conf` com o seguinte conteúdo:

```conf
http {
  server {
    listen 80;

    location / {
      proxy_pass http://localhost:3000;
    }
  }
}
```

Agora, vamos adicionar a instrução `COPY` no Dockerfile:

```dockerfile
FROM node:20-slim

RUN apt-get update && apt-get install -y nginx

COPY nginx.conf /etc/nginx/nginx.conf
```

A instrução `COPY` copia o arquivo `nginx.conf` para dentro da imagem no diretório `/etc/nginx/nginx.conf`.

Agora precisamos nos preocupar com a aplicação Node.js, vamos copiar o arquivo `package.json` para dentro da imagem e instalar as dependências.

Para isto, vamos criar um arquivo `package.json` com o seguinte conteúdo:

Agora, vamos adicionar a instrução `COPY` no Dockerfile:

```dockerfile
FROM node:20-slim

RUN apt-get update && apt-get install -y nginx

COPY nginx.conf /etc/nginx/nginx.conf

WORKDIR /usr/src/app

COPY index.js package.json ./

RUN npm install
```

A instrução `COPY . .`  copia todos os arquivos e diretórios para dentro da imagem no diretório atual. A instrução `RUN npm install` executa o comando `npm install` durante o build da imagem, gerando a node_modules

## Instrução WORKDIR

A instrução `WORKDIR` define o diretório de trabalho da imagem. Por exemplo, se você precisa definir o diretório de trabalho da imagem, você pode utilizar a instrução `WORKDIR` para definir o diretório de trabalho da imagem.

Como temos uma aplicação que rodará dentro do container, é importante delimitar um diretório de trabalho para a aplicação.

Muitas imagens não definem o diretório de trabalho, por exemplo, a imagem `node:20-slim` não define o diretório de trabalho. Por isto, quando executamos o comando `npm install`, sem `WORDIR` definida, o diretório de trabalho seria diretório raiz `/`, e o comando `npm install` instalaria as dependências no diretório raiz `/`, já outras imagens definem o diretório de trabalho, por exemplo, a imagem `nginx:1.21` define o diretório de trabalho como `/usr/share/nginx/html`.

Ao montarmos uma imagem, definir um diretório de trabalho é uma boa prática, pois organiza o código fonte da aplicação dentro da imagem, e evita que o código fonte fique espalhado dentro da imagem.

## Instrução EXPOSE

A instrução `EXPOSE` informa ao Docker que a imagem expõe uma porta. Por exemplo, se você precisa expor a porta 80, você pode utilizar a instrução `EXPOSE` para informar ao Docker que a imagem expõe a porta 80.

Agora precisamos informar ao Docker que a imagem expõe a porta 80, para isto, vamos adicionar a instrução `EXPOSE` no Dockerfile:

```dockerfile
FROM node:20-slim

RUN apt-get update && apt-get install -y nginx

COPY nginx.conf /etc/nginx/nginx.conf

COPY index.js package.json ./

RUN npm install

EXPOSE 80
```

A instrução `EXPOSE 80` informa ao Docker que a imagem expõe a porta 80. Esta instrução não publica a porta 80, ela apenas informa ao Docker que a imagem expõe a porta 80. Ela é utilizada para documentação e para o comando `docker run` publicar a porta 80.

## Instrução CMD

A instrução `CMD` executa um comando quando o container é iniciado. Por exemplo, se você precisa iniciar um servidor web quando o container é iniciado, você pode utilizar a instrução `CMD` para executar o comando de inicialização do servidor web quando o container é iniciado.

Agora precisamos informar ao Docker que a imagem deve iniciar o servidor web quando o container é iniciado, para isto, vamos adicionar a instrução `CMD` no Dockerfile:

```dockerfile
FROM node:20-slim

RUN apt-get update && apt-get install -y nginx

COPY nginx.conf /etc/nginx/nginx.conf

WORKDIR /usr/src/app

COPY index.js package.json ./

RUN npm install

EXPOSE 80

CMD ["node", "/usr/src/app/index.js"]
```

A instrução `CMD ["node", "/usr/src/app/index.js"]` informa ao Docker que a imagem deve executar o comando `npm start` quando o container é iniciado.

Esta notação de array é a notação JSON, ela é utilizada para executar comandos que possuem argumentos. Por exemplo, se você precisa executar o comando `node /usr/src/app/index.js`, você pode utilizar a notação JSON para executar o comando. Você também pode executar a notação de string, mas o Docker interpreta melhor a notação JSON, porque ela organiza melhor os argumentos.

## Testando a imagem primeira versão da imagem

Agora que criamos o Dockerfile, vamos testar a imagem. Para isto, vamos executar o comando `docker build` para criar a imagem:

```bash
docker build -t node-nginx .
```

O comando `docker build` cria a imagem `node-nginx` com base no Dockerfile no diretório atual.

Agora, vamos executar o comando `docker run` para iniciar o container:

```bash
docker run -p 8000:3000 node-nginx
```

Acesse o endereço `http://localhost:8000` no navegador para testar a aplicação.

## Versão da imagem com o Nginx como proxy reverso

Agora que testamos a imagem, vamos melhorar a imagem. Vamos adicionar o Nginx como proxy reverso para o Node.js.

Vamos alterar o arquivo `Dockerfile` para o seguinte conteúdo:

```dockerfile
FROM node:20-slim

RUN apt-get update && apt-get install -y nginx

COPY nginx.conf /etc/nginx/nginx.conf

WORKDIR /usr/src/app

COPY index.js package.json ./

RUN npm install

EXPOSE 80

CMD [ "/bin/sh", "-c", "node /usr/src/app/index.js & nginx -g 'daemon off;'" ]
```

Para testar a imagem, precisamos publicar a porta 80, vez da 3000. Para isto, vamos executar o comando `docker run` para iniciar o container:

```bash
docker run -p 8000:80 node-nginx
```

No caso do CMD, como temos 2 comandos, precisamos utilizar o `sh` diretamente para usar o & para rodar os 2 comandos ao mesmo tempo.

