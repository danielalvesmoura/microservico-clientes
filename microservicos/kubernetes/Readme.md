# Kubernetes — Microserviço de Produtos

Este roteiro apresenta uma introdução prática ao Kubernetes utilizando o microserviço **Produtos** do projeto de microserviços.

O objetivo é executar o serviço `produtos` no Kubernetes, utilizando múltiplas réplicas, um `Service` para comunicação e o PostgreSQL executado separadamente via Docker Compose.

Ao final, teremos aproximadamente esta arquitetura:

    Windows / Host
         |
         | port-forward (somente para acesso externo/testes)
         v
    Service produtos :3001
         |
         +-- Pod produtos
         +-- Pod produtos
         +-- Pod produtos
                |
                | host.docker.internal:5433
                v
         PostgreSQL
         Docker Compose


## 1. Pré-requisitos

Antes de iniciar, é necessário ter instalado:

- Docker Desktop
- Kubernetes habilitado no Docker Desktop
- `kubectl`
- `kind`

Verifique o Docker:

    docker --version

Verifique o Kubernetes:

    kubectl version --client

Verifique o `kind`:

    kind version

Caso o `kind` ainda não esteja instalado no Windows:

    winget install Kubernetes.kind

Após a instalação, pode ser necessário fechar e abrir novamente o PowerShell.


## 2. Criar o cluster Kubernetes

No Docker Desktop, acesse:

    Kubernetes

Crie um cluster utilizando:

- Cluster type: `kind`
- Nodes: `1`
- Kubernetes: versão disponível/recomendada pelo Docker Desktop

Aguarde até o cluster estar em execução.

Depois, no terminal:

    kubectl get nodes

O resultado deverá apresentar o node como `Ready`, por exemplo:

    NAME                    STATUS   ROLES           AGE   VERSION
    desktop-control-plane   Ready    control-plane   ...   ...


## 3. Verificar o cluster

Antes de criar nossa aplicação, podemos verificar os recursos existentes:

    kubectl get pods

Por padrão, esse comando consulta o namespace `default`.

Também podemos verificar:

    kubectl get deployments

e:

    kubectl get services


## 4. Construir a imagem do microserviço Produtos

O projeto já possui um `Dockerfile` dentro da pasta `produtos`.

Na pasta `microservicos`, construa a imagem:

    docker build -t microservicos-produtos:latest ./produtos

Verifique se ela foi criada:

    docker images

Deverá existir uma imagem semelhante a:

    microservicos-produtos   latest


## 5. Disponibilizar a imagem para o cluster kind

Um ponto importante é que possuir uma imagem no Docker local não significa necessariamente que ela esteja disponível dentro do node do cluster `kind`.

O cluster Kubernetes precisa conseguir encontrar a imagem.

Primeiro descubra o nome do cluster:

    kind get clusters

No Docker Desktop, normalmente será apresentado:

    desktop

Carregue a imagem no cluster:

    kind load docker-image microservicos-produtos:latest --name desktop

O `kind` copiará a imagem para o node do Kubernetes.

Esse passo evita a necessidade de publicar a imagem no Docker Hub.


## 6. Criar o Deployment

Crie a pasta:

    kubernetes

Dentro dela, crie:

    produtos-deployment.yaml

Conteúdo:

    apiVersion: apps/v1
    kind: Deployment

    metadata:
      name: produtos

    spec:
      replicas: 3

      selector:
        matchLabels:
          app: produtos

      template:
        metadata:
          labels:
            app: produtos

        spec:
          containers:
            - name: produtos
              image: microservicos-produtos:latest
              imagePullPolicy: Never

              ports:
                - containerPort: 3001

              env:
                - name: DATABASE_URL
                  value: "postgres://postgres:postgres@postgres:5432/produtos_db"

### Entendendo o Deployment

O trecho:

    replicas: 3

determina que o Kubernetes deverá manter três instâncias da aplicação em execução.

Cada instância será executada em um Pod.

Portanto:

    Deployment produtos
         |
         +-- Pod produtos
         +-- Pod produtos
         +-- Pod produtos

O trecho:

    app: produtos

é um `label`.

Esse label será importante posteriormente para o `Service` localizar os Pods.


## 7. Sobre imagePullPolicy

Foi utilizada:

    imagePullPolicy: Never

Isso significa:

> O Kubernetes não deverá tentar baixar a imagem de um registry externo.

Ele deverá utilizar a imagem existente no node.

Por isso foi necessário executar anteriormente:

    kind load docker-image microservicos-produtos:latest --name desktop

Caso a imagem não esteja disponível no node, os Pods poderão apresentar:

    ErrImageNeverPull

Nesse caso, carregue novamente a imagem:

    kind load docker-image microservicos-produtos:latest --name desktop


## 8. Subir somente o PostgreSQL

Neste exemplo, inicialmente não colocaremos o PostgreSQL dentro do Kubernetes.

O banco continuará sendo executado pelo Docker Compose.

Na pasta `microservicos`:

    docker compose up -d postgres

Onde:

