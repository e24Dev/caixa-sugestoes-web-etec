# Parte 3 — Integração com controllers e com a API

[Voltar ao README](../../README.md) · [Anterior: Parte 2 — Thymeleaf](parte-2-thymeleaf.md)

## Objetivo

Substituir o controller mock da Parte 2 pelo controller real e fazer as nove templates conversarem com a API que já existe. Nesta etapa:

- o frontend **não acessa banco de dados**;
- todo dado vem de HTTP via `RestClient`;
- o JWT recebido no login fica somente na sessão HTTP do servidor;
- PUT e DELETE levam `Authorization: Bearer <token>`;
- as templates permanecem exatamente como estão.

Pré-requisitos:

- API em execução em `http://localhost:8000`;
- Swagger UI em <http://localhost:8000/swagger-ui/index.html> (o JSON do contrato está em <http://localhost:8000/v3/api-docs>);
- JDK 17 e o wrapper Gradle do projeto;
- Parte 2 concluída, com `static/prototipo/` apagado.

> O tutorial da API já foi feito em outro material. Aqui a API é tratada como **serviço pronto**: leia o Swagger, use os campos documentados e não crie endpoints novos.

## Passo 1 — contrato consumido pela aplicação

| Método e caminho | Uso na aplicação web | Corpo / parâmetros | Resposta importante |
|---|---|---|---|
| `GET /api/courses` | popular `<select>` de cursos | — | `[{ "id": 1, "name": "..." }]` |
| `GET /api/categories` | popular `<select>` de categorias | — | `[{ "id": 1, "name": "..." }]` |
| `GET /api/suggestions` | mural e painel | `courseId`, `categoryId`, `q` opcionais | array de sugestões, pode ser vazio |
| `GET /api/suggestions/{id}` | página de detalhes e pré-carga da edição | ID positivo na URL | uma sugestão; `404` se não existir |
| `POST /api/suggestions` | publicar sugestão | `title`, `content`, `author?`, `courseId`, `categoryId` | `201` com a sugestão criada |
| `POST /api/auth/login` | autenticar administrador | `username`, `password` | `{ accessToken, tokenType, expiresIn }` |
| `PUT /api/suggestions/{id}` | salvar edição | mesmos campos de `POST`, mais `Bearer` | sugestão atualizada |
| `DELETE /api/suggestions/{id}` | excluir sugestão | `Bearer`, sem corpo | `204` sem corpo |

Formato da sugestão devolvida pela API:

```json
{
  "id": 1,
  "title": "Mais tomadas na biblioteca",
  "content": "Instalar tomadas nas mesas de estudo.",
  "author": "Anônimo",
  "createdAt": "2026-10-08T12:00:00Z",
  "course": { "id": 1, "name": "Desenvolvimento de Sistemas" },
  "category": { "id": 1, "name": "Infraestrutura" }
}
```

Regras do contrato que já estão documentadas no Swagger:

- `courseId` e `categoryId` devem vir das listas de catálogo; não invente IDs;
- `q` busca título **ou** conteúdo, sem diferenciar maiúsculas, e combina com os outros filtros por AND;
- a listagem não tem paginação e vem ordenada por `createdAt` e `id` decrescentes;
- IDs positivos inexistentes em filtros devolvem `[]`, não erro;
- autor ausente, nulo ou em branco é devolvido como `Anônimo`;
- `PUT` preserva `id` e `createdAt`; omitir `author` remove o nome anterior;
- `DELETE` é físico e repeti-lo devolve `404`;
- erros seguem o formato `{ timestamp, status, message, path, fieldErrors }`.

Use as credenciais fornecidas pelo professor para o login. Não há cadastro público de administrador nesta aplicação.

## Passo 2 — configuração

### `src/main/resources/application.yaml`

Substitua pelo conteúdo final:

```yaml
server:
  port: ${WEB_PORT:8081}
  servlet:
    session:
      cookie:
        http-only: true
        same-site: lax
spring:
  application:
    name: caixa-sugestoes-web-etec
  thymeleaf:
    cache: false
app:
  api:
    base-url: ${API_BASE_URL:http://localhost:8000}
```

- `WEB_PORT` define a porta do web app; padrão `8081`.
- `API_BASE_URL` é a **base da API**, sem `/api`. O cliente adiciona `/api/...` em cada chamada.
- `http-only: true` impede que JavaScript leia o cookie de sessão.
- `same-site: lax` reduz o risco de envio do cookie em requisições externas.

