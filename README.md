# Estágio de Construção da Imagem Docker

Este estágio `docker_build` no pipeline GitLab CI/CD é responsável por criar uma imagem Docker do projeto, verificando a versão na mensagem de commit e armazenando a imagem no repositório Nexus.

## Funcionalidades Principais

- **Construção de Imagem Docker**: Criação de uma imagem Docker a partir dos arquivos do repositório.
- **Validação de Versionamento**: Verificação do padrão de versionamento na mensagem de commit.
- **Integração com Nexus**: Login e envio da imagem para o repositório Nexus.

## Requisitos

### Variáveis de Ambiente

Configurar as variáveis de ambiente abaixo no GitLab para o correto funcionamento do estágio `docker_build`.

| Variável            | Descrição                                   |
|---------------------|---------------------------------------------|
| `PROXY_DOCKER`      | URL do proxy HTTP para o Docker.            |
| `NEXUS_USERNAME_DEV`| Nome de usuário para autenticação no Nexus. |
| `NEXUS_PASSWORD_DEV`| Senha de autenticação no Nexus.             |
| `NEXUS_URL_DEV`     | URL do repositório Nexus.                   |
| `CI_PROJECT_NAME`   | Nome do projeto, gerado automaticamente pelo GitLab. |

## Explicação do Pipeline

O estágio `docker_build` executa as seguintes etapas:

1. **Configuração do Proxy**: Define as variáveis `http_proxy` e `https_proxy` para o container Docker.

2. **Preparação do Ambiente**:
   - Instala pacotes necessários (`ca-certificates`, `curl`, `git`) no container.

3. **Verificação e Extração da Versão**:
   - Obtém a última mensagem de commit e verifica se contém uma versão no formato `X.X.X`.
   - Caso a versão não esteja presente, o pipeline é interrompido.
   - Armazena a versão extraída na variável `VERSION`.

4. **Login no Nexus**:
   - Realiza o login no Nexus usando as credenciais configuradas para permitir o push da imagem Docker.

5. **Build da Imagem Docker**:
   - Cria a imagem Docker e a nomeia no formato `${NEXUS_URL_DEV}/dsis-1/${CI_PROJECT_NAME}:${VERSION}`.

## Exemplo de Uso

### Pipeline GitLab CI/CD - Exemplo de `docker_build`:

```yaml
docker_build:
  stage: docker_build
  before_script: []                    # Campo vazio para ignorar configurações globais.
  image: docker:stable                 # Imagem para o container que fará o deploy.
  services:
    - docker:stable-dind
  script:
    - export http_proxy=$PROXY_DOCKER  # Define o proxy para o container.
    - export https_proxy=$PROXY_DOCKER # Define o proxy para o container.
    - apk update
    - apk add --no-cache ca-certificates curl git
    - |
      # Obtém a última mensagem de commit
      COMMIT_MSG=$(git log -1 --pretty=%B)  
      # Define o padrão para a versão (ex.: "Version: 1.0.0" ou "VERSION: 1.0.0")
      VERSION_PATTERN="^version: [0-9]+\.[0-9]+\.[0-9]+"
      # Verifica se a mensagem contém a versão no padrão esperado
      if ! echo "$COMMIT_MSG" | grep -iEq "$VERSION_PATTERN"; then
        echo "Error: Commit message must include a version number in the format 'Version: X.X.X'"
        exit 1
      fi  
      # Extrai a versão da mensagem de commit
      VERSION=$(echo "$COMMIT_MSG" | grep -ioE "[0-9]+\.[0-9]+\.[0-9]+")
      # Exibe a versão para confirmar
      echo "Version found: $VERSION"
      # Exporta a variável para que envsubst possa acessá-la
      export VERSION
    - echo "Login Nexus"
    - docker login -u $NEXUS_USERNAME_DEV -p $NEXUS_PASSWORD_DEV $NEXUS_URL_DEV  # Login no Nexus
    - echo "Login realizado"
    - echo "Build Docker"
    - docker build -t $NEXUS_URL_DEV/dsis-1/${CI_PROJECT_NAME}:$VERSION .        # Faz o build da imagem
    - echo "Fim build"
  tags:
    - docker
  allow_failure: false
