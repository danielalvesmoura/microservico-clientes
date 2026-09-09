# Kubernetes — Utilizando imagem do Docker Hub

Este roteiro apresenta a execução do microserviço **Produtos** no Kubernetes utilizando uma imagem previamente publicada no Docker Hub.

Diferentemente da abordagem com imagens locais, **não é necessário instalar o CLI do `kind` nem executar `kind load docker-image`**.

A arquitetura utilizada será:

```text
Docker Hub
terenciani/microservicos-produtos:1.0
              |
              | pull
              v
+------------------------------------+
| Kubernetes                         |
|                                    |
|       Service produtos :3001       |
|                |                   |
|       +--------+--------+          |
|       |        |        |          |
|       v        v        v          |
|     Pod 1    Pod 2    Pod 3        |
|                                    |
+------------------------------------+
         |       |       |
         +-------+-------+
                 |
                 v
       host.docker.internal:5433
                 |
                 v
           PostgreSQL
         Docker Compose
```

---

## 1. Pré-requisitos

É necessário ter:

- Docker Desktop;
- Kubernetes habilitado no Docker Desktop;
- `kubectl`;
- acesso à Internet.

Não é necessário instalar o CLI do `kind`.

O próprio Docker Desktop pode utilizar `kind` internamente para criar o cluster Kubernetes.

Verifique o Docker:

```powershell
docker --version
```

Verifique o `kubectl`:

```powershell
kubectl version --client
```

---

## 2. Criar o cluster Kubernetes

No Docker Desktop, acesse:

```text
Kubernetes
```

Crie o cluster utilizando:

```text
Cluster type: kind
Nodes: 1
Kubernetes: versão disponível/recomendada
```

Aguarde o cluster ficar disponível.

Depois execute:

```powershell
kubectl get nodes
```

O resultado deverá apresentar o node como `Ready`:

```text
NAME                    STATUS   ROLES
desktop-control-plane   Ready    control-plane
```

---

## 3. Imagem utilizada

Neste exemplo, não construiremos a imagem localmente para disponibilizá-la ao Kubernetes.

Utilizaremos a imagem publicada no Docker Hub:

```text
terenciani/microservicos-produtos:1.0
```

O fluxo será:

```text
Docker Hub
    |
    | docker pull realizado pelo runtime
    v
Kubernetes Node
    |
    v
Container
```

O Kubernetes poderá obter a imagem automaticamente quando criar os Pods.

---

## 4. Subir o PostgreSQL

Neste estágio, somente os microserviços serão executados no Kubernetes.

O PostgreSQL continuará sendo executado pelo Docker Compose.

Na pasta `microservicos`, execute:

```powershell
docker compose up -d postgres
```

Onde:

- `docker compose up` inicia os serviços;
- `-d` executa em segundo plano (`detached`);
- `postgres` seleciona somente o serviço PostgreSQL.

Verifique:

```powershell
docker ps
```

Aguarde até o PostgreSQL estar em execução e, preferencialmente, com status:

```text
healthy
```

---

## 5. Comunicação com o PostgreSQL

No `docker-compose.yml`, o PostgreSQL utiliza o mapeamento:

```text
5433:5432
```

Isso significa:

```text
Host Windows :5433
        |
        v
Container PostgreSQL :5432
```

Dentro de um Pod Kubernetes, não devemos utilizar:

```text
localhost:5432
```

porque `localhost` representa o próprio Pod.

Para acessar um serviço disponibilizado pelo Docker Desktop no host, utilizaremos:

```text
host.docker.internal:5433
```

Assim:

```text
Pod Produtos
     |
     | host.docker.internal:5433
     v
Host
     |
     | 5433 -> 5432
     v
PostgreSQL
```

---

## 6. Criar o Deployment

Crie:

```text
kubernetes/produtos-deployment.yaml
```

Conteúdo:

```yaml
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

          image: terenciani/microservicos-produtos:1.0
          imagePullPolicy: IfNotPresent

          ports:
            - containerPort: 3001

          env:
            - name: DATABASE_URL
              value: "postgres://postgres:postgres@host.docker.internal:5433/produtos_db"
```

