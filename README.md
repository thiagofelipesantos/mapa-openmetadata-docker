# OpenMetadata: Ambiente Local

Este documento descreve a configuração e operação de um ambiente local para o projeto OpenMetadata, utilizando PostgreSQL como banco de dados principal e Elasticsearch para indexação e pesquisa, além de um serviço de Ingestão baseado em Airflow.

## Breve Descrição

Este setup utiliza o Docker Compose para orquestrar quatro serviços principais, criando uma infraestrutura local e isolada, ideal para desenvolvimento e testes do OpenMetadata. A arquitetura segue o passo a passo de configuração detalhado em [https://atlan.com/openmetadata-setup/](https://atlan.com/openmetadata-setup/ "null").

Para iniciar, clone o repositório que contém este arquivo de configuração:

```
git clone https://github.com/thiagofelipesantos/mapa-openmetadata-docker.git

```

## Estrutura dos Serviços

O ambiente é composto pelos seguintes serviços, volumes e redes:

### Serviços

| Serviço | Descrição | Componente Principal | Porta Exposta |
| --- | --- | --- | --- |
| **postgresql** | Banco de Dados Relacional. | PostgreSQL | `5432:5432` |
| **elasticsearch** | Motor de Busca e Indexação. | Elasticsearch | `9200:9200` |
| **openmetadata** | Aplicação Principal (API e UI). | OpenMetadata Server | `8585:8585` |
| **ingestion** | Serviço de Ingestão e Agendamento. | Airflow | `8080:8080` |

### Redes

*   **`app_net`**: Rede interna que conecta todos os serviços do _compose_, garantindo que eles possam se comunicar de forma isolada do _host_.
    

### Volumes (Comentados no Compose)

Embora os volumes nomeados (`ingestion-volume-dag-airflow`, `ingestion-volume-dags`, `ingestion-volume-tmp`, `es-data`) estejam comentados no arquivo, a intenção deles é fornecer **armazenamento persistente**.

*   O arquivo utiliza mapeamentos diretos do sistema de arquivos (`/home/thiago/volume_openmetadata/...`) para os containers, sobrescrevendo a necessidade dos volumes nomeados para persistência.
    

## Explicação Detalhada de Cada Serviço

Abaixo estão os detalhes de cada container, incluindo as configurações mais importantes definidas no arquivo `docker-compose.yml`.

### 1\. postgresql (Banco de Dados)

É o serviço de banco de dados relacional que armazena os metadados.

*   **Imagem:** `postgres:13.2`
    
*   **Nome do Container:** `openmetadata_postgres`
    
*   **Variáveis de Ambiente:**
    
    *   `POSTGRES_USER`: `openmetadata_user` (Nome do usuário do BD)
        
    *   `POSTGRES_PASSWORD`: `openmetadata_password` (Senha do usuário do BD)
        
    *   `POSTGRES_DB`: `openmetadata_db` (Nome do banco de dados)
        
*   **Portas:** Mapeia a porta interna `5432` para a porta `5432` da sua máquina host.
    
*   **Rede:** Associado à rede `app_net`.
    
*   **Condição:** É configurado para reiniciar a menos que seja explicitamente parado (`restart: unless-stopped`).
    
*   **Observação Importante:** A configuração `healthcheck` garante que o contêiner do PostgreSQL só estará pronto e disponível para outros serviços (como o OpenMetadata) após a inicialização do banco de dados ser concluída com sucesso.
    

### 2\. elasticsearch (Motor de Busca)

Este serviço é crucial para o recurso de busca e indexação do OpenMetadata.

*   **Imagem:** `docker.elastic.co/elasticsearch/elasticsearch:7.13.0`
    
*   **Nome do Container:** `openmetadata_elasticsearch`
    
*   **Variáveis de Ambiente:**
    
    *   `discovery.type`: `single-node` (Configura o ES para rodar em modo de nó único, ideal para desenvolvimento).
        
    *   `ES_JAVA_OPTS`: `-Xms512m -Xmx512m` (Limita o uso de memória Java a 512MB para otimizar o consumo de recursos).
        
*   **Portas:** Mapeia a porta interna `9200` para a porta `9200` da sua máquina host.
    
*   **Rede:** Associado à rede `app_net`.
    
*   **Dependência:** Garante que o serviço do Elasticsearch só inicie após o PostgreSQL estar **pronto** (`depends_on: {postgresql: {condition: service_healthy}}`).
    
*   **Configuração de Recursos:** O limite de memória (`mem_limit: 1g`) é definido para o container, limitando-o a 1GB de RAM.
    

### 3\. openmetadata (Servidor da Aplicação)

O coração da aplicação, contendo a API e a Interface do Usuário (UI).

*   **Imagem:** `openmetadata/openmetadata-server:0.12.1`
    
*   **Nome do Container:** `openmetadata_server`
    
*   **Variáveis de Ambiente:** Contém inúmeras variáveis que configuram a conexão com o PostgreSQL e Elasticsearch, além de definir o ambiente como "local".
    
*   **Portas:** Mapeia a porta interna `8585` para a porta `8585` da sua máquina host.
    
*   **Rede:** Associado à rede `app_net`.
    
*   **Dependência:** Só inicia após o PostgreSQL e o Elasticsearch estarem **prontos** (`depends_on`).
    
*   **Comando:** Executa o _script_ de inicialização principal (`./bin/start-server.sh`).
    

### 4\. ingestion (Serviço de Ingestão)

Este serviço é responsável por rodar os _jobs_ de ingestão de metadados, utilizando a estrutura do Airflow.

*   **Imagem:** `openmetadata/ingestion:0.12.1`
    
*   **Nome do Container:** `openmetadata_ingestion`
    
*   **Variáveis de Ambiente:** Configura a conexão com o servidor OpenMetadata principal (`OPENMETADATA_SERVER_URL: http://openmetadata:8585`).
    
*   **Portas:** Mapeia a porta interna `8080` (Webserver do Airflow) para a porta `8080` da sua máquina host.
    
*   **Rede:** Associado à rede `app_net`.
    
*   **Volumes Mapeados (ATENÇÃO!):** Os volumes nomeados originais foram substituídos por mapeamentos diretos (bind mounts) para caminhos específicos no host. **Nota:** O `docker-compose.yml` indica que estes mapeamentos foram **ALTERADOS** (`<--- ALTERADO`), utilizando caminhos específicos do host, como `/home/thiago/volume_openmetadata/...`.
    
    *   `dag_generated_configs`: Mapeado para `/opt/airflow/dag_generated_configs`
        
    *   `dags`: Mapeado para `/opt/airflow/dags`
        
    *   `tmp`: Mapeado para `/tmp`
        

## Dependências

1.  **`postgresql`** inicia primeiro.
    
2.  **`elasticsearch`** espera o `postgresql` estar com o _healthcheck_ aprovado (`service_healthy`).
    
3.  **`openmetadata`** espera tanto o `postgresql` quanto o `elasticsearch` estarem com _healthcheck_ aprovado.
    
4.  **`ingestion`** (Airflow) espera o `openmetadata` estar com _healthcheck_ aprovado.
    

## Como Executar o Projeto

Para subir todos os serviços e iniciar o ambiente OpenMetadata, utilize o Docker Compose no diretório onde o arquivo `docker-compose-postgres-comentado.yml` está localizado.

1.  **Pré-requisitos:** Certifique-se de que os seguintes _scripts_ auxiliares estejam disponíveis no seu ambiente:
    
    *   **`instalar.openmetadata.sh`**: Responsável por configurar o ambiente inicial, **verificando a presença de dependências essenciais** como Docker, Python e Git. Deve ser executado antes de qualquer outra etapa.
        
    *   **`exec_logar_arqSh.sh`**: Um executor padrão do Shell que é usado para **logar a execução de outros scripts**.
        
2.  **Abra o terminal** no diretório do arquivo.
    
3.  **Execute o comando:**
    

<!-- end list -->

```
docker compose -f docker-compose-postgres-comentado.yml up -d
```

*   O argumento `-f` especifica o arquivo YAML a ser usado.
    
*   O argumento `-d` (detach) executa os containers em segundo plano.
    

## Como Parar o Projeto

Para parar e remover todos os containers, volumes anônimos e redes criadas por este _compose_, utilize o comando `down`:

1.  **Abra o terminal** no diretório do arquivo.
    
2.  **Execute o comando:**
    

<!-- end list -->

```
docker compose -f docker-compose-postgres-comentado.yml down
```

## Como Acessar os Serviços

Após executar o comando `up -d`, você pode acessar os serviços através das portas mapeadas na sua máquina local:

| Serviço | URL de Acesso |
| --- | --- |
| **OpenMetadata UI** | `http://localhost:8585` |
| **Airflow Webserver (Ingestion)** | `http://localhost:8080` |
| **Elasticsearch** | `http://localhost:9200` |
| **PostgreSQL** | Acessível via cliente BD na porta `5432` |

## Provisionamento de Hardware

O ambiente tem restrições de recursos definidas para o serviço **Elasticsearch**:

*   **Limite de Memória (RAM):** O container do `elasticsearch` está limitado a **1GB de RAM** (`mem_limit: 1g`).
    
*   **Opções Java:** O heap de memória Java para o Elasticsearch (`ES_JAVA_OPTS`) está fixado em **512MB** para alocação inicial (`-Xms`) e máxima (`-Xmx`).
    

Recomenda-se que o sistema host tenha memória RAM suficiente para acomodar esses limites, além dos recursos necessários para o OpenMetadata (que não tem limites de hardware explícitos no _compose_).

## Requisitos de Hardware

Para o dimensionamento adequado da infraestrutura, considere as seguintes recomendações:

### Requisitos Mínimos (Desenvolvimento/Teste)

Para fins de desenvolvimento ou avaliação inicial, os requisitos são mais modestos:

*   **Memória RAM:** 8 GB.
    
*   **Espaço em Disco:** 20 GB.
    
*   **Sistemas Operacionais:** GNU/Linux, Windows ou macOS.
    

### Requisitos para Produção (Recomendado)

Para uma implantação em produção, que exige maior robustez e desempenho, as especificações recomendadas são:

*   **vCPUs:** Mínimo de 4 vCPUs.
    
*   **Memória RAM:** 16 GiB.
    
*   **Volume de Armazenamento:** 100 GiB (o tipo de armazenamento, como SSD, impactará o desempenho).
    

É importante notar que o OpenMetadata é uma plataforma que pode ser implantada em diferentes arquiteturas (como Kubernetes), e o dimensionamento final dependerá da carga de trabalho esperada e da quantidade de fontes de dados a serem integradas.

## Observações Importantes

1.  **Versão do Compose:** A versão do formato do arquivo Compose (`version: "3.9"`) está **desativada (comentada)**. Isso significa que a execução usará a versão padrão/implícita do seu Docker Compose (geralmente a mais recente compatível).
    
2.  **Volumes:** O arquivo usa mapeamentos de host (**bind mounts**) em vez de volumes nomeados. Isso significa que a persistência dos dados de ingestão (DAGs e _configs_ geradas) depende dos diretórios `/home/thiago/volume_openmetadata/...` existirem no sistema de arquivos do seu host e serem acessíveis ao Docker.
    
3.  **Modo Elasticsearch:** O Elasticsearch está configurado para operar em modo de **nó único** (`discovery.type: single-node`), o que é o padrão e ideal para ambientes de desenvolvimento ou teste locais.
    
4.  **Segurança da UI:** A interface web do OpenMetadata está exposta, e as credenciais iniciais de administrador (padrão) devem ser gerenciadas imediatamente.
    

## Alerta sobre Segurança

**ATENÇÃO: Este `docker-compose.yml` está configurado para um ambiente de desenvolvimento local, utilizando senhas e usuários padrão e com portas expostas publicamente (`localhost`).**

*   **Credenciais Padrão:** As senhas e usuários (`openmetadata_user`, `openmetadata_password`, etc.) estão visíveis no arquivo. **NUNCA** utilize esta configuração em ambientes de produção.
    
*   **Portas Expostas:** As portas dos serviços (`5432`, `9200`, `8585`, `8080`) estão abertas para o `localhost`. Em ambientes de produção, restrinja o acesso a estas portas.
    
*   **Verificação de Segurança:** É importante notar que este ambiente básico de desenvolvimento não inclui mecanismos automáticos de varredura ou verificação contra a presença de scripts maliciosos ou vulnerabilidades que possam tentar capturar dados. A segurança deve ser tratada como prioridade ao mover para qualquer ambiente que não seja local.
    

Para ambientes reais, é crucial utilizar variáveis de ambiente para credenciais, _secrets_ do Docker e configurar regras de firewall rigorosas.