```bash
# API em http://localhost:8000, web em 8081
./gradlew bootRun

# API em 8081 e web em 8082, para evitar conflito de porta
API_BASE_URL=http://localhost:8081 WEB_PORT=8082 ./gradlew bootRun
```

Windows PowerShell:

```powershell
.\gradlew.bat bootRun
$env:API_BASE_URL = 'http://localhost:8081'
$env:WEB_PORT = '8082'
.\gradlew.bat bootRun
```

### `config/HttpClientConfig.java`

Crie o bean do `RestClient` com a base URL configurada:

```java
package br.com.e24dev.caixaweb.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.*;
import org.springframework.web.client.RestClient;

@Configuration
public class HttpClientConfig {
    @Bean
    RestClient apiRestClient(@Value("${app.api.base-url}") String baseUrl) {
        return RestClient.builder().baseUrl(baseUrl).build();
    }
}
```

Como o bean tem `baseUrl`, os caminhos passados depois podem ser relativos: `/api/courses`, `/api/suggestions` etc.

## Passo 3 — modelos finais

`CatalogItem`, `Suggestion`, `LoginForm` e `SuggestionForm` já foram criados na Parte 2. Faltam dois ajustes.

### `model/LoginResponse.java`

```java
package br.com.e24dev.caixaweb.model;

public record LoginResponse(String accessToken, String tokenType, long expiresIn) {
}
```

Os nomes dos componentes do record batem com as chaves do JSON do Swagger: `accessToken`, `tokenType`, `expiresIn`.

### `model/SuggestionForm.java` atualizado

Acrescente o método estático `from(...)` ao arquivo da Parte 2:

```java
public static SuggestionForm from(Suggestion s) {
    SuggestionForm f = new SuggestionForm();
    f.setTitle(s.title());
    f.setContent(s.content());
    f.setAuthor(s.author());
    f.setCourseId(s.course().id());
    f.setCategoryId(s.category().id());
    return f;
}
```

Ele converte a resposta da API (`Suggestion`) no objeto que o formulário de edição usa (`SuggestionForm`). Repare que `course` e `category` viram apenas os IDs, pois são os campos editáveis.

## Passo 4 — cliente HTTP e tradução de erros

### `client/ApiException.java`

```java
package br.com.e24dev.caixaweb.client;

public class ApiException extends RuntimeException {
    private final int status;

    public ApiException(int status, String message) {
        super(message);
        this.status = status;
    }

    public int status() {
        return status;
    }
}
```

A exceção carrega o status HTTP para o controller decidir: `401` encerra a sessão, `404` vai para a página de erro, e assim por diante.

### `client/CaixaApiClient.java`