---

## 7. Entendendo a imagem

A configuração:

```yaml
image: terenciani/microservicos-produtos:1.0
```

indica qual imagem deverá ser utilizada.

Como ela está publicada em um repositório público no Docker Hub, o Kubernetes pode baixá-la.

Não precisamos executar:

```powershell
docker build ...
```

nem:

```powershell
kind load docker-image ...
```

na máquina que apenas executará o Deployment.

---

## 8. imagePullPolicy

Utilizamos:

```yaml
imagePullPolicy: IfNotPresent
```

Isso significa:

```text
A imagem já existe no node?
        |
    +---+---+
    |       |
   SIM     NÃO
    |       |
 utiliza   baixa
 local     do registry
```

Portanto, se a imagem ainda não estiver no node, o Kubernetes poderá obtê-la do Docker Hub.

---

## 9. Aplicar o Deployment

Execute:

```powershell
kubectl apply -f kubernetes/produtos-deployment.yaml
```

Confira:

```powershell
kubectl get deployments
```

Depois:

```powershell
kubectl get pods
```

O Kubernetes deverá criar três Pods:

```text
NAME                        READY   STATUS    RESTARTS
produtos-xxxxxxxxxx-aaaaa   1/1     Running   0
produtos-xxxxxxxxxx-bbbbb   1/1     Running   0
produtos-xxxxxxxxxx-ccccc   1/1     Running   0
```

Isso ocorre porque definimos:

```yaml
replicas: 3
```

---

## 10. Acompanhar a criação dos Pods

Também podemos acompanhar em tempo real:

```powershell
kubectl get pods -w
```

Durante a primeira execução poderá aparecer:

```text
Pending
ContainerCreating
Running
```

Para interromper o acompanhamento:

```text
Ctrl + C
```

---

## 11. Consultar logs

Se algum Pod apresentar erro:

```powershell
kubectl get pods
```

Copie o nome do Pod e execute:

```powershell
kubectl logs NOME_DO_POD
```

Exemplo:

```powershell
kubectl logs produtos-xxxxxxxxxx-aaaaa
```

Se o container já tiver reiniciado:

```powershell
kubectl logs NOME_DO_POD --previous
```

---

## 12. Criar o Service

Agora criaremos um endereço estável para acessar os Pods.

Crie:

```text
kubernetes/produtos-service.yaml
```

Conteúdo:

```yaml
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
```

---

## 13. Aplicar o Service

Execute:

```powershell
kubectl apply -f kubernetes/produtos-service.yaml
```

Confira:

```powershell
kubectl get services
```

Deverá aparecer:

```text
NAME         TYPE        CLUSTER-IP      PORT(S)
kubernetes   ClusterIP   ...
produtos     ClusterIP   ...             3001/TCP
```

---

## 14. Relação entre Service e Pods

O Deployment coloca o seguinte label nos Pods:

```yaml
labels:
  app: produtos
```

O Service procura:

```yaml
selector:
  app: produtos
```

Assim:

```text
Service produtos
       |
       | selector
       | app=produtos
       |
       +--------+--------+
       |        |        |
       v        v        v
     Pod 1    Pod 2    Pod 3
```

Podemos conferir:

```powershell
kubectl get pods -l app=produtos
```

---

## 15. Endereço interno

Como o Service é:

```yaml
type: ClusterIP
```

ele é destinado principalmente à comunicação dentro do cluster.

Outro microserviço poderá acessar Produtos utilizando:

```text
http://produtos:3001
```

Não precisamos descobrir o IP dos Pods.

Por exemplo, futuramente:

```text
Pod Pedidos
     |
     | http://produtos:3001
     v
Service produtos
     |
     +---- Pod Produtos 1
     +---- Pod Produtos 2
     +---- Pod Produtos 3
```

---

## 16. Testar pelo Windows

Para acessar temporariamente o Service a partir do computador:

```powershell
kubectl port-forward service/produtos 3001:3001
```

Enquanto esse comando estiver ativo:

