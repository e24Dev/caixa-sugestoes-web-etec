# Parte 1 — Interface estática (HTML, CSS e um pouco de JS)

[Voltar ao README](../../README.md) · [Próxima: Parte 2 — Thymeleaf](parte-2-thymeleaf.md)

## Objetivo

Construir as **seis telas** da aplicação apenas com HTML, CSS e JavaScript de navegador. Nesta etapa:

- não existe controller Spring;
- não existe chamada à API;
- não existe banco de dados;
- os dados são texto fixo escrito no próprio HTML;
- nenhum atributo `th:` aparece nos arquivos.

O resultado é um protótipo navegável. Ele servirá de matéria-prima para a Parte 2, onde o mesmo markup será adaptado para Thymeleaf.

## Convenção de pastas

Crie esta estrutura dentro do projeto:

```text
src/main/resources/static/
├── css/
│   └── site.css
└── prototipo/
    ├── index.html
    ├── new.html
    ├── details.html
    ├── login.html
    ├── js/
    │   └── ui.js
    └── admin/
        ├── dashboard.html
        └── edit.html
```

Usamos `static/prototipo/` em vez de colocar `index.html` diretamente em `static/` por dois motivos:

1. na Parte 2, a rota `/` passará a ser atendida pelo controller e um `static/index.html` poderia competir com ela;
2. a regra de segurança da aplicação protege `/admin/**`; ao colocar o protótipo em `/prototipo/admin/...`, você consegue visualizar a tela administrativa sem precisar entrar com login nesta etapa.

> As URLs `/webjars/...` e `/css/site.css` só funcionam quando o Spring Boot está executando. Por isso, abra o protótipo em `http://localhost:8081/prototipo/index.html`, e não clicando no arquivo com `file://`.

## Passo 1 — CSS (`static/css/site.css`)

Copie o arquivo `src/main/resources/static/css/site.css` deste repositório para obter o visual final completo. Para entender o que está sendo usado, esta é uma versão reduzida com as classes que aparecem nas telas:

```css
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700;800&family=DM+Serif+Display:ital@0;1&display=swap');

:root {
  --ink: #38262d;
  --muted: #79666e;
  --brand: #800020;
  --light: #faf0f3;
  --line: #eee2e6;
}

* { box-sizing: border-box; }
body { font-family: 'DM Sans', system-ui, sans-serif; color: var(--ink); background: #fdfdfa; }
h1, h2, h3 { letter-spacing: -.035em; }

.text-brand { color: var(--brand) !important; }
.site-nav { background: rgba(255,255,252,.95); backdrop-filter: blur(14px); border-bottom: 1px solid var(--line); }
.navbar-brand { font-weight: 800; letter-spacing: -.07em; font-size: 1.48rem; }
.brand-mark { display: grid; place-items: center; background: var(--brand); color: #fff; width: 42px; height: 42px; border-radius: 13px; font-size: 1.18rem; }
.brand-subtitle { font-size: .48rem; letter-spacing: .2em; color: #806c74; }
.nav-link { font-weight: 600; font-size: .91rem; color: #5e4650; }

.btn-brand { background: var(--brand); border-color: var(--brand); color: #fff; font-weight: 700; }
.btn-brand:hover, .btn-brand:focus { background: #650019; border-color: #650019; color: #fff; }

.hero { padding: 7.5rem 0 6.5rem; background: radial-gradient(circle at 89% 44%, #f4e0e7 0, transparent 42%), linear-gradient(145deg, #fffefa, #fcf6f8); overflow: hidden; }
.eyebrow { font-size: .72rem; font-weight: 800; letter-spacing: .16em; color: var(--brand); }
.serif-accent { font-family: 'DM Serif Display', Georgia, serif; color: var(--brand); font-weight: 400; font-style: italic; }
.hero-copy { max-width: 540px; line-height: 1.75; }

.hero-art { height: 420px; position: relative; display: grid; place-items: center; }
.art-orbit { position: absolute; border: 1px solid #dfb8c5; border-radius: 50%; transform: rotate(-24deg); }
.orbit-a { height: 370px; width: 370px; }
.orbit-b { height: 270px; width: 270px; transform: rotate(35deg); }
.art-center { height: 232px; width: 232px; border-radius: 42% 58% 56% 44%; background: var(--brand); display: grid; place-items: center; color: #fff; font-size: 6.2rem; transform: rotate(-12deg); box-shadow: 0 30px 65px #80002031; }
.art-center i { transform: rotate(12deg); }
.art-card { position: absolute; z-index: 2; display: flex; align-items: center; gap: 12px; background: #fff; box-shadow: 0 18px 65px #4b15281a; border-radius: 16px; padding: 16px 19px; font-weight: 700; }
.art-card-top { top: 55px; right: 0; transform: rotate(7deg); }
.art-card-top i { color: #e5a83c; font-size: 1.5rem; }
.art-card-bottom { bottom: 65px; left: -20px; transform: rotate(-5deg); }
.art-card-bottom i { color: var(--brand); }
.art-spark { position: absolute; font-size: 38px; color: #dda655; }
.spark-a { top: 18px; left: 65px; }
.spark-b { bottom: 20px; right: 65px; color: #b4788b; }

.section-heading { font-size: 2.65rem; font-weight: 800; }
.count-pill, .category-tag { background: #f6e6ec; color: var(--brand); border-radius: 999px; font-weight: 700; font-size: .75rem; padding: 8px 12px; }
.count-pill { padding: 10px 15px; }
.filter-panel { padding: 25px 28px; background: #fff; border: 1px solid var(--line); border-radius: 22px; box-shadow: 0 9px 28px #4b152810; }
.form-control, .form-select, .input-group-text { border-color: #e9dfe3; }
.form-control:focus, .form-select:focus { box-shadow: 0 0 0 .18rem #80002026; border-color: #b45f79; }

.idea-card { position: relative; background: #fff; border: 1px solid var(--line); border-radius: 22px; padding: 27px; transition: transform .2s, box-shadow .2s; min-height: 285px; }
.idea-card:hover { transform: translateY(-4px); box-shadow: 0 18px 40px #4b152814; }
.card-seq { font-size: .68rem; letter-spacing: .08em; color: #a8999f; font-weight: 800; }
.idea-excerpt { color: var(--muted); font-size: .92rem; line-height: 1.7; min-height: 64px; display: -webkit-box; -webkit-line-clamp: 3; -webkit-box-orient: vertical; overflow: hidden; }
.idea-meta { display: flex; flex-wrap: wrap; gap: 13px; font-size: .75rem; color: #88737c; border-top: 1px solid #f1e9ec; padding-top: 14px; margin-top: 20px; }
.view-link { display: inline-block; color: var(--brand); text-decoration: none; font-weight: 700; font-size: .82rem; margin-top: 22px; }
.empty-state { padding: 90px 20px; }
.cta-strip { background: var(--brand); color: #fff; padding: 62px 0; }
.footer { background: #faf4f6; border-top: 1px solid #ece0e4; }

.page-narrow { max-width: 920px; }
.page-title { font-size: clamp(2.3rem, 5vw, 3.6rem); font-weight: 800; }
.back-link { color: var(--brand); text-decoration: none; font-weight: 700; font-size: .9rem; }
.paper-card { background: #fff; border-radius: 24px; border: 1px solid var(--line); box-shadow: 0 15px 45px #4b152810; }
.article-copy { font-size: 1.09rem; line-height: 1.95; white-space: pre-wrap; }
.neutral-tag { background: #f3f0f1; color: #66545c; }

.login-shell { max-width: 560px; }
.login-icon { display: grid; place-items: center; font-size: 1.6rem; color: var(--brand); background: var(--light); width: 58px; height: 58px; border-radius: 18px; margin-bottom: 22px; }

.stat-card { background: #fff; padding: 24px; border: 1px solid var(--line); border-radius: 22px; display: flex; justify-content: space-between; align-items: center; min-height: 145px; }
.stat-icon { background: #f6e6ec; color: var(--brand); font-size: 2rem; padding: 14px 19px; border-radius: 18px; }
.stat-banner { background: #f7e8ee; }
.table thead { background: #faf4f6; font-size: .72rem; text-transform: uppercase; letter-spacing: .08em; color: #806b74; }
.table thead th { background: #faf4f6; }

@media (max-width: 767px) {
  .hero { padding: 4rem 0; }
  .hero-art { height: 320px; }
  .art-center { height: 180px; width: 180px; font-size: 4.8rem; }
  .orbit-a { height: 295px; width: 295px; }
  .orbit-b { height: 215px; width: 215px; }
  .art-card { font-size: .76rem; padding: 12px; }
  .section-heading { font-size: 2rem; }
  .filter-panel { padding: 20px; }
  .idea-card { min-height: 0; }
}
```

Pontos para observar:

- as cores vivem em variáveis CSS (`--brand`, `--ink`, `--line`); trocar `--brand` recoloriza a interface inteira;
- Bootstrap fornece grid, botões, formulários, tabela e navbar; `site.css` só ajusta identidade visual;
- `DM Serif Display` é usada apenas no destaque `.serif-accent`; se o Google Fonts falhar, `Georgia` assume;
- `.article-copy { white-space: pre-wrap; }` preserva quebras de linha do texto da sugestão.

## Passo 2 — JavaScript (`static/prototipo/js/ui.js`)

O JavaScript desta etapa é pequeno e serve apenas para simular interação. Ele não conversa com servidor nenhum:

```js
document.addEventListener('DOMContentLoaded', () => {
  // Filtro simulado: esconde cartões no navegador.
  const filterForm = document.querySelector('[data-filter-form]');
  if (filterForm) {
    filterForm.addEventListener('submit', (event) => {
      event.preventDefault();

      const q = filterForm.querySelector('[name="q"]').value.trim().toLowerCase();
      const course = filterForm.querySelector('[name="courseId"]').value;
      const category = filterForm.querySelector('[name="categoryId"]').value;
      let visible = 0;

      document.querySelectorAll('[data-card]').forEach((card) => {
        const matchesText = !q || card.dataset.search.includes(q);
        const matchesCourse = !course || card.dataset.course === course;
        const matchesCategory = !category || card.dataset.category === category;
        const show = matchesText && matchesCourse && matchesCategory;

        card.classList.toggle('d-none', !show);
        if (show) visible += 1;
      });

      const counter = document.querySelector('[data-counter]');
      if (counter) counter.textContent = `${visible} sugestões encontradas`;
    });
  }

  // Confirmação antes de excluir na tela administrativa.
  document.querySelectorAll('form[data-confirm]').forEach((form) => {
    form.addEventListener('submit', (event) => {
      if (!confirm(form.dataset.confirm)) event.preventDefault();
    });
  });
});
```