```java
package br.com.e24dev.caixaweb.client;

import br.com.e24dev.caixaweb.model.*;

import java.util.*;

import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.HttpHeaders;
import org.springframework.stereotype.Component;
import org.springframework.web.client.*;
import org.springframework.web.util.UriComponentsBuilder;

@Component
public class CaixaApiClient {
    private final RestClient client;

    public CaixaApiClient(RestClient apiRestClient) {
        this.client = apiRestClient;
    }

    private ApiException map(RestClientResponseException e) {
        return new ApiException(e.getStatusCode().value(),
                switch (e.getStatusCode().value()) {
                    case 400 -> "Verifique os dados informados.";
                    case 401 -> "Sessão inválida ou expirada. Entre novamente.";
                    case 403 -> "Você não tem permissão para realizar esta operação.";
                    case 404 -> "Registro não encontrado.";
                    default -> "Não foi possível concluir a operação (HTTP " + e.getStatusCode().value() + ").";
                });
    }

    private ApiException offline(RestClientException e) {
        return new ApiException(503, "Não foi possível conectar à API. Verifique se ela está em execução.");
    }

    public List<CatalogItem> courses() {
        return catalog("/api/courses");
    }

    public List<CatalogItem> categories() {
        return catalog("/api/categories");
    }

    private List<CatalogItem> catalog(String path) {
        try {
            return Optional.ofNullable(
                    client
                            .get()
                            .uri(path)
                            .retrieve()
                            .body(new ParameterizedTypeReference<List<CatalogItem>>() {
                            })
            ).orElse(List.of());
        } catch (RestClientResponseException e) {
            throw map(e);
        } catch (RestClientException e) {
            throw offline(e);
        }
    }

    public List<Suggestion> suggestions(Long courseId, Long categoryId, String query) {
        try {
            String uri = UriComponentsBuilder.fromPath("/api/suggestions")
                    .queryParamIfPresent("courseId", Optional.ofNullable(courseId))
                    .queryParamIfPresent("categoryId", Optional.ofNullable(categoryId))
                    .queryParamIfPresent("q", Optional.ofNullable(query).filter(q -> !q.isBlank()))
                    .build().encode().toUriString();
            return Optional.ofNullable(client.get().uri(uri).retrieve().body(new ParameterizedTypeReference<List<Suggestion>>() {
            })).orElse(List.of());
        } catch (RestClientResponseException e) {
            throw map(e);
        } catch (RestClientException e) {
            throw offline(e);
        }
    }

    public Suggestion suggestion(Long id) {
        try {
            return client.get().uri("/api/suggestions/{id}", id).retrieve().body(Suggestion.class);
        } catch (RestClientResponseException e) {
            throw map(e);
        } catch (RestClientException e) {
            throw offline(e);
        }
    }

    public Suggestion create(SuggestionForm form) {
        try {
            return client.post().uri("/api/suggestions").body(form).retrieve().body(Suggestion.class);
        } catch (RestClientResponseException e) {
            throw map(e);
        } catch (RestClientException e) {
            throw offline(e);
        }
    }

    public LoginResponse login(String username, String password) {
        try {
            return client.post().uri("/api/auth/login").body(Map.of("username", username, "password", password)).retrieve().body(LoginResponse.class);
        } catch (RestClientResponseException e) {
            throw map(e);
        } catch (RestClientException e) {
            throw offline(e);
        }
    }

    public Suggestion update(Long id, SuggestionForm form, String jwt) {
        try {
            return client.put().uri("/api/suggestions/{id}", id).header(HttpHeaders.AUTHORIZATION, "Bearer " + jwt).body(form).retrieve().body(Suggestion.class);
        } catch (RestClientResponseException e) {
            throw map(e);
        } catch (RestClientException e) {
            throw offline(e);
        }
    }

    public void delete(Long id, String jwt) {
        try {
            client.delete().uri("/api/suggestions/{id}", id).header(HttpHeaders.AUTHORIZATION, "Bearer " + jwt).retrieve().toBodilessEntity();
        } catch (RestClientResponseException e) {
            throw map(e);
        } catch (RestClientException e) {
            throw offline(e);
        }
    }
}
```

### Pontos que merecem atenção

1. **Listas tipadas**: `List<CatalogItem>` e `List<Suggestion>` exigem `ParameterizedTypeReference`; se você usar `List.class`, os itens viram `LinkedHashMap` e as templates não conseguem acessar `s.title`.
2. **`Optional.ofNullable(...).orElse(List.of())`**: a API pode responder `200` com corpo vazio; o frontend trata como lista vazia, sem erro.
3. **Filtros**: `UriComponentsBuilder.queryParamIfPresent` só inclui o parâmetro quando há valor. `q` ainda é filtrado por `!q.isBlank()`, evitando `?q=`. `.build().encode()` codifica acentos e espaços.
4. **Path variable**: `.uri("/api/suggestions/{id}", id)` deixa o `RestClient` tratar o ID como segmento de URL, sem concatenar string.
5. **Corpo de login**: usamos `Map.of("username", ..., "password", ...)` porque o Swagger define exatamente essas duas chaves.
6. **Bearer apenas onde a API exige**: `PUT` e `DELETE` recebem `Authorization`. `GET` público e `POST /api/suggestions` não levam token.
7. **`toBodilessEntity()` no DELETE**: a API responde `204`, então não tentamos desserializar um corpo.
8. **Dois `catch`**: `RestClientResponseException` é resposta com status HTTP (4xx/5xx) e vira mensagem amigável; `RestClientException` é falha de conexão e vira `503`.

Se você esquecer de iniciar a API, o primeiro acesso ao mural já cai no `error.html` com status `503` e a mensagem sobre conectar à API.

## Passo 5 — sessão administrativa

O JWT nunca vai para o navegador. Ele fica em um objeto `AdminSession` guardado na sessão HTTP do servidor. O navegador recebe somente o cookie de sessão, com `HttpOnly`.

