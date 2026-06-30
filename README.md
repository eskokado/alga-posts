# AlgaPosts

Sistema distribuído para publicação e processamento assíncrono de posts de texto. O projeto é composto por dois microsserviços que se comunicam via **RabbitMQ**, permitindo criar posts, consultar detalhes e listar publicações com paginação enquanto o processamento de conteúdo ocorre em segundo plano.

## Visão geral

O **AlgaPosts** recebe publicações de texto (posts), envia o conteúdo para processamento assíncrono e armazena o resultado quando disponível. O processamento inclui:

- Contagem de palavras no corpo do texto
- Cálculo de valor estimado (`palavras × R$ 0,10`)

```text
┌─────────────┐     REST API      ┌──────────────┐
│   Cliente   │ ────────────────► │ post-service │──┐
└─────────────┘                   └──────────────┘  │
                                                    │ RabbitMQ
                    ┌───────────────────────────────┘
                    ▼
         text-processor-service.post-processing.v1.q
                    │
                    ▼
         ┌──────────────────────┐
         │ text-processor-service│
         └──────────────────────┘
                    │
                    ▼
         post-service.post-processing-result.v1.q
                    │
                    ▼
         ┌──────────────┐
         │ post-service │ ──► H2 (persistência)
         └──────────────┘
```

## Microsserviços

| Serviço | Porta | Responsabilidade |
|---------|-------|------------------|
| [post-service](microservices/post-service) | `8080` | API REST, persistência H2, envio e consumo de mensagens |
| [text-processor-service](microservices/text-processor-service) | `8081` | Processamento assíncrono de texto via RabbitMQ |

## Tecnologias

- Java 21
- Spring Boot 3.5
- Spring Data JPA + H2
- Spring AMQP (RabbitMQ)
- Docker Compose
- Gradle

## Pré-requisitos

- Java 21+
- Docker e Docker Compose
- Git (com suporte a submódulos)

## Clonando o repositório

```bash
git clone --recurse-submodules git@github.com:eskokado/alga-posts.git
cd alga-posts
```

Se já clonou sem submódulos:

```bash
git submodule update --init --recursive
```

## Infraestrutura local

Subir o RabbitMQ (obrigatório para o fluxo completo):

```bash
docker compose up -d algaposts-rabbitmq
```

| Serviço | URL | Credenciais |
|---------|-----|-------------|
| RabbitMQ AMQP | `localhost:5672` | `rabbitmq` / `rabbitmq` |
| RabbitMQ Management | http://localhost:15672 | `rabbitmq` / `rabbitmq` |

O `docker-compose.yml` também inclui SonarQube e PostgreSQL para análise de código.

## Executando os microsserviços

Em terminais separados:

```bash
# 1. Text Processor
cd microservices/text-processor-service
./gradlew bootRun

# 2. Post Service
cd microservices/post-service
./gradlew bootRun
```

## Testando a API

### Postman

Importe os arquivos em `postman/`:

- `AlgaPosts.postman_collection.json`
- `AlgaPosts-Local.postman_environment.json`

### Exemplo com cURL

```bash
# Criar post
curl -X POST http://localhost:8080/api/posts \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Meu primeiro post",
    "body": "Linha um\nLinha dois\nLinha tres",
    "author": "Eskokado"
  }'

# Consultar post (aguarde alguns segundos para wordCount e calculatedValue)
curl http://localhost:8080/api/posts/{postId}

# Listar posts paginados
curl "http://localhost:8080/api/posts?page=0&size=10"
```

> **Nota:** `wordCount` e `calculatedValue` são preenchidos de forma assíncrona e podem não aparecer imediatamente após a criação.

## Filas RabbitMQ

| Fila | Produtor | Consumidor |
|------|----------|------------|
| `text-processor-service.post-processing.v1.q` | post-service | text-processor-service |
| `post-service.post-processing-result.v1.q` | text-processor-service | post-service |

Cada fila possui uma DLQ (Dead Letter Queue) configurada via Java.

## Estrutura do repositório

```text
alga-posts/
├── configs/rabbitmq/          # Imagem customizada do RabbitMQ
├── docker-compose.yml         # Infraestrutura local
├── microservices/
│   ├── post-service/          # Submódulo — API REST
│   └── text-processor-service/ # Submódulo — processamento assíncrono
└── postman/                   # Collection e environment
```

## Licença

Projeto desenvolvido para fins educacionais (Algaworks).