O menu colapsável do navbar não precisa de código nosso: o bundle do Bootstrap já faz isso. Ao final de cada página incluiremos:

```html
<script src="/webjars/bootstrap/5.3.3/js/bootstrap.bundle.min.js"></script>
<script src="/prototipo/js/ui.js"></script>
```

> Na Parte 2, o filtro simulado desaparece: ele passa a ser feito no servidor com `GET /`. O `confirm()` permanece, mas escrito diretamente no formulário do template final.

## Passo 3 — `static/prototipo/index.html` (mural)

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Ideias que transformam · Caixa de Sugestões</title>
  <link href="/webjars/bootstrap/5.3.3/css/bootstrap.min.css" rel="stylesheet">
  <link href="/webjars/bootstrap-icons/1.11.3/font/bootstrap-icons.css" rel="stylesheet">
  <link href="/css/site.css" rel="stylesheet">
</head>
<body>
<nav class="navbar navbar-expand-lg navbar-light sticky-top site-nav">
  <div class="container py-2">
    <a class="navbar-brand d-flex align-items-center gap-2" href="index.html">
      <span class="brand-mark"><i class="bi bi-chat-square-heart-fill"></i></span>
      <span>caixa<span class="text-brand">ideias</span>
        <small class="brand-subtitle d-block">VOZES QUE TRANSFORMAM</small>
      </span>
    </a>
    <button class="navbar-toggler border-0" type="button" data-bs-toggle="collapse"
            data-bs-target="#navigation" aria-controls="navigation" aria-expanded="false"
            aria-label="Abrir navegação"><span class="navbar-toggler-icon"></span></button>
    <div class="collapse navbar-collapse" id="navigation">
      <div class="navbar-nav ms-auto align-items-lg-center gap-lg-3">
        <a class="nav-link" href="index.html">Explorar ideias</a>
        <a class="nav-link" href="new.html">Enviar sugestão</a>
        <a class="btn btn-sm btn-outline-dark rounded-pill px-4 ms-lg-2" href="login.html">
          <i class="bi bi-shield-lock me-1"></i> Área administrativa
        </a>
      </div>
    </div>
  </div>
</nav>

<section class="hero">
  <div class="container">
    <div class="row align-items-center gy-5">
      <div class="col-lg-7">
        <span class="eyebrow"><i class="bi bi-stars me-1"></i> UM ESPAÇO PARA SER OUVIDO</span>
        <h1 class="display-3 fw-bold lh-sm mt-3">Toda grande mudança<br>começa com
          <span class="serif-accent">uma ideia.</span></h1>
        <p class="lead text-secondary mt-4 mb-4 hero-copy">Sua voz faz a diferença. Compartilhe sugestões para
          tornar nossa comunidade cada vez melhor.</p>
        <div class="d-flex gap-3 flex-wrap">
          <a class="btn btn-brand btn-lg rounded-pill px-4" href="new.html">
            Compartilhar uma ideia <i class="bi bi-arrow-up-right ms-2"></i></a>
          <a class="btn btn-outline-dark btn-lg rounded-pill px-4" href="#ideias">Explorar sugestões</a>
        </div>
      </div>
      <div class="col-lg-5">
        <div class="hero-art">
          <div class="art-orbit orbit-a"></div>
          <div class="art-orbit orbit-b"></div>
          <div class="art-card art-card-top"><i class="bi bi-lightbulb-fill"></i><span>Novas possibilidades</span></div>
          <div class="art-center"><i class="bi bi-chat-square-heart-fill"></i></div>
          <div class="art-card art-card-bottom"><span>Ideias viram <strong>impacto</strong></span><i class="bi bi-arrow-up-right"></i></div>
          <div class="art-spark spark-a">✳</div>
          <div class="art-spark spark-b">✦</div>
        </div>
      </div>
    </div>
  </div>
</section>