### `service/AdminSession.java`

```java
package br.com.e24dev.caixaweb.service;

import java.io.Serializable;
import java.time.Instant;

import br.com.e24dev.caixaweb.model.LoginResponse;

public record AdminSession(String username, String token, Instant expiresAt) implements Serializable {
    public static AdminSession from(String username, LoginResponse response) {
        return new AdminSession(username, response.accessToken(), Instant.now().plusSeconds(response.expiresIn()));
    }

    public boolean active() {
        return token != null && !token.isBlank() && Instant.now().isBefore(expiresAt.minusSeconds(5));
    }
}
```

`active()` desconta cinco segundos do `expiresIn` para não usar um token que está prestes a vencer. Como a API não tem refresh, depois que expira é preciso entrar novamente.

### `service/SessionAuthentication.java`

```java
package br.com.e24dev.caixaweb.service;

import jakarta.servlet.http.*;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.context.HttpSessionSecurityContextRepository;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class SessionAuthentication {
    public static final String KEY = "adminSession";

    public void signIn(HttpServletRequest request, AdminSession session) {
        request.changeSessionId();
        HttpSession httpSession = request.getSession();
        httpSession.setAttribute(KEY, session);
        var auth = new UsernamePasswordAuthenticationToken(session.username(), null, List.of(new SimpleGrantedAuthority("ROLE_ADMIN")));
        var context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(auth);
        SecurityContextHolder.setContext(context);
        httpSession.setAttribute(HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY, context);
    }

    public AdminSession get(HttpSession session) {
        Object stored = session.getAttribute(KEY);
        return stored instanceof AdminSession admin && admin.active() ? admin : null;
    }

    public AdminSession get(HttpServletRequest request) {
        HttpSession session = request.getSession(false);
        return session == null ? null : get(session);
    }

    public void signOut(HttpServletRequest request) {
        SecurityContextHolder.clearContext();
        HttpSession session = request.getSession(false);
        if (session != null) session.invalidate();
    }
}
```

- `request.changeSessionId()` troca o ID da sessão no login e evita *session fixation*.
- O `SecurityContext` com `ROLE_ADMIN` é o que faz `authorizeHttpRequests("/admin/**").authenticated()` aceitar a navegação.
- `get(...)` devolve `null` se o token expirou, mesmo que o objeto ainda esteja na sessão.

> No repositório, `SessionAuthentication` não tem a sobrecarga `get(HttpServletRequest)`; o controller usa `auth.get(req.getSession())`. A sobrecarga aqui apenas deixa o controller mais legível. Se quiser reproduzir o repositório exatamente, remova-a e use `auth.get(req.getSession())`.

### `config/AdminSessionFilter.java`

```java
package br.com.e24dev.caixaweb.config;

import br.com.e24dev.caixaweb.service.*;
import jakarta.servlet.*;
import jakarta.servlet.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class AdminSessionFilter extends OncePerRequestFilter {
    private final SessionAuthentication auth;

    public AdminSessionFilter(SessionAuthentication auth) {
        this.auth = auth;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpSession session = request.getSession(false);
        if (session != null && session.getAttribute(SessionAuthentication.KEY) != null && auth.get(session) == null) {
            auth.signOut(request);
            if (request.getRequestURI().startsWith("/admin/")) {
                response.sendRedirect(request.getContextPath() + "/login?expired");
                return;
            }
        }
        chain.doFilter(request, response);
    }
}
```

O filtro detecta sessão expirada e, em rota administrativa, redireciona para `/login?expired`. É isso que faz o alerta amarelo da template `login.html` aparecer.

### `config/SecurityConfig.java` final

Substitua a configuração temporária da Parte 2:

```java
package br.com.e24dev.caixaweb.config;

import org.springframework.context.annotation.*;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {
    @Bean
    SecurityFilterChain webSecurity(HttpSecurity http) throws Exception {
        return http
                .authorizeHttpRequests(auth -> auth.requestMatchers("/admin/**")
                        .authenticated()
                        .anyRequest()
                        .permitAll())
                .exceptionHandling(ex -> ex.authenticationEntryPoint((request, response, authException) -> response.sendRedirect(request.getContextPath() + "/login")))
                .formLogin(AbstractHttpConfigurer::disable)
                .httpBasic(AbstractHttpConfigurer::disable)
                .logout(AbstractHttpConfigurer::disable)
                .build();
    }
}
```

