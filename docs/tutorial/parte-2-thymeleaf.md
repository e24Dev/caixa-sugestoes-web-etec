# Parte 2 — Adaptando o HTML para templates Thymeleaf

[Voltar ao README](../../README.md) · [Anterior: Parte 1](parte-1-interface-statica.md) · [Próxima: Parte 3](parte-3-integracao-api.md)

## Objetivo

Transformar os seis protótipos estáticos da Parte 1 em templates processados no servidor. Nesta etapa:

- o navegador continua recebendo HTML pronto;
- os dados ainda são **mock** (criados em Java, sem API);
- o layout é dividido em fragments reutilizáveis;
- formulários passam a ser vinculados a objetos Java;
- filtros, listas, links e mensagens passam a ser gerados pelo `Model`.

Ao final, você terá as nove templates que existem em `src/main/resources/templates/`. A Parte 3 apenas troca o controller mock pelo controller que conversa com a API — **as templates não mudam**.

> Este é o documento mais longo do tutorial. Isso é intencional: a maior parte dos erros em MVC com Thymeleaf acontece aqui, antes de existir API envolvida.

## O que muda em relação à Parte 1

| Parte 1 (estático) | Parte 2 (Thymeleaf) |
|---|---|
| `static/prototipo/index.html` | `templates/home.html` |
| Links relativos `new.html`, `index.html` | `th:href="@{/suggestions/new}"`, `th:href="@{/}"` |
| Texto fixo no HTML | `th:text="${...}"` |
| Três cartões escritos à mão | um cartão com `th:each="s : ${suggestions}"` |
| Cabeçalho/rodapé repetidos em cada arquivo | `th:replace="~{fragments/layout :: head('...')}"` etc. |
| Campos com `name="title"` | `th:field="*{title}"` dentro de `th:object="${form}"` |
| Filtro feito por `ui.js` | formulário `GET /` com `th:value` e `th:selected` |
| `data-confirm` no formulário de exclusão | `onsubmit="return confirm(...)"` no template final |
| Nenhum controller | `@Controller` temporário com dados mock |

## Estrutura de pastas ao final desta etapa

```text
src/main/java/br/com/e24dev/caixaweb/
├── CaixaWebApplication.java
├── config/
│   └── SecurityConfig.java        <- versão temporária, ver abaixo
├── controller/
│   └── WebController.java         <- versão mock, substituída na Parte 3
└── model/
    ├── CatalogItem.java
    ├── LoginForm.java
    ├── Suggestion.java
    └── SuggestionForm.java

src/main/resources/
├── application.yaml
├── static/
│   ├── css/
│   │   └── site.css
│   └── prototipo/                 <- apague ao final da Parte 2
└── templates/
    ├── fragments/
    │   ├── form.html
    │   └── layout.html
    ├── admin/
    │   ├── dashboard.html
    │   └── edit.html
    ├── details.html
    ├── error.html
    ├── home.html
    ├── login.html
    └── new.html
```

Confirme em `application.yaml`:

```yaml
spring:
  thymeleaf:
    cache: false
```

Com `cache: false`, alterar um template e recarregar a página funciona sem reiniciar a aplicação. Em produção, o valor costuma ser `true`.

## Passo 1 — classes Java de apoio (somente dados)

Crie estas quatro classes. Elas são pequenas e representam exatamente o formato que a API devolverá na Parte 3.

### `model/CatalogItem.java`

```java
package br.com.e24dev.caixaweb.model;

public record CatalogItem(Long id, String name) {
}
```

### `model/Suggestion.java`

```java
package br.com.e24dev.caixaweb.model;

import java.time.Instant;

public record Suggestion(Long id, String title, String content, String author, Instant createdAt,
                         CatalogItem course, CatalogItem category) {
}
```

Repare em `course` e `category`: não são IDs soltos. São objetos `CatalogItem` completos, porque a API devolve `course: { id, name }` e `category: { id, name }`. É por isso que no template escrevemos `s.course.name`.

### `model/SuggestionForm.java`

```java
package br.com.e24dev.caixaweb.model;

import jakarta.validation.constraints.*;

public class SuggestionForm {
    @NotBlank(message = "Informe o título")
    private String title;
    @NotBlank(message = "Descreva sua sugestão")
    private String content;
    private String author;
    @NotNull(message = "Selecione um curso")
    @Positive(message = "Selecione um curso")
    private Long courseId;
    @NotNull(message = "Selecione uma categoria")
    @Positive(message = "Selecione uma categoria")
    private Long categoryId;

    public String getTitle() { return title; }
    public void setTitle(String v) { title = v; }
    public String getContent() { return content; }
    public void setContent(String v) { content = v; }
    public String getAuthor() { return author; }
    public void setAuthor(String v) { author = v; }
    public Long getCourseId() { return courseId; }
    public void setCourseId(Long v) { courseId = v; }
    public Long getCategoryId() { return categoryId; }
    public void setCategoryId(Long v) { categoryId = v; }
}
```

Este é um **form bean**: precisa de getters e setters porque `th:field` lê e escreve propriedades por nome. As anotações de validação fazem `th:errors` mostrar mensagens quando o usuário deixa campo obrigatório em branco.

### `model/LoginForm.java`

```java
package br.com.e24dev.caixaweb.model;

import jakarta.validation.constraints.NotBlank;

public class LoginForm {
    @NotBlank
    private String username;
    @NotBlank
    private String password;

    public String getUsername() { return username; }
    public void setUsername(String v) { username = v; }
    public String getPassword() { return password; }
    public void setPassword(String v) { password = v; }
}
```

## Passo 2 — controller temporário com dados mock

Este controller existe **apenas para você enxergar as templates funcionando**. Ele será apagado na Parte 3.

### `controller/WebController.java`

Substitua o conteúdo atual por esta versão:

```java
package br.com.e24dev.caixaweb.controller;

import br.com.e24dev.caixaweb.model.*;
import jakarta.validation.Valid;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

import java.time.Instant;
import java.util.List;

@Controller
public class WebController {
    private final List<CatalogItem> courses = List.of(
            new CatalogItem(1L, "Desenvolvimento de Sistemas"),
            new CatalogItem(2L, "Eletromecânica"));

    private final List<CatalogItem> categories = List.of(
            new CatalogItem(1L, "Infraestrutura"),
            new CatalogItem(2L, "Ensino"));

    private final List<Suggestion> suggestions = List.of(
            new Suggestion(1L, "Mais tomadas na biblioteca",
                    "Instalar tomadas nas mesas de estudo.", "Anônimo",
                    Instant.parse("2026-10-08T12:00:00Z"), courses.get(0), categories.get(0)),
            new Suggestion(2L, "Clube de estudos semanal",
                    "Um encontro semanal para revisar conteúdos.", "Maria Silva",
                    Instant.parse("2026-10-10T12:00:00Z"), courses.get(0), categories.get(1)),
            new Suggestion(3L, "Iluminação dos corredores",
                    "Trocar as lâmpadas antigas por LED.", "Anônimo",
                    Instant.parse("2026-10-12T12:00:00Z"), courses.get(1), categories.get(0)));

    @ModelAttribute("isAdmin")
    boolean isAdmin() {
        return true; // mude para false para ver o botão "Área administrativa"
    }

    @ModelAttribute("activeAdmin")
    String activeAdmin() {
        return "administrador";
    }

    private void catalogs(Model m) {
        m.addAttribute("courses", courses);
        m.addAttribute("categories", categories);
    }

    @GetMapping({"/", "/suggestions"})
    String home(@RequestParam(required = false) Long courseId,
                @RequestParam(required = false) Long categoryId,
                @RequestParam(required = false) String q, Model m) {
        catalogs(m);
        m.addAttribute("suggestions", suggestions);
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
    String create(@Valid @ModelAttribute("form") SuggestionForm form,
                  BindingResult result, Model m, RedirectAttributes flash) {
        if (result.hasErrors()) {
            catalogs(m);
            return "new";
        }
        flash.addFlashAttribute("success", "Sugestão de exemplo publicada!");
        return "redirect:/";
    }

    @GetMapping("/suggestions/{id}")
    String details(@PathVariable Long id, Model m) {
        m.addAttribute("suggestion", suggestions.get(0));
        return "details";
    }

    @GetMapping("/login")
    String login(Model m) {
        if (!m.containsAttribute("loginForm")) m.addAttribute("loginForm", new LoginForm());
        return "login";
    }

    @PostMapping("/login")
    String authenticate(@Valid @ModelAttribute LoginForm loginForm,
                        BindingResult result, Model m) {
        if (result.hasErrors()) return "login";
        m.addAttribute("error", "Login simulado: na Parte 3 este formulário chamará a API.");
        return "login";
    }

    @PostMapping("/logout")
    String logout(RedirectAttributes flash) {
        flash.addFlashAttribute("success", "Logout simulado.");
        return "redirect:/";
    }

    @GetMapping("/admin/suggestions")
    String dashboard(Model m) {
        m.addAttribute("suggestions", suggestions);
        return "admin/dashboard";
    }

    @GetMapping("/admin/suggestions/{id}/edit")
    String edit(@PathVariable Long id, Model m) {
        m.addAttribute("suggestionId", id);
        Suggestion s = suggestions.get(0);
        SuggestionForm form = new SuggestionForm();
        form.setTitle(s.title());
        form.setContent(s.content());
        form.setAuthor(s.author());
        form.setCourseId(s.course().id());
        form.setCategoryId(s.category().id());
        m.addAttribute("form", form);
        catalogs(m);
        return "admin/edit";
    }

    @PostMapping("/admin/suggestions/{id}/edit")
    String update(@PathVariable Long id, @Valid @ModelAttribute("form") SuggestionForm form,
                  BindingResult result, Model m, RedirectAttributes flash) {
        if (result.hasErrors()) {
            catalogs(m);
            m.addAttribute("suggestionId", id);
            return "admin/edit";
        }
        flash.addFlashAttribute("success", "Sugestão de exemplo atualizada.");
        return "redirect:/admin/suggestions";
    }

    @PostMapping("/admin/suggestions/{id}/delete")
    String delete(@PathVariable Long id, RedirectAttributes flash) {
        flash.addFlashAttribute("success", "Sugestão de exemplo excluída.");
        return "redirect:/admin/suggestions";
    }
}
```

### `config/SecurityConfig.java` temporário