<section class="container pt-5" id="ideias">
  <div class="d-flex justify-content-between align-items-end flex-wrap gap-3 mb-4">
    <div>
      <span class="eyebrow">MURAL DA COMUNIDADE</span>
      <h2 class="section-heading mt-2">Ideias em destaque<span class="text-brand">.</span></h2>
      <p class="text-muted mb-0">Explore, descubra e inspire-se com novas sugestões.</p>
    </div>
    <span class="count-pill" data-counter>3 sugestões encontradas</span>
  </div>

  <form class="filter-panel row g-3 align-items-end" method="get" data-filter-form>
    <div class="col-lg-5">
      <label for="q" class="form-label small fw-semibold">O que você procura?</label>
      <div class="input-group">
        <span class="input-group-text border-end-0 bg-white"><i class="bi bi-search"></i></span>
        <input id="q" class="form-control border-start-0" name="q" placeholder="Pesquise por um tema...">
      </div>
    </div>
    <div class="col-lg-3">
      <label for="courseId" class="form-label small fw-semibold">Curso</label>
      <select id="courseId" class="form-select" name="courseId">
        <option value="">Todos os cursos</option>
        <option value="1">Desenvolvimento de Sistemas</option>
        <option value="2">Eletromecânica</option>
      </select>
    </div>
    <div class="col-lg-2">
      <label for="categoryId" class="form-label small fw-semibold">Categoria</label>
      <select id="categoryId" class="form-select" name="categoryId">
        <option value="">Todas</option>
        <option value="1">Infraestrutura</option>
        <option value="2">Ensino</option>
      </select>
    </div>
    <div class="col-lg-2">
      <button class="btn btn-brand w-100" type="submit">Filtrar <i class="bi bi-arrow-right ms-2"></i></button>
    </div>
  </form>

  <div class="row g-4 mt-2">
    <div class="col-md-6 col-lg-4" data-card data-course="1" data-category="1"
         data-search="mais tomadas na biblioteca estudo tomadas">
      <article class="idea-card h-100">
        <div class="d-flex justify-content-between align-items-center mb-4">
          <span class="category-tag"><i class="bi bi-bookmark me-1"></i>Infraestrutura</span>
          <span class="card-seq">IDEIA 1</span>
        </div>
        <h3 class="h5 fw-bold">Mais tomadas na biblioteca</h3>
        <p class="idea-excerpt">Instalar tomadas nas mesas de estudo para que todos possam usar seus computadores...</p>
        <div class="idea-meta">
          <span><i class="bi bi-mortarboard me-1"></i>Desenvolvimento de Sistemas</span>
          <span><i class="bi bi-person me-1"></i>Anônimo</span>
        </div>
        <a class="stretched-link view-link" href="details.html">Ler sugestão <i class="bi bi-arrow-up-right"></i></a>
      </article>
    </div>

    <div class="col-md-6 col-lg-4" data-card data-course="1" data-category="2"
         data-search="criar um clube de estudos semana estudos">
      <article class="idea-card h-100">
        <div class="d-flex justify-content-between align-items-center mb-4">
          <span class="category-tag"><i class="bi bi-bookmark me-1"></i>Ensino</span>
          <span class="card-seq">IDEIA 2</span>
        </div>
        <h3 class="h5 fw-bold">Clube de estudos semanal</h3>
        <p class="idea-excerpt">Um encontro semanal para revisar conteúdos e trocar dicas entre turmas...</p>
        <div class="idea-meta">
          <span><i class="bi bi-mortarboard me-1"></i>Desenvolvimento de Sistemas</span>
          <span><i class="bi bi-person me-1"></i>Maria Silva</span>
        </div>
        <a class="stretched-link view-link" href="details.html">Ler sugestão <i class="bi bi-arrow-up-right"></i></a>
      </article>
    </div>

    <div class="col-md-6 col-lg-4" data-card data-course="2" data-category="1"
         data-search="melhorar a iluminação dos corredores iluminação">
      <article class="idea-card h-100">
        <div class="d-flex justify-content-between align-items-center mb-4">
          <span class="category-tag"><i class="bi bi-bookmark me-1"></i>Infraestrutura</span>
          <span class="card-seq">IDEIA 3</span>
        </div>
        <h3 class="h5 fw-bold">Iluminação dos corredores</h3>
        <p class="idea-excerpt">Trocar as lâmpadas antigas por LED para melhorar a visibilidade e reduzir custo...</p>
        <div class="idea-meta">
          <span><i class="bi bi-mortarboard me-1"></i>Eletromecânica</span>
          <span><i class="bi bi-person me-1"></i>Anônimo</span>
        </div>
        <a class="stretched-link view-link" href="details.html">Ler sugestão <i class="bi bi-arrow-up-right"></i></a>
      </article>
    </div>
  </div>

  <div class="cta-strip mt-5">
    <div class="container d-flex flex-column flex-lg-row align-items-lg-center justify-content-between gap-3">
      <div>
        <span class="eyebrow text-white">SUA VEZ DE PARTICIPAR</span>
        <h2 class="fw-bold mt-2 mb-0">E se a próxima grande ideia for sua?</h2>
      </div>
      <a class="btn btn-light rounded-pill px-4 py-3" href="new.html">Quero participar <i class="bi bi-arrow-up-right ms-2"></i></a>
    </div>
  </div>

  <footer class="py-5 mt-5 footer">
    <div class="container d-flex flex-column flex-md-row align-items-start justify-content-between gap-3">
      <div>
        <strong><i class="bi bi-chat-square-heart-fill text-brand me-2"></i>caixaideias</strong>
        <p class="text-muted small mt-2 mb-0">Pequenas ideias. Grandes mudanças.</p>
      </div>
      <div class="small text-muted">Projeto educacional · Spring Boot · Thymeleaf</div>
    </div>
  </footer>
</div>