- `/admin/**` exige autenticação; todo o resto é público.
- O login é feito pelo **nosso** `POST /login` contra a API, não pelo formulário padrão do Spring Security — por isso `formLogin` está desabilitado.
- `logout` também está desabilitado porque temos `POST /logout` no controller.
- CSRF permanece ativo; as templates já usam `th:action`.

## Passo 6 — controller final

Substitua o controller mock da Parte 2 por este:

```java
package br.com.e24dev.caixaweb.controller;

import br.com.e24dev.caixaweb.client.*;
import br.com.e24dev.caixaweb.model.*;
import br.com.e24dev.caixaweb.service.*;
import jakarta.servlet.http.*;
import jakarta.validation.Valid;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Controller
public class WebController {
    private final CaixaApiClient api;
    private final SessionAuthentication auth;

    public WebController(CaixaApiClient api, SessionAuthentication auth) {
        this.api = api;
        this.auth = auth;
    }

    @ModelAttribute("isAdmin")
    boolean isAdmin(HttpServletRequest req) {
        HttpSession s = req.getSession(false);
        return s != null && auth.get(s) != null;
    }

    @ModelAttribute("activeAdmin")
    String activeAdmin(HttpServletRequest req) {
        HttpSession s = req.getSession(false);
        AdminSession a = s == null ? null : auth.get(s);
        return a == null ? "" : a.username();
    }

    private void catalogs(Model m) {
        m.addAttribute("courses", api.courses());
        m.addAttribute("categories", api.categories());
    }

    @GetMapping({"/", "/suggestions"})
    String home(@RequestParam(required = false) Long courseId, @RequestParam(required = false) Long categoryId,
                @RequestParam(required = false) String q, Model m) {
        catalogs(m);
        m.addAttribute("suggestions", api.suggestions(courseId, categoryId, q));
        m.addAttribute("courseId", courseId);
        m.addAttribute("categoryId", categoryId);
        m.addAttribute("q", q);
        return "home";
    }

    @GetMapping("/suggestions/new")
    String newSuggestion(Model m) {
        catalogs(m);
        m.addAttribute("form", new SuggestionForm());
        return "new";
    }

    @PostMapping("/suggestions")
    String create(@Valid @ModelAttribute("form") SuggestionForm form, BindingResult result, Model m, RedirectAttributes flash) {
        if (result.hasErrors()) {
            catalogs(m);
            return "new";
        }
        try {
            api.create(form);
            flash.addFlashAttribute("success", "Sua sugestão foi publicada!");
            return "redirect:/";
        } catch (ApiException e) {
            catalogs(m);
            m.addAttribute("error", e.getMessage());
            return "new";
        }
    }

    @GetMapping("/suggestions/{id}")
    String details(@PathVariable Long id, Model m) {
        m.addAttribute("suggestion", api.suggestion(id));
        return "details";
    }

    @GetMapping("/login")
    String login(Model m) {
        if (!m.containsAttribute("loginForm")) m.addAttribute("loginForm", new LoginForm());
        return "login";
    }

    @PostMapping("/login")
    String authenticate(@Valid @ModelAttribute LoginForm loginForm, BindingResult result, HttpServletRequest request, Model m) {
        if (result.hasErrors()) return "login";
        try {
            LoginResponse token = api.login(loginForm.getUsername(), loginForm.getPassword());
            if (token == null || !"Bearer".equalsIgnoreCase(token.tokenType()) || token.accessToken() == null || token.accessToken().isBlank() || token.expiresIn() <= 0) {
                m.addAttribute("error", "Resposta de login inválida.");
                return "login";
            }
            auth.signIn(request, AdminSession.from(loginForm.getUsername(), token));
            return "redirect:/admin/suggestions";
        } catch (ApiException e) {
            m.addAttribute("error", e.status() == 401 ? "Usuário ou senha inválidos." : e.getMessage());
            return "login";
        }
    }

    @PostMapping("/logout")
    String logout(HttpServletRequest request, RedirectAttributes flash) {
        auth.signOut(request);
        flash.addFlashAttribute("success", "Você saiu da área administrativa.");
        return "redirect:/";
    }

    @GetMapping("/admin/suggestions")
    String dashboard(Model m) {
        m.addAttribute("suggestions", api.suggestions(null, null, null));
        return "admin/dashboard";
    }

    @GetMapping("/admin/suggestions/{id}/edit")
    String edit(@PathVariable Long id, Model m) {
        m.addAttribute("suggestionId", id);
        m.addAttribute("form", SuggestionForm.from(api.suggestion(id)));
        catalogs(m);
        return "admin/edit";
    }

    @PostMapping("/admin/suggestions/{id}/edit")
    String update(@PathVariable Long id, @Valid @ModelAttribute("form") SuggestionForm form,
                  BindingResult result, Model m, HttpServletRequest req, RedirectAttributes flash) {
        if (result.hasErrors()) {
            catalogs(m);
            m.addAttribute("suggestionId", id);
            return "admin/edit";
        }
        try {
            api.update(id, form, jwt(req));
            flash.addFlashAttribute("success", "Sugestão atualizada com sucesso.");
            return "redirect:/admin/suggestions";
        } catch (ApiException e) {
            if (e.status() == 401) {
                auth.signOut(req);
                return "redirect:/login?expired";
            }
            catalogs(m);
            m.addAttribute("suggestionId", id);
            m.addAttribute("error", e.getMessage());
            return "admin/edit";
        }
    }

    @PostMapping("/admin/suggestions/{id}/delete")
    String delete(@PathVariable Long id, HttpServletRequest req, RedirectAttributes flash) {
        try {
            api.delete(id, jwt(req));
            flash.addFlashAttribute("success", "Sugestão excluída.");
            return "redirect:/admin/suggestions";
        } catch (ApiException e) {
            if (e.status() == 401) {
                auth.signOut(req);
                return "redirect:/login?expired";
            }
            flash.addFlashAttribute("error", e.getMessage());
            return "redirect:/admin/suggestions";
        }
    }

    private String jwt(HttpServletRequest req) {
        AdminSession s = auth.get(req.getSession());
        if (s == null) throw new ApiException(401, "Sessão expirada.");
        return s.token();
    }

    @ExceptionHandler(ApiException.class)
    String handle(ApiException e, Model m) {
        m.addAttribute("status", e.status());
        m.addAttribute("message", e.getMessage());
        return "error";
    }
}
```