O repositório final protege `/admin/**`. Durante a Parte 2 ainda não existe sessão administrativa, então use esta configuração provisória para conseguir abrir o painel:

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
                .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
                .formLogin(AbstractHttpConfigurer::disable)
                .httpBasic(AbstractHttpConfigurer::disable)
                .logout(AbstractHttpConfigurer::disable)
                .build();
    }
}
```

Não desabilitamos CSRF: as templates usarão `th:action` nos formulários POST, e o dialeto do Spring Security para Thymeleaf injeta o campo `_csrf` automaticamente. Na Parte 3, esta configuração é substituída pela versão que exige autenticação em `/admin/**`.

Rode `./gradlew bootRun` e acesse `http://localhost:8081/`. Por enquanto você verá erro de template, pois ainda não criamos os arquivos em `templates/`. Isso é esperado.

## Passo 3 — a linguagem das expressões Thymeleaf

Antes de colar as templates, aprenda os quatro prefixos que aparecem o tempo todo:

| Prefixo | Significado | Exemplo |
|---|---|---|
| `${...}` | acessa atributo do `Model` ou objeto utilitário | `${suggestions}`, `${isAdmin}`, `${#lists.size(suggestions)}` |
| `*{...}` | acessa propriedade do objeto definido por `th:object` | `*{title}`, `*{courseId}` |
| `@{...}` | constrói URL respeitando o context path | `@{/css/site.css}`, `@{/suggestions/{id}(id=${s.id})}` |
| `~{...}` | referencia um fragmento | `~{fragments/layout :: nav}` |

Diferenças que confundem iniciantes:

- `${title}` procura um atributo chamado `title` no `Model`;
- `*{title}` procura o getter `getTitle()` do objeto atual de `th:object`;
- `@{/}` gera `/` quando a aplicação está na raiz e `/meu-app/` se houver context path — por isso nunca use `href="/..."` fixo;
- `~{...}` não gera texto: ele aponta para um pedaço de outro template.

Atributos mais usados neste projeto:

| Atributo | Função | Exemplo no projeto |
|---|---|---|
| `th:text` | escreve texto escapado | `th:text="${s.title}"` |
| `th:href` | gera `href` | `th:href="@{/suggestions/new}"` |
| `th:src` | gera `src` | `th:src="@{/webjars/bootstrap/5.3.3/js/bootstrap.bundle.min.js}"` |
| `th:action` | gera `action` e habilita CSRF em POST | `th:action="@{/login}" method="post"` |
| `th:each` | repete um elemento | `th:each="s,st:${suggestions}"` |
| `th:if` / `th:unless` | inclusão condicional | `th:if="${isAdmin}"`, `th:unless="${isAdmin}"` |
| `th:object` | define o objeto de referência para `*` | `th:object="${form}"` |
| `th:field` | gera `id`, `name` e `value` de um campo | `th:field="*{title}"` |
| `th:errors` | mostra mensagens de validação | `th:errors="*{content}"` |
| `th:value` | gera `value` | `th:value="${q}"` |
| `th:selected` | marca `<option>` em selects de filtro | `th:selected="${courseId == c.id}"` |
| `th:replace` | substitui o elemento por um fragmento | `th:replace="~{fragments/layout :: nav}"` |
| `th:fragment` | declara um fragmento reutilizável | `th:fragment="head(title)"` |

Concatenação de texto usa `+`, como em `th:text="'IDEIA ' + ${s.id}"`. Também existe a forma com barras verticais, `th:text="|IDEIA ${s.id}|"`; o repositório usa a primeira.

`th:text` escapa HTML. Nunca troque por `th:utext` para título, conteúdo ou autor: são dados escritos por usuários.

## Passo 4 — `templates/fragments/layout.html`

Crie a pasta `fragments` e este arquivo. Ele concentra cabeçalho, navegação, alertas, rodapé e scripts.

```html
<!DOCTYPE html>
<html lang="pt-BR" xmlns:th="http://www.thymeleaf.org">
<head th:fragment="head(title)">
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <meta name="description" content="Caixa de Sugestões — um espaço para transformar ideias em melhorias.">
    <title th:text="${title} + ' · Caixa de Sugestões'">Caixa de Sugestões</title>
    <link th:href="@{/webjars/bootstrap/5.3.3/css/bootstrap.min.css}" rel="stylesheet">
    <link th:href="@{/webjars/bootstrap-icons/1.11.3/font/bootstrap-icons.css}" rel="stylesheet">
    <link th:href="@{/css/site.css}" rel="stylesheet">
</head>
<body>
<nav th:fragment="nav" class="navbar navbar-expand-lg navbar-light sticky-top site-nav">
    <div class="container py-2">
        <a class="navbar-brand d-flex align-items-center gap-2" th:href="@{/}"><span class="brand-mark"><i
                class="bi bi-chat-square-heart-fill"></i></span><span>caixa<span class="text-brand">ideias</span><small
                class="brand-subtitle d-block">VOZES QUE TRANSFORMAM</small></span></a>
        <button class="navbar-toggler border-0" type="button" data-bs-toggle="collapse" data-bs-target="#navigation"
                aria-controls="navigation" aria-expanded="false" aria-label="Abrir navegação"><span
                class="navbar-toggler-icon"></span></button>
        <div class="collapse navbar-collapse" id="navigation">
            <div class="navbar-nav ms-auto align-items-lg-center gap-lg-3">
                <a class="nav-link" th:href="@{/}">Explorar ideias</a>
                <a class="nav-link" th:href="@{/suggestions/new}">Enviar sugestão</a>
                <a th:if="${isAdmin}" class="nav-link" th:href="@{/admin/suggestions}">Painel admin</a>
                <a th:unless="${isAdmin}" class="btn btn-sm btn-outline-dark rounded-pill px-4 ms-lg-2"
                   th:href="@{/login}"><i class="bi bi-shield-lock me-1"></i> Área administrativa</a>
                <form th:if="${isAdmin}" th:action="@{/logout}" method="post" class="ms-lg-2">
                    <button class="btn btn-sm btn-outline-dark rounded-pill px-4">Sair <i
                            class="bi bi-box-arrow-right"></i></button>
                </form>
            </div>
        </div>
    </div>
</nav>
<div th:fragment="alerts" class="container mt-3">
    <div th:if="${success}" class="alert alert-success border-0 rounded-4" role="status"><i
            class="bi bi-check-circle-fill me-2"></i><span th:text="${success}"></span></div>
    <div th:if="${error}" class="alert alert-danger border-0 rounded-4" role="alert"><i
            class="bi bi-exclamation-circle-fill me-2"></i><span th:text="${error}"></span></div>
</div>
<footer th:fragment="footer" class="py-5 mt-5 footer">
    <div class="container d-flex flex-column flex-md-row align-items-start justify-content-between gap-3">
        <div><strong><i class="bi bi-chat-square-heart-fill text-brand me-2"></i>caixaideias</strong>
            <p class="text-muted small mt-2 mb-0">Pequenas ideias. Grandes mudanças.</p></div>
        <div class="small text-muted">Projeto educacional · Spring Boot · Thymeleaf</div>
    </div>
</footer>
<div th:fragment="scripts">
    <script th:src="@{/webjars/bootstrap/5.3.3/js/bootstrap.bundle.min.js}"></script>
</div>
</body>
</html>
```

### Leitura guidada

1. `xmlns:th="http://www.thymeleaf.org"` no `<html>` declara o namespace. Sem ele, o IDE pode reclamar e alguns processadores não reconhecem os atributos.
2. `th:fragment="head(title)"` cria um fragmento com **parâmetro**. Quem chama escreve `~{fragments/layout :: head('Ideias que transformam')}`; dentro do fragmento, esse valor chega como `title`.
3. `<title th:text="${title} + ' · Caixa de Sugestões'">Caixa de Sugestões</title>` usa o parâmetro. O texto entre as tags é o *placeholder* exibido se você abrir o arquivo diretamente no editor — ele é substituído no processamento.
4. Os `<link th:href="@{...}">` apontam para WebJars e para `static/css/site.css`. Continuam sendo CSS; só a URL é gerada pelo Thymeleaf.
5. `th:fragment="nav"` cria a barra de navegação. As versões públicas e administrativas são decididas por `th:if="${isAdmin}"` e `th:unless="${isAdmin}"`. São complements exatos: `unless` é o inverso de `if`.
6. O formulário de logout é `POST` com `th:action`; o token CSRF aparece sozinho no HTML final.
7. `th:fragment="alerts"` centraliza `success` e `error`. Um `flash.addFlashAttribute("success", ...)` no controller faz o primeiro bloco aparecer após um redirect.
8. `th:fragment="scripts"` deixa o bundle do Bootstrap em um único lugar. O `ui.js` da Parte 1 não volta: o filtro agora é servidor.
9. `th:replace` (usado pelas outras templates) **substitui** a tag chamadora pelo conteúdo do fragmento. Se fosse `th:insert`, a tag continuaria e haveria um wrapper extra.

Experimente: mude `isAdmin()` no controller entre `true` e `false`, recarregue `/` e veja o navbar trocar entre **Painel admin + Sair** e **Área administrativa**.

## Passo 5 — `templates/fragments/form.html`

Os campos de cadastro e edição são iguais. Em vez de duplicar, criamos um fragmento de formulário.

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<body>
<div th:fragment="fields">
    <div class="row g-4">
        <div class="col-12"><label for="title" class="form-label fw-semibold">Título da sugestão <span
                class="text-danger">*</span></label><input id="title" th:field="*{title}"
                                                           class="form-control form-control-lg"
                                                           placeholder="Ex.: Mais espaços de estudo" maxlength="255"
                                                           required>
            <div class="text-danger small mt-1" th:errors="*{title}"></div>
        </div>
        <div class="col-12"><label for="content" class="form-label fw-semibold">Conte sua ideia <span
                class="text-danger">*</span></label><textarea id="content" th:field="*{content}" class="form-control"
                                                               rows="5"
                                                               placeholder="Descreva o que pode melhorar e por quê..."
                                                               required></textarea>
            <div class="text-danger small mt-1" th:errors="*{content}"></div>
        </div>
        <div class="col-md-6"><label for="courseId" class="form-label fw-semibold">Curso <span
                class="text-danger">*</span></label><select id="courseId" th:field="*{courseId}"
                                                            class="form-select form-select-lg" required>
            <option value="">Selecione seu curso</option>
            <option th:each="c : ${courses}" th:value="${c.id}" th:text="${c.name}"></option>
        </select>
            <div class="text-danger small mt-1" th:errors="*{courseId}"></div>
        </div>
        <div class="col-md-6"><label for="categoryId" class="form-label fw-semibold">Categoria <span
                class="text-danger">*</span></label><select id="categoryId" th:field="*{categoryId}"
                                                            class="form-select form-select-lg" required>
            <option value="">Selecione a categoria</option>
            <option th:each="c : ${categories}" th:value="${c.id}" th:text="${c.name}"></option>
        </select>
            <div class="text-danger small mt-1" th:errors="*{categoryId}"></div>
        </div>
        <div class="col-12"><label for="author" class="form-label fw-semibold">Seu nome <span
                class="text-muted fw-normal">(opcional)</span></label><input id="author" th:field="*{author}"
                                                                              class="form-control form-control-lg"
                                                                              placeholder="Você pode participar anonimamente">
            <p class="form-text">Deixe em branco para publicar como Anônimo.</p></div>
    </div>
</div>
</body>
</html>
```

### Leitura guidada

1. `th:fragment="fields"` não recebe parâmetros. Quem usa precisa garantir que o `Model` tenha `courses`, `categories` e um objeto de formulário.
2. `th:field="*{title}"` faz três coisas: gera `name="title"`, `id="title"` e `value` com o valor atual de `getTitle()`. Por isso não escrevemos `name=` à mão.
3. `th:errors="*{title}"` preenche o `<div>` quando `@Valid` encontrou erro em `title`. Se não houver erro, a `<div>` fica vazia.
4. Nos `<select>`, `th:field="*{courseId}"` vincula a seleção inteira. As opções vêm de `${courses}` com `th:value="${c.id}"` e `th:text="${c.name}"`. **Não use `th:selected` aqui**: o Thymeleaf já marca a opção cujo valor iguala `getCourseId()`.
5. O `required` do HTML e as anotações `@NotBlank`/`@NotNull` do Java são complementares: o primeiro impede envio no navegador; o segundo protege o servidor e alimenta `th:errors`.
6. `author` não tem `th:errors` porque é opcional.

Para testar: envie o formulário em branco com o DevTools do navegador desativando validação HTML, ou simplesmente limpe o título e habilite `required` novamente. As mensagens "Informe o título" e "Descreva sua sugestão" devem aparecer.

## Passo 6 — `templates/home.html` (mural)

```html
<!DOCTYPE html>
<html lang="pt-BR" xmlns:th="http://www.thymeleaf.org">
<head th:replace="~{fragments/layout :: head('Ideias que transformam')}"></head>
<body>
<nav th:replace="~{fragments/layout :: nav}"></nav>
<div th:replace="~{fragments/layout :: alerts}"></div>
<section class="hero">
    <div class="container">
        <div class="row align-items-center gy-5">
            <div class="col-lg-7"><span class="eyebrow"><i
                    class="bi bi-stars me-1"></i> UM ESPAÇO PARA SER OUVIDO</span>
                <h1 class="display-3 fw-bold lh-sm mt-3">Toda grande mudança<br>começa com <span class="serif-accent">uma ideia.</span>
                </h1>
                <p class="lead text-secondary mt-4 mb-4 hero-copy">Sua voz faz a diferença. Compartilhe sugestões para
                    tornar nossa comunidade cada vez melhor.</p>
                <div class="d-flex gap-3 flex-wrap"><a class="btn btn-brand btn-lg rounded-pill px-4"
                                                       th:href="@{/suggestions/new}">Compartilhar uma ideia <i
                        class="bi bi-arrow-up-right ms-2"></i></a><a
                        class="btn btn-outline-dark btn-lg rounded-pill px-4" href="#ideias">Explorar sugestões</a>
                </div>
                <div class="d-flex gap-3 mt-5 align-items-center">
                    <div class="mini-avatars"><span>✦</span><span>✳</span><span>✿</span></div>
                    <div class="small text-muted">Um espaço aberto a todas as vozes</div>
                </div>
            </div>
            <div class="col-lg-5">
                <div class="hero-art">
                    <div class="art-orbit orbit-a"></div>
                    <div class="art-orbit orbit-b"></div>
                    <div class="art-card art-card-top"><i
                            class="bi bi-lightbulb-fill"></i><span>Novas possibilidades</span></div>
                    <div class="art-center"><i class="bi bi-chat-square-heart-fill"></i></div>
                    <div class="art-card art-card-bottom"><span>Ideias viram <strong>impacto</strong></span><i
                            class="bi bi-arrow-up-right"></i></div>
                    <div class="art-spark spark-a">✳</div>
                    <div class="art-spark spark-b">✦</div>
                </div>
            </div>
        </div>
    </div>
</section>
<section class="container pt-5" id="ideias">
    <div class="d-flex justify-content-between align-items-end flex-wrap gap-3 mb-4">
        <div><span class="eyebrow">MURAL DA COMUNIDADE</span>
            <h2 class="section-heading mt-2">Ideias em destaque<span class="text-brand">.</span></h2>
            <p class="text-muted mb-0">Explore, descubra e inspire-se com novas sugestões.</p></div>
        <span class="count-pill" th:text="${#lists.size(suggestions)} + ' sugestões encontradas'">0 sugestões</span>
    </div>
    <form class="filter-panel row g-3 align-items-end" method="get" th:action="@{/}">
        <div class="col-lg-5"><label for="q" class="form-label small fw-semibold">O que você procura?</label>
            <div class="input-group"><span class="input-group-text border-end-0 bg-white"><i
                    class="bi bi-search"></i></span><input id="q" class="form-control border-start-0" name="q"
                                                            th:value="${q}" placeholder="Pesquise por um tema..."></div>
        </div>
        <div class="col-lg-3"><label for="courseId" class="form-label small fw-semibold">Curso</label><select
                id="courseId" class="form-select" name="courseId">
            <option value="">Todos os cursos</option>
            <option th:each="c:${courses}" th:value="${c.id}" th:text="${c.name}"
                    th:selected="${courseId == c.id}"></option>
        </select></div>
        <div class="col-lg-2"><label for="categoryId" class="form-label small fw-semibold">Categoria</label><select
                id="categoryId" class="form-select" name="categoryId">
            <option value="">Todas</option>
            <option th:each="c:${categories}" th:value="${c.id}" th:text="${c.name}"
                    th:selected="${categoryId == c.id}"></option>
        </select></div>
        <div class="col-lg-2">
            <button class="btn btn-brand w-100" type="submit">Filtrar <i class="bi bi-arrow-right ms-2"></i></button>
        </div>
    </form>
    <div class="row g-4 mt-2">
        <div class="col-md-6 col-lg-4" th:each="s,st:${suggestions}">
            <article class="idea-card h-100">
                <div class="d-flex justify-content-between align-items-center mb-4"><span class="category-tag"><i
                        class="bi bi-bookmark me-1"></i><span th:text="${s.category.name}"></span></span><span
                        class="card-seq" th:text="'IDEIA ' + ${s.id}">IDEIA 01</span></div>
                <h3 class="h5 fw-bold" th:text="${s.title}"></h3>
                <p class="idea-excerpt" th:text="${#strings.abbreviate(s.content,140)}"></p>
                <div class="idea-meta"><span><i class="bi bi-mortarboard me-1"></i><span
                        th:text="${s.course.name}"></span></span><span><i class="bi bi-person me-1"></i><span
                        th:text="${s.author}"></span></span></div>
                <a class="stretched-link view-link" th:href="@{/suggestions/{id}(id=${s.id})}">Ler sugestão <i
                        class="bi bi-arrow-up-right"></i></a></article>
        </div>
        <div th:if="${#lists.isEmpty(suggestions)}" class="empty-state text-center"><i
                class="bi bi-chat-left-dots fs-1 text-brand"></i>
            <h3 class="h4 mt-3">Nenhuma sugestão por aqui ainda.</h3>
            <p class="text-muted">Tente outros filtros ou seja a primeira pessoa a compartilhar uma ideia.</p><a
                    th:href="@{/suggestions/new}" class="btn btn-brand rounded-pill">Enviar sugestão</a></div>
    </div>
    <div class="cta-strip mt-5">
        <div class="container d-flex flex-column flex-lg-row align-items-lg-center justify-content-between gap-3">
            <div><span class="eyebrow text-white">SUA VEZ DE PARTICIPAR</span>
                <h2 class="fw-bold mt-2 mb-0">E se a próxima grande ideia for sua?</h2></div>
            <a class="btn btn-light rounded-pill px-4 py-3" th:href="@{/suggestions/new}">Quero participar <i
                    class="bi bi-arrow-up-right ms-2"></i></a></div>
    </div>
    <footer th:replace="~{fragments/layout :: footer}"></footer>
    <div th:replace="~{fragments/layout :: scripts}"></div>
</div>
</body>
</html>
```

### Leitura guidada

1. As quatro primeiras linhas de corpo mostram o padrão de composição: `head` com parâmetro, `nav`, `alerts` e, no fim, `footer` + `scripts`.
2. `#lists.size(suggestions)` conta os itens. O texto `0 sugestões` é apenas placeholder.
3. O formulário de filtro é `GET /` com `th:action="@{/}"`. Cada `<input>`/`<select>` mantém `name=` porque precisa virar query string; apenas o valor é dinâmico (`th:value="${q}"`).
4. `th:selected="${courseId == c.id}"` compara o parâmetro enviado na URL com o ID da opção. Quando `courseId` é `null`, nenhuma opção fica marcada e `Todos os cursos` prevalece. Aqui **usamos** `th:selected` porque este select não está preso a `th:field`.
5. `th:each="s,st:${suggestions}"` repete a `<div class="col...">` para cada sugestão. `st` é a variável de status (índice, contador, first/last); como não é usada, você pode escrever apenas `th:each="s:${suggestions}"`.
6. `s.category.name` e `s.course.name` navegam dentro dos objetos aninhados que vimos em `Suggestion`.
7. `#strings.abbreviate(s.content,140)` corta o texto longo e acrescenta `...`. É o equivalente servidor do que fazíamos com CSS `-webkit-line-clamp`.
8. `@{/suggestions/{id}(id=${s.id})}` gera `/suggestions/1`, `/suggestions/2` etc. A sintaxe entre chaves declara o nome do parâmetro de path; os parênteses passam o valor.
9. `th:if="${#lists.isEmpty(suggestions)}"` cria o estado vazio. Ele não entra em conflito com o loop: se a lista está vazia, o `th:each` não gera nenhum cartão.
10. Abra `http://localhost:8081/?courseId=1&q=biblioteca` e confirme que o campo de busca e o select voltam preenchidos. O filtro em si ainda não resolve nada porque o controller mock devolve a lista inteira — isso muda na Parte 3.

## Passo 7 — `templates/new.html`

```html
<!DOCTYPE html>
<html lang="pt-BR" xmlns:th="http://www.thymeleaf.org">
<head th:replace="~{fragments/layout :: head('Compartilhe uma sugestão')}"></head>
<body>
<nav th:replace="~{fragments/layout :: nav}"></nav>
<div class="container page-narrow py-5"><a class="back-link" th:href="@{/}"><i class="bi bi-arrow-left me-2"></i>Voltar
    ao mural</a>
    <div class="mt-4 mb-4"><span class="eyebrow">SUA VOZ IMPORTA</span>
        <h1 class="page-title mt-2">Compartilhe sua <span class="serif-accent">ideia.</span></h1>
        <p class="text-muted">Conte o que podemos melhorar. Sua sugestão será publicada no mural.</p></div>
    <div th:replace="~{fragments/layout :: alerts}"></div>
    <form class="paper-card p-4 p-md-5" method="post" th:action="@{/suggestions}" th:object="${form}">
        <div th:replace="~{fragments/form :: fields}"></div>
        <div class="d-flex justify-content-between align-items-center gap-3 flex-wrap mt-4"><a
                class="text-muted text-decoration-none" th:href="@{/}">Cancelar</a>
            <button type="submit" class="btn btn-brand btn-lg rounded-pill px-4">Publicar sugestão <i
                    class="bi bi-send ms-2"></i></button>
        </div>
    </form>
</div>
<footer th:replace="~{fragments/layout :: footer}"></footer>
<div th:replace="~{fragments/layout :: scripts}"></div>
</body>
</html>
```

### Leitura guidada

1. `th:object="${form}"` define o objeto que `*{...}` usará. Ele tem que existir no `Model` — o controller mock coloca `new SuggestionForm()` em `GET /suggestions/new`.
2. `th:action="@{/suggestions}"` + `method="post"` gera o `action` e o token CSRF.
3. `th:replace="~{fragments/form :: fields}"` injeta os cinco campos. Note que o `<form>` (com `th:object`) fica na template chamadora, não no fragmento; é assim que `*{title}` sabe de onde vem.
4. Se houver erro de validação, o controller retorna `"new"` com o mesmo `form` preenchido e o `BindingResult`. Os valores digitados não são perdidos.

## Passo 8 — `templates/details.html`

```html
<!DOCTYPE html>
<html lang="pt-BR" xmlns:th="http://www.thymeleaf.org">
<head th:replace="~{fragments/layout :: head('Detalhes da sugestão')}"></head>
<body>
<nav th:replace="~{fragments/layout :: nav}"></nav>
<main class="container page-narrow py-5"><a class="back-link" th:href="@{/}"><i class="bi bi-arrow-left me-2"></i>Voltar
    às sugestões</a>
    <article class="paper-card p-4 p-md-5 mt-4">
        <div class="d-flex flex-wrap gap-2 mb-4"><span class="category-tag"
                                                       th:text="${suggestion.category.name}"></span><span
                class="category-tag neutral-tag" th:text="${suggestion.course.name}"></span></div>
        <h1 class="page-title" th:text="${suggestion.title}"></h1>
        <p class="text-muted mt-3 mb-4"><i class="bi bi-person me-1"></i> <span th:text="${suggestion.author}"></span> ·
            <i class="bi bi-calendar3 ms-2 me-1"></i><span
                    th:text="${#temporals.format(suggestion.createdAt,'dd/MM/yyyy')}"></span></p>
        <hr>
        <p class="article-copy mt-4" th:text="${suggestion.content}"></p><a th:if="${isAdmin}"
                                                                             class="btn btn-brand rounded-pill mt-4"
                                                                             th:href="@{/admin/suggestions/{id}/edit(id=${suggestion.id})}"><i
            class="bi bi-pencil me-2"></i>Editar</a></article>
</main>
<footer th:replace="~{fragments/layout :: footer}"></footer>
<div th:replace="~{fragments/layout :: scripts}"></div>
</body>
</html>
```

### Leitura guidada

1. `#temporals.format(suggestion.createdAt,'dd/MM/yyyy')` converte o `Instant` da API para uma data legível. O formato usa `dd/MM/yyyy`, não `DD/MM/YYYY` (em `DateTimeFormatter`, `DD` é dia do ano).
2. `th:if="${isAdmin}"` esconde o botão **Editar** de visitantes. O visitante ainda consegue abrir a URL diretamente? Não na Parte 3, porque `/admin/**` será protegido.
3. `th:text="${suggestion.content}"` preserva o texto, e o CSS `.article-copy` cuida das quebras com `white-space: pre-wrap`.
4. Se o controller mock receber um ID que não existe, ele devolve o primeiro item. Na Parte 3, um ID inexistente gera um `ApiException(404)` e a template `error.html` entra em ação.

## Passo 9 — `templates/login.html`

```html
<!DOCTYPE html>
<html lang="pt-BR" xmlns:th="http://www.thymeleaf.org">
<head th:replace="~{fragments/layout :: head('Acesso administrativo')}"></head>
<body>
<nav th:replace="~{fragments/layout :: nav}"></nav>
<main class="container login-shell py-5">
    <div class="paper-card p-4 p-md-5">
        <div class="login-icon"><i class="bi bi-shield-lock-fill"></i></div>
        <span class="eyebrow">ACESSO RESTRITO</span>
        <h1 class="page-title mt-2">Bem-vindo de volta.</h1>
        <p class="text-muted mb-4">Acesse o painel para gerenciar as sugestões da comunidade.</p>
        <div th:if="${param.expired}" class="alert alert-warning">Sua sessão expirou. Entre novamente.</div>
        <div th:if="${error}" class="alert alert-danger" th:text="${error}"></div>
        <form th:object="${loginForm}" th:action="@{/login}" method="post">
            <div class="mb-3"><label class="form-label fw-semibold" for="username">Usuário</label><input
                    class="form-control form-control-lg" id="username" th:field="*{username}" autocomplete="username"
                    required></div>
            <div class="mb-4"><label class="form-label fw-semibold" for="password">Senha</label><input type="password"
                                                                                                        class="form-control form-control-lg"
                                                                                                        id="password"
                                                                                                        th:field="*{password}"
                                                                                                        autocomplete="current-password"
                                                                                                        required></div>
            <button class="btn btn-brand btn-lg w-100 rounded-pill" type="submit">Entrar no painel <i
                    class="bi bi-arrow-right ms-2"></i></button>
        </form>
        <p class="text-muted text-center small mt-4 mb-0"><i class="bi bi-lock me-1"></i> Acesso exclusivo de
            administradores</p></div>
</main>
<footer th:replace="~{fragments/layout :: footer}"></footer>
<div th:replace="~{fragments/layout :: scripts}"></div>
</body>
</html>
```

### Leitura guidada

1. `${param.expired}` é o objeto implícito `param`, que representa parâmetros da query string. A URL `/login?expired` faz este alerta aparecer. Não é atributo do `Model`.
2. `${error}` é atributo do `Model`. Na Parte 3, o controller coloca `"Usuário ou senha inválidos."` quando a API responde 401.
3. `th:object="${loginForm}"` + `th:field="*{username}"` e `*{password}` vinculam o formulário. Não escreva `name=` à mão.
4. O alerta de erro não usa `fragments/layout :: alerts` porque fica dentro do cartão de login, com layout próprio.

## Passo 10 — `templates/error.html`

```html
<!DOCTYPE html>
<html lang="pt-BR" xmlns:th="http://www.thymeleaf.org">
<head th:replace="~{fragments/layout :: head('Não foi possível continuar')}"></head>
<body>
<nav th:replace="~{fragments/layout :: nav}"></nav>
<main class="container text-center py-5">
    <div class="py-5"><span class="display-1 fw-bold text-brand" th:text="${status ?: 'Ops'}"></span>
        <h1 class="page-title mt-3">Algo não saiu como esperado.</h1>
        <p class="text-muted mt-3" th:text="${message ?: 'Tente novamente em instantes.'}"></p><a
                class="btn btn-brand rounded-pill px-4 mt-3" th:href="@{/}">Voltar ao início</a></div>
</main>
<footer th:replace="~{fragments/layout :: footer}"></footer>
<div th:replace="~{fragments/layout :: scripts}"></div>
</body>
</html>
```

`?:` é o **operador Elvis**: usa o valor da esquerda se ele não for nulo; senão usa o da direita. Assim, `${status ?: 'Ops'}` mostra `404`, `503` etc. quando o controller fornece o status, e `Ops` quando não fornece. O mesmo vale para a mensagem.

Esta template não é chamada pelo controller mock. Ela entra em cena na Parte 3, quando `@ExceptionHandler(ApiException.class)` devolver `"error"`.

## Passo 11 — `templates/admin/dashboard.html`

```html
<!DOCTYPE html>
<html lang="pt-BR" xmlns:th="http://www.thymeleaf.org">
<head th:replace="~{fragments/layout :: head('Painel administrativo')}"></head>
<body>
<nav th:replace="~{fragments/layout :: nav}"></nav>
<div th:replace="~{fragments/layout :: alerts}"></div>
<main class="container py-5">
    <div class="d-flex flex-column flex-md-row justify-content-between gap-3 align-items-md-end mb-4">
        <div><span class="eyebrow">PAINEL ADMINISTRATIVO</span>
            <h1 class="page-title mt-2">Gerenciar sugestões<span class="text-brand">.</span></h1>
            <p class="text-muted mb-0">Organize e acompanhe as ideias da comunidade.</p></div>
        <a th:href="@{/}" class="btn btn-outline-dark rounded-pill px-4"><i class="bi bi-box-arrow-up-right me-2"></i>Ver
            mural público</a></div>
    <div class="row g-3 mb-4">
        <div class="col-md-4">
            <div class="stat-card">
                <div><span class="text-muted">Sugestões registradas</span>
                    <h2 class="display-5 fw-bold mb-0" th:text="${#lists.size(suggestions)}">0</h2></div>
                <div class="stat-icon"><i class="bi bi-chat-left-text"></i></div>
            </div>
        </div>
        <div class="col-md-8">
            <div class="stat-card stat-banner">
                <div><span class="eyebrow">GESTÃO DE IDEIAS</span>
                    <h2 class="h5 fw-bold mt-2">Cada sugestão é uma oportunidade de melhorar.</h2></div>
                <i class="bi bi-stars fs-1"></i></div>
        </div>
    </div>
    <div class="paper-card table-responsive">
        <table class="table table-hover align-middle mb-0">
            <thead>
            <tr>
                <th>Ideia</th>
                <th>Categoria</th>
                <th>Curso</th>
                <th>Autor</th>
                <th class="text-end">Ações</th>
            </tr>
            </thead>
            <tbody>
            <tr th:each="s:${suggestions}">
                <td><a class="fw-semibold text-decoration-none text-dark" th:href="@{/suggestions/{id}(id=${s.id})}"
                       th:text="${s.title}"></a>
                    <div class="small text-muted" th:text="'ID #' + ${s.id}"></div>
                </td>
                <td><span class="category-tag" th:text="${s.category.name}"></span></td>
                <td th:text="${s.course.name}"></td>
                <td th:text="${s.author}"></td>
                <td class="text-end">
                    <div class="d-flex gap-2 justify-content-end"><a class="btn btn-sm btn-outline-dark rounded-pill"
                                                                     th:href="@{/admin/suggestions/{id}/edit(id=${s.id})}"><i
                            class="bi bi-pencil me-1"></i>Editar</a>
                        <form method="post" th:action="@{/admin/suggestions/{id}/delete(id=${s.id})}"
                              onsubmit="return confirm('Excluir esta sugestão? Esta operação não pode ser desfeita.');">
                            <button type="submit" class="btn btn-sm btn-outline-danger rounded-pill"><i
                                    class="bi bi-trash me-1"></i>Excluir
                            </button>
                        </form>
                    </div>
                </td>
            </tr>
            <tr th:if="${#lists.isEmpty(suggestions)}">
                <td colspan="5" class="text-muted text-center py-5">Ainda não há sugestões cadastradas.</td>
            </tr>
            </tbody>
        </table>
    </div>
</main>
<footer th:replace="~{fragments/layout :: footer}"></footer>
<div th:replace="~{fragments/layout :: scripts}"></div>
</body>
</html>
```

### Leitura guidada

1. O `th:each` está na `<tr>`, então a linha inteira se repete.
2. Cada link usa path variable com o ID corrente: `@{/admin/suggestions/{id}/edit(id=${s.id})}`.
3. O `id` aparece duas vezes: no link e no texto `ID #`. O texto usa concatenação de string.
4. O formulário de exclusão é `POST` com `th:action`, então recebe CSRF. O `onsubmit` é JavaScript puro — o mesmo papel do `data-confirm` da Parte 1.
5. A linha de estado vazio tem `colspan="5"`, porque a tabela tem cinco colunas. Se você adicionar uma coluna, atualize o `colspan`.
6. Aqui também vemos `alerts`: após excluir, o controller mock usa `flash.addFlashAttribute("success", ...)` e o alerta verde aparece no topo.

## Passo 12 — `templates/admin/edit.html`

```html
<!DOCTYPE html>
<html lang="pt-BR" xmlns:th="http://www.thymeleaf.org">
<head th:replace="~{fragments/layout :: head('Editar sugestão')}"></head>
<body>
<nav th:replace="~{fragments/layout :: nav}"></nav>
<main class="container page-narrow py-5"><a class="back-link" th:href="@{/admin/suggestions}"><i
        class="bi bi-arrow-left me-2"></i>Voltar ao painel</a>
    <div class="mt-4 mb-4"><span class="eyebrow">ADMINISTRAÇÃO · EDIÇÃO</span>
        <h1 class="page-title mt-2">Refinar uma <span class="serif-accent">ideia.</span></h1>
        <p class="text-muted">Atualize as informações da sugestão selecionada.</p></div>
    <div th:replace="~{fragments/layout :: alerts}"></div>
    <form class="paper-card p-4 p-md-5" method="post" th:action="@{/admin/suggestions/{id}/edit(id=${suggestionId})}"
          th:object="${form}">
        <div th:replace="~{fragments/form :: fields}"></div>
        <div class="d-flex justify-content-between align-items-center flex-wrap gap-3 mt-4"><a
                class="text-muted text-decoration-none" th:href="@{/admin/suggestions}">Cancelar</a>
            <button type="submit" class="btn btn-brand btn-lg rounded-pill px-4">Salvar alterações <i
                    class="bi bi-check-lg ms-2"></i></button>
        </div>
    </form>
</main>
<footer th:replace="~{fragments/layout :: footer}"></footer>
<div th:replace="~{fragments/layout :: scripts}"></div>
</body>
</html>
```

### Leitura guidada

1. É estruturalmente igual a `new.html`. A diferença está no `action`, que usa `${suggestionId}`, e no texto do botão.
2. `suggestionId` vai no `Model` porque o ID da URL não pertence ao `SuggestionForm` — o formulário só carrega campos editáveis.
3. Ambos os controllers (`GET` e `POST` de edição) precisam recolocar `suggestionId` no `Model` quando há erro de validação; senão o `th:action` gera uma URL sem ID.
4. O reuso de `fragments/form :: fields` mostra o valor principal dos fragments: um ajuste em um campo (por exemplo, trocar o `placeholder`) reflete em cadastro e edição ao mesmo tempo.

## Checklist da conversão

Marque item por item antes de seguir para a Parte 3:

- [ ] `xmlns:th` presente nas nove templates.
- [ ] Nenhum `href="/..."` ou `src="/..."` fixo: tudo virou `@{...}`.
- [ ] Cabeçalho, navbar, alertas, rodapé e scripts vêm de `layout.html`.
- [ ] `head(title)` recebe um título diferente em cada página.
- [ ] Cartões e linhas da tabela saem de `th:each`, não de HTML repetido.
- [ ] Estado vazio usa `#lists.isEmpty(...)`.
- [ ] Filtros preservam `q`, `courseId` e `categoryId` com `th:value` / `th:selected`.
- [ ] Formulários usam `th:object` + `th:field`; nenhum `name=` escrito à mão nos campos vinculados.
- [ ] `th:errors` aparece para os quatro campos obrigatórios.
- [ ] Data formatada com `#temporals.format(..., 'dd/MM/yyyy')`.
- [ ] Links e ações com ID usam `@{/caminho/{id}(id=${...})}`.
- [ ] Botões de admin dependem de `isAdmin`.
- [ ] `error.html` usa `?:` para status e mensagem.
- [ ] `static/prototipo/` foi apagado.
- [ ] HTML entregue ao navegador não contém nenhum `th:`.

## Erros comuns e diagnóstico

| Sintoma | Causa provável | Correção |
|---|---|---|
| `Error resolving template [home]` | arquivo fora de `templates/` ou nome diferente | conferir `return "home"` e `templates/home.html` |
| Atributos `th:` aparecem no código-fonte | template servida como arquivo estático | apagar `static/prototipo/` e acessar `/`, não `/prototipo/index.html` |
| `Link base` ou CSS 404 | URL escrita como `/css/site.css` sem `@{}` | usar `th:href="@{/css/site.css}"` |
| `Property or field 'title' cannot be found` | `th:field="*{title}"` sem `th:object` | colocar `th:object="${form}"` no `<form>` |
| Select não mantém a opção escolhida | `th:selected` usado em select vinculado | trocar por `th:field="*{courseId}"` no `<select>` |
| Filtro volta vazio após enviar | select de filtro sem `th:selected` | comparar `${courseId == c.id}` |
| Erros de validação não aparecem | controller esqueceu `BindingResult` | parâmetro `BindingResult` deve vir logo após `@Valid @ModelAttribute` |
| `/admin/suggestions` redireciona para `/login` | `SecurityConfig` final ainda ativa | usar a configuração temporária da Parte 2 |
| 403 ao enviar POST | CSRF ausente | usar `th:action` nos formulários; não `action=` fixo |
| Data aparece como `2026-10-08T12:00:00Z` | `th:text` direto no `Instant` | usar `#temporals.format(...,'dd/MM/yyyy')` |
| Fragmento não encontrado | nome ou caminho errado | `~{fragments/layout :: nav}` exige `templates/fragments/layout.html` e `th:fragment="nav"` |

## Checkpoint da Parte 2

1. Rode `./gradlew bootRun` e abra `http://localhost:8081/`.
2. Confira três cartões e o contador `3 sugestões encontradas`.
3. Abra `http://localhost:8081/?courseId=1&q=tomadas`: os campos voltam preenchidos e os três cartões continuam aparecendo — o filtro real virá na Parte 3.
4. Envie o formulário de cadastro em branco e veja as mensagens de validação dentro do próprio formulário.
5. Envie com dados válidos: o alerta verde aparece no mural depois do redirect.
6. Acesse `/suggestions/1`: confira título, autor, curso, categoria e data `08/10/2026`.
7. Mude `isAdmin()` para `false`, recarregue e confirme que **Editar** e **Painel admin** desaparecem. Volte para `true`.
8. Abra `/admin/suggestions`, clique em **Editar**, altere o título e salve: alerta verde no painel.
9. Clique em **Excluir**, cancele e depois confirme: o alerta aparece.
10. Abra `/login?expired`: o aviso amarelo aparece. Preencha usuário e senha quaisquer e envie: aparece a mensagem de login simulado.
11. Abra o código-fonte de `/` e procure por `th:` — não deve encontrar nenhum.
12. Apague `src/main/resources/static/prototipo/`.

Tudo certo? As templates estão prontas. Continue em [Parte 3 — integração com controllers e API](parte-3-integracao-api.md).