<script src="/webjars/bootstrap/5.3.3/js/bootstrap.bundle.min.js"></script>
<script src="/prototipo/js/ui.js"></script>
</body>
</html>
```

Os atributos `data-card`, `data-course`, `data-category`, `data-search` e `data-counter` existem somente para o filtro simulado. Eles serão removidos na Parte 2.

## Passo 4 — `static/prototipo/new.html` (formulário público)

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Compartilhe uma sugestão · Caixa de Sugestões</title>
  <link href="/webjars/bootstrap/5.3.3/css/bootstrap.min.css" rel="stylesheet">
  <link href="/webjars/bootstrap-icons/1.11.3/font/bootstrap-icons.css" rel="stylesheet">
  <link href="/css/site.css" rel="stylesheet">
</head>
<body>
<nav class="navbar navbar-expand-lg navbar-light sticky-top site-nav">
  <div class="container py-2">
    <a class="navbar-brand d-flex align-items-center gap-2" href="index.html">
      <span class="brand-mark"><i class="bi bi-chat-square-heart-fill"></i></span>
      <span>caixa<span class="text-brand">ideias</span>
        <small class="brand-subtitle d-block">VOZES QUE TRANSFORMAM</small>
      </span>
    </a>
    <div class="navbar-nav ms-auto align-items-lg-center gap-lg-3">
      <a class="nav-link" href="index.html">Explorar ideias</a>
      <a class="nav-link" href="new.html">Enviar sugestão</a>
      <a class="btn btn-sm btn-outline-dark rounded-pill px-4" href="login.html">
        <i class="bi bi-shield-lock me-1"></i> Área administrativa
      </a>
    </div>
  </div>
</nav>

<div class="container page-narrow py-5">
  <a class="back-link" href="index.html"><i class="bi bi-arrow-left me-2"></i>Voltar ao mural</a>
  <div class="mt-4 mb-4">
    <span class="eyebrow">SUA VOZ IMPORTA</span>
    <h1 class="page-title mt-2">Compartilhe sua <span class="serif-accent">ideia.</span></h1>
    <p class="text-muted">Conte o que podemos melhorar. Sua sugestão será publicada no mural.</p>
  </div>

  <form class="paper-card p-4 p-md-5" action="#" method="post">
    <div class="row g-4">
      <div class="col-12">
        <label for="title" class="form-label fw-semibold">Título da sugestão <span class="text-danger">*</span></label>
        <input id="title" name="title" class="form-control form-control-lg"
               placeholder="Ex.: Mais espaços de estudo" maxlength="255" required>
      </div>
      <div class="col-12">
        <label for="content" class="form-label fw-semibold">Conte sua ideia <span class="text-danger">*</span></label>
        <textarea id="content" name="content" class="form-control" rows="5"
                  placeholder="Descreva o que pode melhorar e por quê..." required></textarea>
      </div>
      <div class="col-md-6">
        <label for="courseId" class="form-label fw-semibold">Curso <span class="text-danger">*</span></label>
        <select id="courseId" name="courseId" class="form-select form-select-lg" required>
          <option value="">Selecione seu curso</option>
          <option value="1">Desenvolvimento de Sistemas</option>
          <option value="2">Eletromecânica</option>
        </select>
      </div>
      <div class="col-md-6">
        <label for="categoryId" class="form-label fw-semibold">Categoria <span class="text-danger">*</span></label>
        <select id="categoryId" name="categoryId" class="form-select form-select-lg" required>
          <option value="">Selecione a categoria</option>
          <option value="1">Infraestrutura</option>
          <option value="2">Ensino</option>
        </select>
      </div>
      <div class="col-12">
        <label for="author" class="form-label fw-semibold">Seu nome
          <span class="text-muted fw-normal">(opcional)</span></label>
        <input id="author" name="author" class="form-control form-control-lg"
               placeholder="Você pode participar anonimamente">
        <p class="form-text">Deixe em branco para publicar como Anônimo.</p>
      </div>
    </div>
    <div class="d-flex justify-content-between align-items-center gap-3 flex-wrap mt-4">
      <a class="text-muted text-decoration-none" href="index.html">Cancelar</a>
      <button type="submit" class="btn btn-brand btn-lg rounded-pill px-4">
        Publicar sugestão <i class="bi bi-send ms-2"></i>
      </button>
    </div>
  </form>
</div>

<footer class="py-5 mt-5 footer">
  <div class="container d-flex flex-column flex-md-row align-items-start justify-content-between gap-3">
    <div><strong><i class="bi bi-chat-square-heart-fill text-brand me-2"></i>caixaideias</strong>
      <p class="text-muted small mt-2 mb-0">Pequenas ideias. Grandes mudanças.</p></div>
    <div class="small text-muted">Projeto educacional · Spring Boot · Thymeleaf</div>
  </div>
</footer>

<script src="/webjars/bootstrap/5.3.3/js/bootstrap.bundle.min.js"></script>
<script src="/prototipo/js/ui.js"></script>
</body>
</html>
```

Repare que os nomes dos campos (`title`, `content`, `courseId`, `categoryId`, `author`) já são exatamente os campos aceitos pela API. Isso não é coincidência: na Parte 3 eles serão vinculados ao objeto `SuggestionForm`.