### Leitura guidada

#### Rotas web x rotas da API

| Rota web | Template | Chamada à API |
|---|---|---|
| `GET /`, `GET /suggestions` | `home.html` | `GET /api/courses`, `/api/categories`, `/api/suggestions?...` |
| `GET /suggestions/new` | `new.html` | catálogos |
| `POST /suggestions` | `new.html` ou redirect | `POST /api/suggestions` |
| `GET /suggestions/{id}` | `details.html` | `GET /api/suggestions/{id}` |
| `GET /login` | `login.html` | — |
| `POST /login` | `login.html` ou redirect | `POST /api/auth/login` |
| `POST /logout` | redirect | — (só encerra a sessão local) |
| `GET /admin/suggestions` | `admin/dashboard.html` | `GET /api/suggestions` |
| `GET /admin/suggestions/{id}/edit` | `admin/edit.html` | `GET /api/suggestions/{id}` + catálogos |
| `POST /admin/suggestions/{id}/edit` | `admin/edit.html` ou redirect | `PUT /api/suggestions/{id}` com Bearer |
| `POST /admin/suggestions/{id}/delete` | redirect | `DELETE /api/suggestions/{id}` com Bearer |

#### `@ModelAttribute` em toda requisição

`isAdmin` e `activeAdmin` são executados antes de qualquer handler. É por isso que `layout.html` pode usar `${isAdmin}` em todas as páginas sem cada método precisar adicionar esse atributo. `activeAdmin` fica disponível no `Model`, mas as templates atuais não o exibem.

#### `catalogs(Model)` é chamado em todos os formulários

Tanto `new.html` quanto `admin/edit.html` usam `fragments/form :: fields`, que precisa de `${courses}` e `${categories}`. Quando a validação falha, o controller chama `catalogs(m)` novamente antes de retornar a template — sem isso, os `<select>` apareceriam vazios.

#### Post/Redirect/Get

Depois de criar, editar, excluir ou entrar, o controller retorna `redirect:/...`. O `RedirectAttributes` guarda mensagens `success`/`error` que sobrevivem ao redirect e são lidas por `fragments/layout :: alerts`. Isso evita o reenvio do formulário quando o usuário recarrega a página.

