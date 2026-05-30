# Gateway-Service

Ponto de entrada único para todas as requisições externas. Roteia chamadas para os microsserviços, valida tokens JWT antes de encaminhar requisições protegidas, enriquece requests com contexto do usuário e gerencia CORS.

## Tecnologias

- Java 25
- Spring Boot 4.0.3
- Spring Cloud 2025.1.0 (Spring Cloud Gateway + WebFlux)
- Micrometer / Prometheus
- Lombok

## Rotas

| Path | Destino | Observações |
|------|---------|-------------|
| `/accounts/**` | `http://account:8080` | Gestão de contas |
| `/auth/**` | `http://auth:8080` | Autenticação |
| `/products/**` | `http://product:8080` | Catálogo de produtos |
| `/insper/**` | `https://www.insper.edu.br` | Proxy externo |

## Autorização

O filtro global (`AuthorizationFilter`) é executado em todas as requisições:

**Endpoints públicos (sem token):**
- `POST /auth/register`
- `POST /auth/login`

**Demais endpoints (token obrigatório):**
1. Extrai o JWT do cookie `__store_jwt_token`
2. Chama `POST /auth/solve` no `auth-service` para validar o token
3. Se válido: injeta headers na requisição e encaminha ao serviço de destino
4. Se inválido ou ausente: retorna `401 Unauthorized`

**Headers injetados nos requests autenticados:**

| Header | Valor |
|--------|-------|
| `id-account` | ID da conta extraído do token |
| `role` | Perfil do usuário extraído do token |
| `Authorization` | `Bearer <token>` |

## CORS

Configurado globalmente para todos os paths (`/**`):

| Configuração | Valor |
|--------------|-------|
| `allowedOrigins` | `${CORS_ALLOWED_ORIGINS}` |
| `allowedHeaders` | `*` |
| `allowedMethods` | `*` |
| `allowCredentials` | `${CORS_ALLOWED_CREDENTIALS}` |

## Variáveis de ambiente

| Variável | Descrição | Padrão |
|----------|-----------|--------|
| `CORS_ALLOWED_ORIGINS` | Origins permitidas pelo CORS | `*` |
| `CORS_ALLOWED_CREDENTIALS` | Permite envio de credenciais no CORS | `true` |

## Executando localmente

### Com Docker (recomendado)

O gateway é implantado com **3 réplicas** atrás de um Nginx na porta `8080`:

```bash
docker compose up
```

### Build manual

```bash
mvn clean package -DskipTests
java -jar target/gateway-service-0.0.1-SNAPSHOT.jar
```

O serviço sobe na porta `8080`.

## Métricas

Prometheus disponível em `/gateway/actuator/prometheus`.

## Comunicação entre serviços

| Tipo | Serviço | Detalhe |
|------|---------|---------|
| HTTP (saída) | `auth-service` | Chama `/auth/solve` para validar tokens JWT |
| Proxy (saída) | `account-service` | Encaminha requisições `/accounts/**` |
| Proxy (saída) | `auth-service` | Encaminha requisições `/auth/**` |
| Proxy (saída) | `product-service` | Encaminha requisições `/products/**` |