## Passo 5 — `static/prototipo/details.html` (detalhes)

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Detalhes da sugestão · Caixa de Sugestões</title>
  <link href="/webjars/bootstrap/5.3.3/css/bootstrap.min.css" rel="stylesheet">
  <link href="/webjars/bootstrap-icons/1.11.3/font/bootstrap-icons.css" rel="stylesheet">
  <link href="/css/site.css" rel="stylesheet">
</head>
<body>
<nav class="navbar navbar-expand-lg navbar-light sticky-top site-nav">
  <div class="container py-2">
    <a class="navbar-brand d-flex align-items-center gap-2" href="index.html">
      <span class="brand-mark"><i class="bi bi-chat-square-heart-fill"></i></span>
      <span>caixa<span class="text-brand">ideias</span></span>
    </a>
    <div class="navbar-nav ms-auto">
      <a class="nav-link" href="index.html">Voltar ao mural</a>
    </div>
  </div>
</nav>

<main class="container page-narrow py-5">
  <a class="back-link" href="index.html"><i class="bi bi-arrow-left me-2"></i>Voltar às sugestões</a>
  <article class="paper-card p-4 p-md-5 mt-4">
    <div class="d-flex flex-wrap gap-2 mb-4">
      <span class="category-tag">Infraestrutura</span>
      <span class="category-tag neutral-tag">Desenvolvimento de Sistemas</span>
    </div>
    <h1 class="page-title">Mais tomadas na biblioteca</h1>
    <p class="text-muted mt-3 mb-4">
      <i class="bi bi-person me-1"></i> Anônimo ·
      <i class="bi bi-calendar3 ms-2 me-1"></i> 08/10/2026
    </p>
    <hr>
    <p class="article-copy mt-4">Instalar tomadas nas mesas de estudo para que todos possam usar seus computadores
      durante os períodos de projeto.</p>
    <a class="btn btn-brand rounded-pill mt-4" href="admin/edit.html">
      <i class="bi bi-pencil me-2"></i>Editar
    </a>
  </article>
</main>

<footer class="py-5 mt-5 footer">
  <div class="container small text-muted">Projeto educacional · Spring Boot · Thymeleaf</div>
</footer>

<script src="/webjars/bootstrap/5.3.3/js/bootstrap.bundle.min.js"></script>
<script src="/prototipo/js/ui.js"></script>
</body>
</html>
```

O botão **Editar** aparece aqui apenas para mostrar o layout. Na Parte 2 ele será condicionado a `isAdmin`.

## Passo 6 — `static/prototipo/login.html`

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Acesso administrativo · Caixa de Sugestões</title>
  <link href="/webjars/bootstrap/5.3.3/css/bootstrap.min.css" rel="stylesheet">
  <link href="/webjars/bootstrap-icons/1.11.3/font/bootstrap-icons.css" rel="stylesheet">
  <link href="/css/site.css" rel="stylesheet">
</head>
<body>
<nav class="navbar navbar-expand-lg navbar-light sticky-top site-nav">
  <div class="container py-2">
    <a class="navbar-brand d-flex align-items-center gap-2" href="index.html">
      <span class="brand-mark"><i class="bi bi-chat-square-heart-fill"></i></span>
      <span>caixa<span class="text-brand">ideias</span></span>
    </a>
  </div>
</nav>

<main class="container login-shell py-5">
  <div class="paper-card p-4 p-md-5">
    <div class="login-icon"><i class="bi bi-shield-lock-fill"></i></div>
    <span class="eyebrow">ACESSO RESTRITO</span>
    <h1 class="page-title mt-2">Bem-vindo de volta.</h1>
    <p class="text-muted mb-4">Acesse o painel para gerenciar as sugestões da comunidade.</p>

    <div class="alert alert-danger" role="alert">Usuário ou senha inválidos.</div>

    <form action="#" method="post">
      <div class="mb-3">
        <label class="form-label fw-semibold" for="username">Usuário</label>
        <input class="form-control form-control-lg" id="username" name="username"
               autocomplete="username" required>
      </div>
      <div class="mb-4">
        <label class="form-label fw-semibold" for="password">Senha</label>
        <input type="password" class="form-control form-control-lg" id="password" name="password"
               autocomplete="current-password" required>
      </div>
      <button class="btn btn-brand btn-lg w-100 rounded-pill" type="submit">
        Entrar no painel <i class="bi bi-arrow-right ms-2"></i>
      </button>
    </form>
    <p class="text-muted text-center small mt-4 mb-0">
      <i class="bi bi-lock me-1"></i> Acesso exclusivo de administradores
    </p>
  </div>
</main>

<footer class="py-5 mt-5 footer">
  <div class="container small text-muted">Projeto educacional · Spring Boot · Thymeleaf</div>
</footer>

<script src="/webjars/bootstrap/5.3.3/js/bootstrap.bundle.min.js"></script>
<script src="/prototipo/js/ui.js"></script>
</body>
</html>
```

A mensagem de erro está fixa no HTML. Na Parte 2 ela passa a existir somente quando o controller colocar um atributo `error` no `Model`.