```text
Windows
localhost:3001
     |
     | port-forward
     v
Service produtos:3001
     |
     +---- Pod 1
     +---- Pod 2
     +---- Pod 3
```

Em outro terminal:

```powershell
curl http://localhost:3001/produtos
```

Ou pelo navegador:

```text
http://localhost:3001/produtos
```

O `port-forward` é necessário apenas para esse acesso externo de teste.

Os serviços dentro do cluster não precisam dele.

---

## 17. Demonstrar self-healing

Confira os Pods:

```powershell
kubectl get pods
```

Escolha um:

```text
produtos-xxxxxxxxxx-aaaaa
```

Exclua:

```powershell
kubectl delete pod produtos-xxxxxxxxxx-aaaaa
```

Acompanhe:

```powershell
kubectl get pods -w
```

O Kubernetes deverá criar automaticamente outro Pod.

Isso ocorre porque o Deployment determina:

```yaml
replicas: 3
```

Portanto:

```text
Estado desejado = 3
        |
        v
    3 Pods
        |
    excluímos 1
        |
        v
    2 Pods
        |
        v
Deployment detecta
        |
        v
    cria 1
        |
        v
    3 Pods
```

---

## 18. Fluxo completo para repetir a aula

### 1. Iniciar Docker Desktop

Certifique-se de que Docker e Kubernetes estejam ativos.

```powershell
kubectl get nodes
```

O node deve estar:

```text
Ready
```

### 2. Subir o PostgreSQL

```powershell
docker compose up -d postgres
```

### 3. Criar o Deployment

```powershell
kubectl apply -f kubernetes/produtos-deployment.yaml
```

### 4. Criar o Service

```powershell
kubectl apply -f kubernetes/produtos-service.yaml
```

### 5. Verificar

```powershell
kubectl get deployments
```

```powershell
kubectl get pods
```

```powershell
kubectl get services
```

### 6. Testar

```powershell
kubectl port-forward service/produtos 3001:3001
```

Em outro terminal:

```powershell
curl http://localhost:3001/produtos
```

### 7. Demonstrar self-healing

```powershell
kubectl delete pod NOME_DO_POD
```

Depois:

```powershell
kubectl get pods -w
```

---

## 19. Fluxo simplificado

Com a imagem no Docker Hub, o fluxo necessário para uma máquina nova fica:

```text
1. Docker Desktop
       |
       v
2. Kubernetes ativo
       |
       v
3. docker compose up -d postgres
       |
       v
4. kubectl apply Deployment
       |
       v
5. Kubernetes baixa:
   terenciani/microservicos-produtos:1.0
       |
       v
6. Cria 3 Pods
       |
       v
7. kubectl apply Service
       |
       v
8. http://produtos:3001
   disponível dentro do cluster
```

Não é necessário:

```text
kind CLI
docker build
kind load docker-image
```

para executar uma imagem que já foi publicada no Docker Hub.

---

## 20. Arquitetura final desta etapa

```text
              Docker Hub
                  |
                  | pull
                  v
     terenciani/microservicos-produtos:1.0
                  |
                  v
+---------------------------------------------+
|              Kubernetes                     |
|                                             |
|           Deployment produtos               |
|              replicas: 3                    |
|                                             |
|      +----------+----------+                |
|      |          |          |                |
|      v          v          v                |
|    Pod 1      Pod 2      Pod 3              |
|      \          |          /                 |
|       \         |         /                  |
|        +--------+--------+                   |
|                 |                            |
|                 v                            |
|          Service produtos                    |
|             :3001                            |
+---------------------------------------------+
         |         |         |
         +---------+---------+
                   |
                   v
        host.docker.internal:5433
                   |
                   v
          PostgreSQL / Docker
```

## Próximo passo

A próxima etapa é colocar o microserviço **Pedidos** no Kubernetes.

Ele poderá utilizar o Service criado neste roteiro para acessar Produtos diretamente por:

```text
http://produtos:3001
```

Isso permitirá demonstrar a comunicação entre microserviços utilizando a descoberta de serviços do próprio Kubernetes.