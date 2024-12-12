# [GCSI] Atividade 2 - Criação do Load Balancer para recursos da aplicação Front-end.

## Passo a passo para configurar um Load Balancer com Nginx

### 1. Criar o build da aplicação

Crie o build da sua aplicação conforme necessário.

```bash
npm run build
```

### 2. Copiar a pasta build para o diretório

Copie a pasta build para o diretório onde será feita a configuração para o balanceamento de carga.

### 3. Configurar o arquivo `nginx.conf`

Configure o arquivo `nginx.conf` com as definições necessárias.

#### Arquivo `nginx.conf`

```nginx
user nginx;  # Usuário Nginx
worker_processes auto;  # Número de processos

events {
    worker_connections 1024;  # Máx. conexões
}

http {
    include mime.types;  # Tipos MIME
    default_type application/octet-stream;  # Tipo padrão
    
    include /etc/nginx/conf.d/*.conf;  # Incluir configs adicionais
}

```

### 4. Configurar o arquivo `default.conf`

Configure o arquivo `default.conf` conforme descrito abaixo.

#### Arquivo `default.conf`

```nginx
upstream nodes {  # Servidores upstream
    server node1:80;
    server node2:80;
    server node3:80;
    server node4:80;
    server node5:80;
}

server {
    listen 80;  
    server_name localhost;  # Nome do servidor

    location / {
        proxy_pass http://nodes;  # Redirecionar para nodes
        proxy_set_header X-Real-IP $remote_addr;  # IP do cliente
    }
}

```

### 5. Criar uma rede personalizada

Crie uma rede personalizada para que os nomes dos contêineres sejam resolvidos automaticamente:

```bash
docker network create my-gcsi-network
```

### 6. Conectar os nós à rede personalizada

Remova os contêineres existentes dos nós e recrie-os na nova rede:

```bash
docker rm -f node1 node2 node3 node4 node5

docker run -d --name node1 --hostname node1 --network my-gcsi-network -v ./build:/usr/share/nginx/html nginx:alpine

docker run -d --name node2 --hostname node2 --network my-gcsi-network -v ./build:/usr/share/nginx/html nginx:alpine

docker run -d --name node3 --hostname node3 --network my-gcsi-network -v ./build:/usr/share/nginx/html nginx:alpine

docker run -d --name node4 --hostname node4 --network my-gcsi-network -v ./build:/usr/share/nginx/html nginx:alpine

docker run -d --name node5 --hostname node5 --network my-gcsi-network -v ./build:/usr/share/nginx/html nginx:alpine
```

O comando cria e executa um contêiner Nginx em segundo plano, nomeia-o como `node1`, configura o hostname como `node1`, conecta-o à rede personalizada `my-gcsi-network`, e mapeia o diretório `./build` do host para `/usr/share/nginx/html` dentro do contêiner. Execute o comando em todos os nós incluídos no upstream.

####  **Por que isso funciona?**

Quando uma **rede personalizada** é criada no Docker, ela configura automaticamente um DNS interno, permitindo que os contêineres se comuniquem pelo nome (`node1`, `node2`, etc.). Isso não acontece com a rede padrão `bridge`.


### 7. Conectar o load balancer à rede personalizada

Remova o contêiner atual do load balancer e recrie-o na mesma rede:

```bash
docker rm -f loadbalancer

docker run -d --name loadbalancer --hostname loadbalancer -p 80:80 --network my-gcsi-network \
-v ./nginx.conf:/etc/nginx/nginx.conf \
-v ./default.conf:/etc/nginx/conf.d/default.conf \
nginx:alpine
```

O comando cria e executa um contêiner Nginx em segundo plano com o nome e hostname loadbalancer, mapeia a porta 80 do host para a porta 80 do contêiner, conecta o contêiner à rede personalizada `my-gcsi-network`, e monta os arquivos `nginx.conf` e `default.conf` do host nos diretórios apropriados dentro do contêiner.
