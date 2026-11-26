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

## API 

Vamos utilizar uma versão já compilada da API para este tutorial, baixe-a [aqui](https://drive.google.com/file/d/1uJjRagzlUT68tcb94w7Y8npzgZ5Ep5yB/view?usp=sharing).

### Executar a API

Para que todos possam focar na integração da API, não será necessário executar o projeto que voce estava desenvolvendo, iremos utilizar uma aplicação já compilada da API, basta apenas executar-la.

> Para executar a API no seu próprio computador execute o comando abaixo

```shell
java -jar build/libs/caixa-sugestoes-web-etec-0.1.0.jar --server.address=0.0.0.0
```

> Para testar na API de outro colega, substitua o IP acima pelo IP do computador de seu colega de classe. Faça este teste apenas em laboratório. Esta