## Passo 7 — `static/prototipo/admin/dashboard.html`

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Painel administrativo · Caixa de Sugestões</title>
  <link href="/webjars/bootstrap/5.3.3/css/bootstrap.min.css" rel="stylesheet">
  <link href="/webjars/bootstrap-icons/1.11.3/font/bootstrap-icons.css" rel="stylesheet">
  <link href="/css/site.css" rel="stylesheet">
</head>
<body>
<nav class="navbar navbar-expand-lg navbar-light sticky-top site-nav">
  <div class="container py-2">
    <a class="navbar-brand d-flex align-items-center gap-2" href="../index.html">
      <span class="brand-mark"><i class="bi bi-chat-square-heart-fill"></i></span>
      <span>caixa<span class="text-brand">ideias</span></span>
    </a>
    <div class="navbar-nav ms-auto align-items-lg-center gap-lg-3">
      <a class="nav-link" href="../index.html">Ver mural público</a>
      <form action="../login.html" method="post" class="ms-lg-2">
        <button class="btn btn-sm btn-outline-dark rounded-pill px-4">
          Sair <i class="bi bi-box-arrow-right"></i>
        </button>
      </form>
    </div>
  </div>
</nav>

<main class="container py-5">
  <div class="d-flex flex-column flex-md-row justify-content-between gap-3 align-items-md-end mb-4">
    <div>
      <span class="eyebrow">PAINEL ADMINISTRATIVO</span>
      <h1 class="page-title mt-2">Gerenciar sugestões<span class="text-brand">.</span></h1>
      <p class="text-muted mb-0">Organize e acompanhe as ideias da comunidade.</p>
    </div>
    <a href="../index.html" class="btn btn-outline-dark rounded-pill px-4">
      <i class="bi bi-box-arrow-up-right me-2"></i>Ver mural público
    </a>
  </div>

  <div class="row g-3 mb-4">
    <div class="col-md-4">
      <div class="stat-card">
        <div><span class="text-muted">Sugestões registradas</span>
          <h2 class="display-5 fw-bold mb-0">3</h2></div>
        <div class="stat-icon"><i class="bi bi-chat-left-text"></i></div>
      </div>
    </div>
    <div class="col-md-8">
      <div class="stat-card stat-banner">
        <div>
          <span class="eyebrow">GESTÃO DE IDEIAS</span>
          <h2 class="h5 fw-bold mt-2">Cada sugestão é uma oportunidade de melhorar.</h2>
        </div>
        <i class="bi bi-stars fs-1"></i>
      </div>
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
      <tr>
        <td>
          <a class="fw-semibold text-decoration-none text-dark" href="../details.html">Mais tomadas na biblioteca</a>
          <div class="small text-muted">ID #1</div>
        </td>
        <td><span class="category-tag">Infraestrutura</span></td>
        <td>Desenvolvimento de Sistemas</td>
        <td>Anônimo</td>
        <td class="text-end">
          <div class="d-flex gap-2 justify-content-end">
            <a class="btn btn-sm btn-outline-dark rounded-pill" href="edit.html">
              <i class="bi bi-pencil me-1"></i>Editar
            </a>
            <form action="#" method="post" data-confirm="Excluir esta sugestão? Esta operação não pode ser desfeita.">
              <button type="submit" class="btn btn-sm btn-outline-danger rounded-pill">
                <i class="bi bi-trash me-1"></i>Excluir
              </button>
            </form>
          </div>
        </td>
      </tr>
      <tr>
        <td>
          <a class="fw-semibold text-decoration-none text-dark" href="../details.html">Clube de estudos semanal</a>
          <div class="small text-muted">ID #2</div>
        </td>
        <td><span class="category-tag">Ensino</span></td>
        <td>Desenvolvimento de Sistemas</td>
        <td>Maria Silva</td>
        <td class="text-end">
          <div class="d-flex gap-2 justify-content-end">
            <a class="btn btn-sm btn-outline-dark rounded-pill" href="edit.html">
              <i class="bi bi-pencil me-1"></i>Editar
            </a>
            <form action="#" method="post" data-confirm="Excluir esta sugestão? Esta operação não pode ser desfeita.">
              <button type="submit" class="btn btn-sm btn-outline-danger rounded-pill">
                <i class="bi bi-trash me-1"></i>Excluir
              </button>
            </form>
          </div>
        </td>
      </tr>
      </tbody>
    </table>
  </div>
</main>

<footer class="py-5 mt-5 footer">
  <div class="container small text-muted">Projeto educacional · Spring Boot · Thymeleaf</div>
</footer>