- `docker compose up` cria/inicia os serviços;
- `-d` executa em segundo plano (`detached`);
- `postgres` indica que queremos iniciar somente o serviço PostgreSQL.

Portanto:

    docker compose up -d

subiria todos os serviços definidos no Compose.

Enquanto:

    docker compose up -d postgres

sobe somente o PostgreSQL.


## 9. Verificar o PostgreSQL

Execute:

    docker ps

O PostgreSQL deverá aparecer em execução.

Neste projeto, o Compose disponibiliza o banco aproximadamente como:

    Host: localhost
    Porta externa: 5433
    Porta interna: 5432

O mapeamento é:

    5433:5432


## 10. Por que não utilizar localhost dentro do Pod?

Dentro de um Pod:

    localhost

refere-se ao próprio Pod.

Portanto, se a aplicação tentar:

    localhost:5432

ela estará procurando PostgreSQL dentro do próprio Pod.

Isso pode gerar:

    ECONNREFUSED 127.0.0.1:5432

ou:

    ECONNREFUSED ::1:5432

Como o PostgreSQL está sendo executado pelo Docker Compose e disponibilizado pelo host, utilizamos:

    host.docker.internal:5433

Por isso definimos:

    DATABASE_URL=postgres://postgres:postgres@host.docker.internal:5433/produtos_db


## 11. Aplicar o Deployment

Execute:

    kubectl apply -f kubernetes/produtos-deployment.yaml

Verifique:

    kubectl get deployments

Depois:

    kubectl get pods

O resultado esperado é semelhante a:

    NAME                        READY   STATUS    RESTARTS
    produtos-xxxxxxxxxx-xxxxx   1/1     Running   0
    produtos-xxxxxxxxxx-xxxxx   1/1     Running   0
    produtos-xxxxxxxxxx-xxxxx   1/1     Running   0

Temos agora três instâncias do microserviço Produtos.


## 12. Consultar logs

Caso algum Pod apresente erro, descubra seu nome:

    kubectl get pods

Depois:

    kubectl logs NOME_DO_POD

Exemplo:

    kubectl logs produtos-795fff59f8-26jk4

Caso o container tenha reiniciado:

    kubectl logs produtos-795fff59f8-26jk4 --previous

Estados como:

    CrashLoopBackOff

indicam que o container conseguiu iniciar, mas a aplicação está encerrando com erro.

Um exemplo observado foi a tentativa de conexão ao PostgreSQL utilizando `localhost`.


## 13. Criar o Service

Os Pods são recursos descartáveis.

Se um Pod for removido e recriado, ele pode receber outro endereço IP.

Por isso, outros microserviços não devem depender diretamente do IP de um Pod.

Criamos então um `Service`.

Crie:

    kubernetes/produtos-service.yaml

Conteúdo:

    apiVersion: v1
    kind: Service

    metadata:
      name: produtos

    spec:
      selector:
        app: produtos

      ports:
        - port: 3001
          targetPort: 3001

      type: ClusterIP


## 14. Aplicar o Service

Execute:

    kubectl apply -f kubernetes/produtos-service.yaml

Verifique:

    kubectl get services

Deverá aparecer algo semelhante a:

    NAME         TYPE        CLUSTER-IP      PORT(S)
    kubernetes   ClusterIP   ...
    produtos     ClusterIP   ...             3001/TCP


## 15. Como o Service encontra os Pods?

No Deployment definimos:

    labels:
      app: produtos

No Service definimos:

    selector:
      app: produtos

Portanto, o Service procura Pods que possuam:

    app=produtos

Podemos visualizar esses Pods com:

    kubectl get pods -l app=produtos

A arquitetura passa a ser:

    Service produtos
          |
          | selector: app=produtos
          |
          +-- Pod produtos
          +-- Pod produtos
          +-- Pod produtos


## 16. Comunicação dentro do Kubernetes

O Service foi criado com:

    type: ClusterIP

Isso significa que ele é acessível internamente no cluster.

Outro microserviço executado no mesmo cluster poderá acessar Produtos utilizando:

    http://produtos:3001

Observe que não precisamos conhecer:

- IP do Pod;
- nome do Pod;
- qual das três réplicas atenderá;
- endereço do computador host.

O Kubernetes fornece resolução de nomes para o Service.


## 17. Testar a aplicação a partir do Windows

Como o `ClusterIP` é interno ao Kubernetes, podemos utilizar `port-forward` para testes.

Execute:

    kubectl port-forward service/produtos 3001:3001

Enquanto o comando estiver ativo:

    Windows localhost:3001
             |
             | port-forward
             v
       Service produtos
             |
             +-- Pod
             +-- Pod
             +-- Pod

Em outro terminal:

    curl http://localhost:3001/produtos

Ou abra no navegador:

    http://localhost:3001/produtos

O `port-forward` é utilizado aqui apenas para teste/acesso externo.

Ele não é necessário para comunicação entre aplicações dentro do Kubernetes.


## 18. Testar o self-healing

Uma das características importantes do Kubernetes é manter o estado desejado da aplicação.

