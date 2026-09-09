# Publicação de Imagem no Docker Hub

Este roteiro mostra como construir uma imagem Docker localmente e publicá-la no Docker Hub.

O processo é independente de Kubernetes.

O fluxo geral é:

    Código-fonte
        |
        v
    Dockerfile
        |
        v
    docker build
        |
        v
    Imagem local
        |
        v
    docker tag
        |
        v
    docker push
        |
        v
    Docker Hub


## 1. Pré-requisitos

É necessário ter:

- Docker instalado e funcionando;
- uma conta no Docker Hub;
- um `Dockerfile` válido no projeto.

Verifique o Docker:

    docker --version

Também é recomendável verificar se o Docker está funcionando:

    docker ps


## 2. Fazer login no Docker Hub

No terminal:

    docker login

O Docker solicitará a autenticação.

Após o login, a máquina estará autorizada a publicar imagens na sua conta.


## 3. Construir a imagem

Supondo que o projeto esteja dentro da pasta:

    produtos

e que exista um `Dockerfile` dentro dela, execute:

    docker build -t microservicos-produtos:latest ./produtos

Onde:

- `docker build` cria uma imagem;
- `-t` define o nome e a tag;
- `microservicos-produtos` é o nome da imagem;
- `latest` é a tag;
- `./produtos` é o diretório utilizado como contexto de build.


## 4. Verificar a imagem criada

Execute:

    docker images

Deverá aparecer algo semelhante a:

    REPOSITORY                 TAG       IMAGE ID       CREATED
    microservicos-produtos     latest    abc123...      ...


## 5. Criar uma tag no padrão do Docker Hub

Para publicar no Docker Hub, a imagem precisa utilizar o formato:

    USUARIO/NOME_DA_IMAGEM:TAG

Neste projeto, o usuário utilizado é:

    terenciani

Portanto:

    docker tag microservicos-produtos:latest terenciani/microservicos-produtos:1.0

Esse comando não cria uma nova imagem completa.

Ele cria uma nova referência para a mesma imagem.


## 6. Conferir as tags

Execute novamente:

    docker images

Você deverá ver algo semelhante a:

    REPOSITORY                              TAG       IMAGE ID
    microservicos-produtos                  latest    abc123
    terenciani/microservicos-produtos       1.0       abc123

Observe que o `IMAGE ID` é o mesmo.

Isso indica que ambas as tags apontam para a mesma imagem.


## 7. Publicar no Docker Hub

Execute:

    docker push terenciani/microservicos-produtos:1.0

O Docker enviará as camadas da imagem para o Docker Hub.

Ao final, a imagem estará disponível no repositório:

    terenciani/microservicos-produtos


## 8. Testar o download da imagem

Para verificar se a publicação funcionou, é possível remover a imagem local e baixá-la novamente.

Primeiro:

    docker pull terenciani/microservicos-produtos:1.0

Se o download ocorrer normalmente, a imagem está disponível no Docker Hub.


## 9. Executar a imagem diretamente do Docker Hub

Depois de publicada, qualquer computador com Docker pode executar:

    docker pull terenciani/microservicos-produtos:1.0

Depois:

    docker run terenciani/microservicos-produtos:1.0

Caso seja necessário mapear uma porta:

    docker run -p 3001:3001 terenciani/microservicos-produtos:1.0

Assim:

    localhost:3001
        |
        v
    container:3001


## 10. Publicar uma nova versão

Se o código da aplicação for alterado, faça um novo build.

Por exemplo:

    docker build -t microservicos-produtos:latest ./produtos

Depois crie uma nova tag:

    docker tag microservicos-produtos:latest terenciani/microservicos-produtos:1.1

Publique:

    docker push terenciani/microservicos-produtos:1.1

Agora existirão versões diferentes da imagem:

    terenciani/microservicos-produtos:1.0
    terenciani/microservicos-produtos:1.1


## 11. Atualizar a tag latest

Também é possível publicar uma versão como `latest`.

Crie a tag:

    docker tag microservicos-produtos:latest terenciani/microservicos-produtos:latest

Depois:

    docker push terenciani/microservicos-produtos:latest

Assim, o Docker Hub terá, por exemplo:

    terenciani/microservicos-produtos:1.0
    terenciani/microservicos-produtos:1.1
    terenciani/microservicos-produtos:latest


## 12. Diferença entre nome local e nome do Docker Hub

A imagem pode existir localmente como:

    microservicos-produtos:latest

Mas, para ser publicada na sua conta do Docker Hub, ela precisa ter o prefixo do usuário:

    terenciani/microservicos-produtos:1.0

Por isso utilizamos:

    docker tag


## 13. Comandos principais

Login:

    docker login

Construir imagem:

    docker build -t microservicos-produtos:latest ./produtos

Listar imagens:

    docker images

Criar tag:

    docker tag microservicos-produtos:latest terenciani/microservicos-produtos:1.0

Publicar:

    docker push terenciani/microservicos-produtos:1.0

Baixar:

    docker pull terenciani/microservicos-produtos:1.0

Executar:

    docker run -p 3001:3001 terenciani/microservicos-produtos:1.0


## 14. Fluxo resumido

Primeira publicação:

    docker login

    docker build -t microservicos-produtos:latest ./produtos

    docker tag microservicos-produtos:latest terenciani/microservicos-produtos:1.0

    docker push terenciani/microservicos-produtos:1.0


Nova versão:

    docker build -t microservicos-produtos:latest ./produtos

    docker tag microservicos-produtos:latest terenciani/microservicos-produtos:1.1

    docker push terenciani/microservicos-produtos:1.1


## 15. Conceito importante

O Docker Hub funciona como um registry de imagens.

Ele não armazena o código-fonte do projeto da mesma maneira que o GitHub.

A diferença pode ser resumida assim:

    GitHub
      |
      +-- código-fonte
      +-- Dockerfile
      +-- documentação

    Docker Hub
      |
      +-- imagens Docker prontas para execução

Ou seja:

    GitHub -> código

    Docker Hub -> imagem construída