<script src="/webjars/bootstrap/5.3.3/js/bootstrap.bundle.min.js"></script>
<script src="/prototipo/js/ui.js"></script>
</body>
</html>
```

## Passo 8 — `static/prototipo/admin/edit.html`

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Editar sugestão · Caixa de Sugestões</title>
  <link href="/webjars/bootstrap/5.3.3/css/bootstrap.min.css" rel="stylesheet">
  <link href="/webjars/bootstrap-icons/1.11.3/font/bootstrap-icons.css" rel="stylesheet">
  <link href="/css/site.css" rel="stylesheet">
</head>
<body>
<nav class="navbar navbar-expand-lg navbar-light sticky-top site-nav">
  <div class="container py-2">
    <a class="navbar-brand d-flex align-items-center gap-2" href="../index.html">
      <span class="brand-mark"><i class="bi bi-chat-square-heart-fill"></i></span>
      <span>caixa<span class="text-brand">ideias</span></span>
    </a>
  </div>
</nav>

<main class="container page-narrow py-5">
  <a class="back-link" href="dashboard.html"><i class="bi bi-arrow-left me-2"></i>Voltar ao painel</a>
  <div class="mt-4 mb-4">
    <span class="eyebrow">ADMINISTRAÇÃO · EDIÇÃO</span>
    <h1 class="page-title mt-2">Refinar uma <span class="serif-accent">ideia.</span></h1>
    <p class="text-muted">Atualize as informações da sugestão selecionada.</p>
  </div>

  <form class="paper-card p-4 p-md-5" action="#" method="post">
    <div class="row g-4">
      <div class="col-12">
        <label for="title" class="form-label fw-semibold">Título da sugestão</label>
        <input id="title" name="title" class="form-control form-control-lg"
               value="Mais tomadas na biblioteca" maxlength="255" required>
      </div>
      <div class="col-12">
        <label for="content" class="form-label fw-semibold">Conte sua ideia</label>
        <textarea id="content" name="content" class="form-control" rows="5" required>Instalar tomadas nas mesas de estudo.</textarea>
      </div>
      <div class="col-md-6">
        <label for="courseId" class="form-label fw-semibold">Curso</label>
        <select id="courseId" name="courseId" class="form-select form-select-lg" required>
          <option value="1" selected>Desenvolvimento de Sistemas</option>
          <option value="2">Eletromecânica</option>
        </select>
      </div>
      <div class="col-md-6">
        <label for="categoryId" class="form-label fw-semibold">Categoria</label>
        <select id="categoryId" name="categoryId" class="form-select form-select-lg" required>
          <option value="1" selected>Infraestrutura</option>
          <option value="2">Ensino</option>
        </select>
      </div>
      <div class="col-12">
        <label for="author" class="form-label fw-semibold">Seu nome
          <span class="text-muted fw-normal">(opcional)</span></label>
        <input id="author" name="author" class="form-control form-control-lg" value="">
        <p class="form-text">Deixe em branco para publicar como Anônimo.</p>
      </div>
    </div>
    <div class="d-flex justify-content-between align-items-center flex-wrap gap-3 mt-4">
      <a class="text-muted text-decoration-none" href="dashboard.html">Cancelar</a>
      <button type="submit" class="btn btn-brand btn-lg rounded-pill px-4">
        Salvar alterações <i class="bi bi-check-lg ms-2"></i>
      </button>
    </div>
  </form>
</main>

<footer class="py-5 mt-5 footer">
  <div class="container small text-muted">Projeto educacional · Spring Boot · Thymeleaf</div>
</footer>

<script src="/webjars/bootstrap/5.3.3/js/bootstrap.bundle.min.js"></script>
<script src="/prototipo/js/ui.js"></script>
</body>
</html>
```

## Mapa das telas

| Protótipo estático | Tema | Template na Parte 2 | Rota na Parte 3 |
|---|---|---|---|
| `prototipo/index.html` | Mural com filtros e cartões | `templates/home.html` | `GET /`, `GET /suggestions` |
| `prototipo/new.html` | Formulário público | `templates/new.html` | `GET /suggestions/new`, `POST /suggestions` |
| `prototipo/details.html` | Detalhes da sugestão | `templates/details.html` | `GET /suggestions/{id}` |
| `prototipo/login.html` | Login administrativo | `templates/login.html` | `GET /login`, `POST /login` |
| `prototipo/admin/dashboard.html` | Painel e tabela | `templates/admin/dashboard.html` | `GET /admin/suggestions` |
| `prototipo/admin/edit.html` | Edição administrativa | `templates/admin/edit.html` | `GET/POST /admin/suggestions/{id}/edit` |

## Checkpoint da Parte 1

1. Com a aplicação parada, confira que não há arquivos Java novos.
2. Execute `./gradlew bootRun` (Windows: `.\gradlew.bat bootRun`).
3. Acesse `http://localhost:8081/prototipo/index.html`.
4. Navegue pelas seis telas pelos links do navbar, dos cartões e do painel.
5. No mural, pesquise por `biblioteca`, filtre por curso `2` e confirme que só o cartão de Eletromecânica aparece e que o contador muda.
6. No painel, clique em **Excluir** e cancele o diálogo; confirme que a linha permanece.
7. Reduza a largura da janela e abra o menu colapsável.
8. No código-fonte das páginas, confirme: nenhum `th:` e nenhum `fetch(`.

Se algum ícone `bi-...` não aparecer, confira a URL do WebJar de Bootstrap Icons. Se o CSS não carregar, confira se `site.css` está em `static/css/` e se o servidor foi reiniciado depois de criar o arquivo.

Quando tudo estiver navegável, continue em [Parte 2 — Thymeleaf](parte-2-thymeleaf.md).