Nosso Deployment especifica:

    replicas: 3

Primeiro:

    kubectl get pods

Escolha um dos Pods e remova:

    kubectl delete pod NOME_DO_POD

Exemplo:

    kubectl delete pod produtos-xxxxxxxxxx-xxxxx

Em seguida:

    kubectl get pods

O Deployment perceberá que existem somente dois Pods e criará automaticamente um terceiro.

Para acompanhar continuamente:

    kubectl get pods -w

O `-w` significa `watch`.

Para encerrar:

    Ctrl + C

Conceitualmente:

    Estado desejado
       3 Pods
         |
         v
       Kubernetes
         |
         v
       3 Pods

Ao excluir um:

       3 desejados
         |
         v
       2 existentes
         |
         v
    Kubernetes detecta
         |
         v
      cria outro
         |
         v
       3 existentes

Isso demonstra o mecanismo de **self-healing**.


## 19. Deployment, Pod e Service

### Pod

É a menor unidade gerenciada pelo Kubernetes.

Neste projeto, cada Pod contém uma instância do microserviço Produtos.

    Pod
     |
     +-- container Node.js


### Deployment

Define o estado desejado da aplicação.

Por exemplo:

    replicas: 3

O Deployment é responsável por manter essa quantidade de Pods.


### Service

Fornece um endereço estável para acessar um conjunto de Pods.

Neste exemplo:

    http://produtos:3001

pode representar qualquer uma das três instâncias do microserviço.


## 20. Arquitetura atual

Após esses passos:

    +---------------------------------------------+
    | Kubernetes                                  |
    |                                             |
    |          Service produtos :3001             |
    |                  |                          |
    |        +---------+---------+                |
    |        |         |         |                |
    |        v         v         v                |
    |      Pod 1     Pod 2     Pod 3              |
    |      Node      Node      Node               |
    |       .js       .js       .js               |
    |        \         |         /                 |
    |         \        |        /                  |
    +----------\-------|-------/-------------------+
                \      |      /
                 v     v     v

           host.docker.internal:5433
                      |
                      v
             +----------------+
             |   PostgreSQL   |
             | Docker Compose |
             |  produtos_db   |
             +----------------+


## 21. Comandos principais

Verificar o cluster:

    kubectl get nodes

Listar Pods:

    kubectl get pods

Listar Pods continuamente:

    kubectl get pods -w

Listar Deployments:

    kubectl get deployments

Listar Services:

    kubectl get services

Aplicar Deployment:

    kubectl apply -f kubernetes/produtos-deployment.yaml

Aplicar Service:

    kubectl apply -f kubernetes/produtos-service.yaml

Ver logs:

    kubectl logs NOME_DO_POD

Excluir um Pod:

    kubectl delete pod NOME_DO_POD

Excluir todos os Pods de Produtos:

    kubectl delete pods -l app=produtos

Carregar imagem local no cluster:

    kind load docker-image microservicos-produtos:latest --name desktop

Subir somente PostgreSQL:

    docker compose up -d postgres

Testar o Service externamente:

    kubectl port-forward service/produtos 3001:3001


## 22. Fluxo completo para repetir a demonstração

Depois que toda a configuração já estiver pronta, o fluxo resumido para repetir o experimento é:

### 1. Iniciar Docker Desktop e Kubernetes

Verificar:

    kubectl get nodes


### 2. Construir a imagem

    docker build -t microservicos-produtos:latest ./produtos


### 3. Carregar a imagem no kind

    kind load docker-image microservicos-produtos:latest --name desktop


### 4. Subir PostgreSQL

    docker compose up -d postgres


### 5. Criar o Deployment

    kubectl apply -f kubernetes/produtos-deployment.yaml


### 6. Criar o Service

    kubectl apply -f kubernetes/produtos-service.yaml


### 7. Conferir

    kubectl get pods
    kubectl get deployments
    kubectl get services


### 8. Testar externamente

    kubectl port-forward service/produtos 3001:3001

Em outro terminal:

    curl http://localhost:3001/produtos


### 9. Demonstrar self-healing

    kubectl get pods

Excluir um:

    kubectl delete pod NOME_DO_POD

Acompanhar:

    kubectl get pods -w


## Próximos passos

A partir deste ponto, a evolução natural do exemplo é:

1. adicionar o microserviço `pedidos` ao Kubernetes;
2. criar um Deployment para `pedidos`;
3. criar um Service para `pedidos`;
4. configurar `pedidos` para acessar Produtos por:

       http://produtos:3001

5. adicionar o API Gateway;
6. fazer o Gateway acessar Produtos e Pedidos pelos respectivos Services;
7. posteriormente avaliar a execução do PostgreSQL dentro do próprio Kubernetes.

Isso permitirá evoluir da demonstração de um único microserviço para a arquitetura completa:

    Cliente
       |
       v
    Gateway
       |
       +----------------+
       |                |
       v                v
    Pedidos          Produtos
       |                |
       +-------> Produtos
       |
       v
    Bancos de dados