#### Validação da resposta de login

O controller confere `tokenType` igual a `Bearer`, `accessToken` não vazio e `expiresIn > 0`. Se a API responder algo fora do contrato, a tela mostra `Resposta de login inválida.` em vez de quebrar.

#### Tratamento de expiração

Em `PUT` e `DELETE`, um `401` da API chama `auth.signOut(req)` e redireciona para `/login?expired`. Um `404` ou `400` cai no `@ExceptionHandler` e abre `error.html` com o status e a mensagem.

## Passo 7 — checkpoint final

1. Com a API rodando em `http://localhost:8000`, execute `./gradlew bootRun`.
2. Acesse `http://localhost:8081/`. Confira que os cursos/categorias e sugestões vêm da API — crie uma sugestão no Swagger e recarregue o mural para vê-la.
3. Use os filtros: `q=tomadas`, depois `courseId=1`, depois os três combinados. Confira que os campos continuam preenchidos e que a lista obedece aos filtros.
4. Abra um cartão e confira o autor `Anônimo` quando o cadastro foi sem nome e a data formatada.
5. Envie uma sugestão nova pelo formulário; veja o alerta verde e o cartão no topo do mural.
6. Envie o formulário com campos obrigatórios vazios; confira as mensagens de `th:errors` e os `<select>` ainda populados.
7. Abra `/admin/suggestions` sem sessão: você deve ser redirecionado para `/login`.
8. Entre com as credenciais fornecidas pelo professor. O navbar troca para **Painel admin** e **Sair**.
9. Edite uma sugestão: limpe o autor, salve e confirme que a API devolve `Anônimo`.
10. Exclua uma sugestão, cancele o `confirm()` e depois confirme.
11. Clique em **Sair**: alerta verde no mural e `/admin/suggestions` volta a redirecionar para `/login`.
12. Pare a API e recarregue `/`: a template `error.html` deve mostrar `503` e a mensagem sobre conectar à API.
13. Volte a iniciar a API antes de entregar.

## Solução de problemas

| Sintoma | Causa provável | Correção |
|---|---|---|
| Tela `503` já no mural | API fora ou URL errada | conferir `http://localhost:8000/swagger-ui/index.html`; ajustar `API_BASE_URL` |
| `404` em todas as chamadas | `API_BASE_URL` terminou com `/api` | usar só `http://localhost:8000` |
| Porta `8081` já em uso | API e web na mesma porta | `WEB_PORT=8082 ./gradlew bootRun` |
| Lista vazia mesmo com dados no Swagger | filtro com ID inexistente ou `q` com espaços | conferir parâmetros na URL; IDs devem vir dos catálogos |
| Login dá `Usuário ou senha inválidos.` | credenciais incorretas ou API 401 | usar as credenciais do professor; conferir logs da API |
| Após login, `/admin/**` ainda redireciona | `SecurityConfig` temporária da Parte 2 | substituir pela configuração final |
| 403 ao editar/excluir | CSRF ausente ou sessão perdida | usar `th:action`; entrar novamente |
| 401 ao editar/excluir | JWT expirado | entrar novamente; a API não tem refresh |
| Erro `Property or field 'name' cannot be found on null` | `course` ou `category` veio `null` | conferir se os IDs enviados existem nos catálogos |
| Sugestão criada com `Anônimo` apesar de ter digitado nome | campo `author` em branco ou omitido | comportamento documentado pela API |
| `createdAt` aparece cru | template fora da Parte 2 | usar `#temporals.format(...,'dd/MM/yyyy')` |
| HTML mostra `th:` | arquivo aberto como estático | apagar `static/prototipo/` e acessar pela rota do controller |

## Entrega

Antes de entregar, confira:

- [ ] API em execução e Swagger acessível.
- [ ] `./gradlew clean build` passa.
- [ ] Nenhuma credencial ou token versionado.
- [ ] `API_BASE_URL` sem `/api`.
- [ ] JWT apenas em sessão server-side; nada em `localStorage` ou campo oculto.
- [ ] Criar, filtrar, detalhar, login, editar, excluir e sair funcionando.
- [ ] Erro `503` aparece quando a API para.
- [ ] README com as instruções de execução e os links das três partes.

Você chegou ao final do tutorial. O código completo deste repositório é a referência para comparar qualquer dúvida de sintaxe ou de layout.
