# caixa-sugestoes-web-etec

Projeto web Spring Boot + Thymeleaf criado a partir das instruções de `caixa-sugestoes-web/caixa-sugestoes-web-tutorial`, reproduzindo a integração com a API Spring Boot.

## Tutorial de reprodução em 3 partes

Os alunos já desenvolveram a API em outro tutorial. Este roteiro reconstrói o frontend de forma simplificada, do protótipo estático até a integração final:

| Parte | Documento | Entrega | Verificação |
|---|---|---|---|
| 1 | [Interface estática — HTML, CSS e JS](docs/tutorial/parte-1-interface-statica.md) | Seis telas em `static/prototipo/`, com dados fixos e nenhum `th:` | Navegar pelas seis telas em `http://localhost:8081/prototipo/index.html` |
| 2 | [Adaptação para Thymeleaf](docs/tutorial/parte-2-thymeleaf.md) | Nove templates em `templates/`, fragments, expressões e formulários vinculados | Renderizar com controller mock e conferir o HTML sem `th:` |
| 3 | [Controllers e integração com a API](docs/tutorial/parte-3-integracao-api.md) | `RestClient`, modelos, sessão administrativa e rotas finais | Criar, filtrar, detalhar, login, editar e excluir com a API em execução |

Pré-requisitos para todo o percurso:

- JDK 17 e o wrapper Gradle incluído (`./gradlew` ou `.\gradlew.bat`);
- API disponível em `http://localhost:8000`, com Swagger UI em <http://localhost:8000/swagger-ui/index.html>;
- `API_BASE_URL` apontando para a base da API, **sem** o sufixo `/api`.

As partes são cumulativas: conclua a Parte 1 antes da Parte 2 e a Parte 2 antes da Parte 3. Cada documento traz blocos de código completos, um checkpoint e uma seção de erros comuns. O código deste repositório é a referência final; os documentos usam versões reduzidas para facilitar a reprodução em sala. A Parte 2 é a mais detalhada porque concentra a sintaxe de Thymeleaf.

## Integração com a API

O frontend chama a API no servidor Java via `RestClient`, sem acesso direto ao banco. O cliente já inclui o prefixo `/api`, então **não** acrescente `/api` em `API_BASE_URL`.

- `app.api.base-url`: configurado por `API_BASE_URL`; padrão `http://localhost:8000`.
- `server.port`: configurado por `WEB_PORT`; padrão `8081`.

Se a API estiver em outra porta, por exemplo `8081`, inicie com `API_BASE_URL=http://localhost:8081`. Observe que este web app também usa `8081` por padrão; use `WEB_PORT` para evitar conflito.

## Executar

Use JDK 17 e o wrapper Gradle incluído:

```bash
# API em http://localhost:8000
./gradlew bootRun

# API em outra porta / web em outra porta
API_BASE_URL=http://localhost:8081 WEB_PORT=8082 ./gradlew bootRun
```

## Build/test

```bash
./gradlew clean build
./gradlew test
```

## Estrutura

- `src/main/java/br/com/e24dev/caixaweb/controller/`: rotas web.
- `src/main/java/br/com/e24dev/caixaweb/client/`: `CaixaApiClient` e `ApiException`.
- `src/main/java/br/com/e24dev/caixaweb/model/`: modelos JSON/formulários.
- `src/main/java/br/com/e24dev/caixaweb/service/`: sessão administrativa.
- `src/main/java/br/com/e24dev/caixaweb/config/`: `RestClient`, segurança e filtros.
- `src/main/resources/templates/`: templates Thymeleaf.
- `src/main/resources/application.yaml`: configuração.

## Endpoints consumidos

Contrato HTTP da API: `/api/courses`, `/api/categories`, `/api/suggestions`, `/api/suggestions/{id}`, `POST /api/suggestions`, `POST /api/auth/login`, `PUT /api/suggestions/{id}`, `DELETE /api/suggestions/{id}`. Não invente campos, IDs ou endpoints; a collection Insomnia entregue com a API é a referência.

## Segurança

JWTs ficam exclusivamente em sessão HTTP server-side; CSRF protegido nos formulários POST. Não versione credenciais/tokens.
