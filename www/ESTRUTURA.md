# A ÁGUIA CHEGOU — Documentação da Arquitetura do App (Cordova)

Documento de referência da estrutura do código em `www/`, destinado a desenvolvedores
e LLMs que precisem entender ou modificar o projeto. Descreve a organização MVC,
o padrão de chamadas AJAX, o sistema de views e o fluxo de estado.

> Observação: detalhes de autenticação/login estão propositalmente fora do escopo;
> o foco é a organização do código.

---

## 1. Visão geral

- App híbrido **Apache Cordova** (Android/iOS), escrito em **HTML + JavaScript puro (ES6 classes)**.
- **Sem framework de front-end** (sem Angular/React/Vue). A "SPA" é construída manualmente:
  um único `index.html` serve de shell e as telas são injetadas via jQuery em um container.
- Backend remoto via **API REST própria em PHP** (`https://aaguiachegou.com.br/apiservicekeys/`),
  com um domínio separado para pagamentos (`/pay/`).
- Padrão arquitetural: **MVC ad hoc em 5 arquivos JavaScript globais**, todos carregados via
  `<script>` no `index.html`, sem módulos/bundler. Tudo vive no escopo global.

### Domínio do negócio

Marketplace de serviços com **dois perfis de usuário** (selecionáveis na tela inicial):

- **Cliente**: publica solicitações de orçamento ("anúncios") por categoria de serviço.
- **Profissional**: visualiza orçamentos disponíveis e **desbloqueia** os anúncios gastando
  **MOEDAS** (moeda virtual comprada por boleto ou cartão de crédito).

Funcionalidades anexas: Cursos & Treinamentos (com aulas em vídeo e testes), "Indique e ganhe",
alertas, edição de perfil, configurações e exclusão de conta.

---

## 2. Mapa de arquivos (`www/`)

```
www/
├── index.html                 # Shell único do app (header, footer, modais, scripts)
├── app.js                     # Bootstrap: instancia o objeto global `app` e inicia
├── controllers/
│   └── Controllers.js         # classes `App` (controller principal) e `Sessao`
├── models/
│   └── Models.js              # classe `Models`: todas as chamadas AJAX à API
├── views/
│   └── Views.js               # classe `Views`: geração de HTML das telas
├── helpers/
│   └── Helpers.js             # classe `Helpers`: máscaras e conversões
├── commom/
│   └── commom.js              # Funções globais utilitárias (fora de classes)
└── assets/                    # CSS (Bootstrap, style, sweetalert2...), imagens, JS de libs
```

### Ordem de carregamento dos scripts (final do `index.html`)

A ordem importa, pois cada arquivo usa símbolos definidos no anterior:

1. Libs: jQuery 3.4, OwlCarousel (CDN), tether, bootstrap, sweetalert2, wow, touchSwipe, jquery.form, inputmask
2. `commom/commom.js` — funções globais (aviso, confirmacao, etc.)
3. `controllers/Controllers.js` — classes `App` e `Sessao`
4. `models/Models.js` — classe `Models`
5. `helpers/Helpers.js` — classe `Helpers`
6. `views/Views.js` — classe `Views`
7. `app.js` — instancia `var app = new App(...)` e chama `app.initApp()`
8. `cordova.js` — bridge nativa do Cordova

---

## 3. Bootstrap (`app.js`)

Arquivo de 9 linhas. Instancia o **objeto global `app`**, que é o ponto de entrada de toda a
aplicação e é referenciado diretamente nos `onclick` do HTML:

```js
var app = new App(appId, appName, appVersion, appOs, ambiente, token, tokenSms);
app.initApp();
```

Parâmetros do construtor de `App` (`controllers/Controllers.js`):

- `appId`, `appName`, `appVersion`, `appOs` — metadados do app.
- `ambiente` — `"HOMOLOGACAO"` ou `"PRODUCAO"` (define as URLs base; hoje ambas apontam
  para os mesmos endpoints de produção).
- `token` — token estático enviado em **todas** as chamadas à API (`&token=...`).
- `tokenSms` — token do serviço de SMS (verificação de código no login).

O construtor de `App` também compõe as outras camadas:

```js
this.views   = new Views();
this.sessao  = new Sessao();
this.models  = new Models();
this.helpers = new Helpers();
```

E define as URLs base:

| Propriedade | Uso |
|---|---|
| `app.urlApi` | API principal (`.../apiservicekeys/`) |
| `app.urlApiPagto` | API de pagamentos (`.../pay/`) |
| `app.urlDom` | Domínio do app |
| `app.urlCdn` | CDN de arquivos/imagens |

---

## 4. Arquitetura MVC

Não há rotas nem framework. O "MVC" é uma convenção de separação em 3 classes +
funções globais, orquestrada pelo objeto `app`:

```
HTML (onclick="app.metodo()")  →  App (controller)  →  Views (renderiza HTML)
                                     │              →  Models (AJAX → API)
                                     ↓
                              localStorage (estado)
```

### 4.1 Padrão de uma "tela"

Uma tela é sempre montada por **duas chamadas em sequência** no controller:

```js
minhasSolicitacoes(){
    this.views.minhasSolicitacoes();   // 1) injeta o HTML esqueleto da tela
    this.models.minhasSolicitacoes();  // 2) AJAX busca os dados e preenche o HTML
}
```

Ou seja:

- **Views** — métodos que fazem `this._content.html(`...template literal...`)`.
  O HTML contém `onclick="app.outroMetodo()"` inline para navegação/ações.
- **Models** — métodos que fazem a chamada AJAX e, **no callback de sucesso**,
  manipulam o DOM diretamente via jQuery (`$("#lista").html(...)`) e/ou gravam no
  `localStorage`. **Não há camada de dados desacoplada**: o model conhece os seletores
  do DOM da view que o chamou.

### 4.2 Container principal

`index.html` tem apenas um container de conteúdo:

```html
<section id="content"> ... </section>
```

A classe `Views` guarda referências no construtor:

```js
this._content  = $("section#content");  // onde toda tela é injetada
this.header    = $("header");
this.footer    = $("footer");           // navegação inferior (3 menus)
this._menu1..3 // itens do footer
```

`index.html` também contém **fixos** (não são views): header com logo e botão de menu,
footer de navegação, os dois menus laterais (`.menu-adicional-cliente` e
`.menu-adicional-profissional`) e os modais globais de aviso/confirmação.

---

## 5. Padrão das chamadas AJAX (Models.js)

Os models usam **XMLHttpRequest vanilla** (não `$.ajax`), sempre com o mesmo molde:

```js
nomeDoMetodo(){
    let xhr = new XMLHttpRequest();

    var idUsuario = localStorage.getItem("idUsuario");

    xhr.open('POST', app.urlApi + 'endpoint-da-api', true);
    xhr.setRequestHeader('Content-type', 'application/x-www-form-urlencoded');

    var params = 'idUsuario=' + idUsuario +
                 '&outroParam=' + valor +
                 '&token=' + app.token;        // token SEMPRE no final

    xhr.onreadystatechange = () => {
        if (xhr.readyState == 4) {
            if (xhr.status == 200) {
                var dados = JSON.parse(xhr.responseText);
                // 1. localStorage.setItem(...) com o que precisar persistir
                // 2. $("#seletor").html(`...html com os dados...`)
                // 3. em erro de negócio: aviso("Oops!...", "...")
            } else {
                aviso("Oops! Algo deu errado.", "Nossos servidores estão passando por dificuldades...");
            }
        }
    };

    xhr.send(params);
}
```

Convenções:

- **Método HTTP**: sempre `POST`, body `application/x-www-form-urlencoded`, montado
  concatenando strings (sem `FormData` na maioria dos casos).
- **Resposta**: JSON. Sucesso de negócio costuma ser checado via `dados.sucesso == 200`.
- **Autenticação**: token estático `app.token` + `idUsuario` do `localStorage`.
- **Erro**: o callback de falha chama `aviso(titulo, mensagem)` (modal global de `commom.js`).
- **Feedback de UI**: o controller troca o texto do botão antes de chamar o model
  (ex.: `$("#btnEnviarSolicitacao").html("enviando... aguarde")`).

Exceção: `commom.js` tem um `$.ajax` para upload de selfie (`upload-selfie-camera.php`)
e um helper `ajaxSubmit(form)` genérico para formulários.

### Inventário dos models (Models.js)

| Método | Endpoint / finalidade |
|---|---|
| `testeApi()` | Health-check da API no boot |
| `procLogin()` / `procLoginSms()` / `verificarCodigoSms()` | Fluxo de login (email+senha e SMS) |
| `procCadastro()` | Cadastro de usuário |
| `procResetSenha()` | "Esqueci minha senha" |
| `liberacoes()` | Retorna **Promise** booleana usada como trava/guarda de fluxo |
| `editarPerfil()` / `procEditarPerfil()` | Carrega e salva dados do perfil |
| `categoriasAtendimento()` | `categorias-atendimento` — árvore de categorias; salva em `localStorage["categoiasAtendimento"]` |
| `enviarAtendimento()` | `enviar-atendimento` — cliente publica solicitação de orçamento |
| `minhasSolicitacoes()` | Lista solicitações do cliente |
| `removerSolicitacaoOrcamento(id)` / `fecharSolicitacaoOrcamento(id)` | Cancela / encerra anúncio |
| `orcamentosDisponiveis()` | Lista orçamentos para o profissional (home profissional) |
| `orcamentosDisponiveisDesbloqueados()` | "Meus Serviços" — orçamentos já desbloqueados |
| `carregarDetalheAtendimento(idAnuncio, acao)` | Detalhe do anúncio/orçamento |
| `pacoteChaves()` / `selecaoPacoteDeChaves(pacote)` | Pacotes de MOEDAS e seleção do pacote |
| `payBoleto()` / `payCartaoDeCredito()` | Pagamento (API `urlApiPagto`); cartão verifica `status=="CONFIRMED"` |
| `atualizarSaldoCompra()` / `salvarDadosCompraUsuario(customer, id)` | Pós-pagamento: saldo e dados do comprador |
| `cursos()` / `detalheCurso(id)` | Listagem e detalhe de cursos |
| `iniciarCurso()` / `carregarProximaAula()` / `salvarInicioCurso(id)` / `atualizarHistoricoAluno()` | Player de curso: aulas, progresso, testes |
| `salvarMinhasCategorias()` | Salva as 2 categorias de atuação do profissional |
| `duvidasESuporte()` | Conteúdo de suporte/FAQ |

---

## 6. Views (Views.js)

Cada método renderiza uma tela inteira via template literal injetado em `this._content`.
Pontos importantes:

- **Navegação por `onclick` inline**: o HTML gerado chama diretamente métodos do
  controller global, ex.: `onclick="app.novoAtendimentoPasso2(${n.id},'${n.titulo}')"`.
- **Loops no template**: listas são montadas com `dados.map(n => \`<li>...\`).join('')`
  (às vezes no model, no callback do AJAX).
- **Atributo `view-name`**: as divs raiz das telas carregam `view-name="..."` para
  identificação/depuração.
- **Menus do footer**: controlados por `ativarMenuUm/Dois/Tres()` e
  `desativarTodosMenus()` (telas de login/cadastro escondem o footer).
- **Animações**: `animarTransicao()` inicializa o WOW.js após injetar o HTML.

### Inventário das views

- **Onboarding/perfil**: `viewPrincipal` (seleção cliente/profissional), `selecionarMinhasCategorias`
- **Cliente**: `viewPrincipalCliente`, `novoAtendimento`, `minhasSolicitacoes`, `listagemNovaBlocada`
- **Profissional**: `viewPrincipalProfissional` (home com orçamentos), `servicosDesbloqueadosProfissional`,
  `alertasProfissionais`, `viewDetalheAnuncio(idAnuncio, acao)`
  (existem versões legadas: `viewPrincipalProfissionalAntigo`, `viewPrincipalProfissionalNovaTeste`,
  `viewListagemItensNovaTeste` — código histórico mantido)
- **Conta**: `editarPerfil`, `configuracoes`, `apagarConta`, `duvidasESuporte`
- **Pagamentos**: `viewComprarChaves`, `paginaDeCmopra` (sic), `processandoPagamento`,
  `processandoPagamentoCartao`, `dadosBoleto`, `dadosCartao`, `dadosCartaoPendente`
- **Cursos**: `cursos`, `detalheCurso`, `iniciarCurso`, `nextAula`, `detalheTeste`, `corrigirTeste`
- **Marketing**: `indiqueEGanhe`
- **Autenticação**: `viewLogin`, `viewCodigoSms`, `viewLoginEmailSenha`, `viewCadastro`,
  `viewEsqueciMinhaSenha`, `viewUploadFoto`
- **Genéricas/legado**: `view2`, `view3`

---

## 7. Controllers (Controllers.js)

Contém duas classes:

### `class App` — o controller/FAÇADE da aplicação

Responsabilidades:

- **Orquestrar view + model** (padrão da seção 4.1).
- **Fluxos multi-passo**: ex. criação de atendimento em 3 passos
  (`novoAtendimentoPasso2` → `novoAtendimentoPasso3` → `enviarAtendimento`), com lógica
  de categorias-pai/filhas feita no controller a partir do JSON cacheado em
  `localStorage["categoiasAtendimento"]`.
- **Regras de UI**: confirmações (`confirmacao(...)`), avisos (`aviso(...)`),
  travas de negócio (ex.: `desbloqAnuncio` verifica saldo de MOEDAS e categorias do
  profissional antes de permitir o desbloqueio).
- **Menus laterais**: `abrirFecharMenuCliente/Profissional` (toggle de classe CSS `aberto`).
- **Filtros client-side**: `filtrotabela()`, `filtrotabelaCursos()` (busca em `<li>`).
- **Sessão**: `logoff()` limpa o `localStorage` e volta ao login.

### `class Sessao`

Espelha o estado de login a partir do `localStorage` (`bdLogado`, `idUsuario`,
`emailUsuario`, `dadosUsuario`). `logarUsuario()` grava a sessão e redireciona;
`verificarLogado()` decide a tela inicial no boot.

---

## 8. Estado global — `localStorage`

**Não há store/state management**: todo estado compartilhado vai para o `localStorage`
e é lido sob demanda nos controllers/models. Chaves relevantes:

| Chave | Conteúdo |
|---|---|
| `bdLogado`, `idUsuario`, `emailUsuario`, `dadosUsuario`, `dadosCompletosUsuario` | Sessão |
| `selecaoPerfil` | `"cliente"` ou `"profissional"` |
| `categoiasAtendimento` (sic) | JSON da árvore de categorias |
| `categoria1`, `categoria2` | Categorias de atuação do profissional |
| `saldoPrestadorServico` | Saldo de MOEDAS |
| `posLoginAtivacao`, `posLoginIdCategoria`, `posLoginNomeCategoria` | Retomar fluxo de orçamento após login/cadastro |
| `idCategoriaHistorico`, `nomeCategoria`, `anuncioHeranca`, `aulaHasTeste` | Contexto de navegação |
| `campoParaPreenchimento` | Alvo do input flutuante |

---

## 9. Funções globais (`commom/commom.js`)

Funções soltas (não pertencem a classe), usadas por todas as camadas:

- **Modais de feedback**:
  - `aviso(titulo, mensagem)` / `fecharAviso()` — modal de aviso (`#swipeAviso`), fecha por swipe-down (touchSwipe).
  - `confirmacao(titulo, mensagem, funcaoConfirmacao, textoBotao)` / `fecharConfirmacao()` —
    modal de confirmação. **Atenção**: `funcaoConfirmacao` é uma *string* de JS executada
    via `onclick`, ex.: `confirmacao("...", "...", "app.logoff()", "Sair")`.
- **Input flutuante** (acessibilidade/mobile): `ativarFormularioFlutuante(campo, label)` e
  `validarFormularioFlutuante(event)` — campo sobreposto que copia o valor para o input alvo.
- `abrirUrl(url)` — abre URL externa via `cordova.InAppBrowser`.
- **Upload**: `uploadLocal()` + `processJson()` via `jquery.form` (`ajaxForm`);
  `initCameraSelfie()` / `uploadMyImageSelfie()` via plugin Cordova Camera
  (`navigator.camera.getPicture`, base64 → POST para `upload-selfie-camera.php`).
- `copiarCodigoPix()` — copia código PIX para a área de transferência.
- `filtrarCategorias()` — toggle que mostra/esconde cards `.caixa-destaque-servicos`
  conforme `categoria1`/`categoria2` do profissional.
- `ajaxSubmit(form)` — submit genérico de formulário via `$.ajax`.

## 10. Helpers (`helpers/Helpers.js`)

Classe mínima:

- `Helpers.converterData(date)` — estático, `AAAA-MM-DD` → `DD/MM/AAAA`.
- `carregarMascaras()` — aplica máscaras inputmask (telefone, código SMS, CPF,
  cartão de crédito) nos campos de formulário.

---

## 11. Dependências de front-end

Carregadas localmente em `assets/` (salvo CDNs indicados):

- **jQuery 3.4** — toda manipulação de DOM e parte dos AJAX.
- **Bootstrap 3/4 (CSS+JS)** + bootstrap-dropdownhover — layout e componentes.
- **SweetAlert2** (`swal`) — alerts pontuais (o modal principal é o caseiro `aviso/confirmacao`).
- **WOW.js + animate.css** — animações de entrada das telas (`animarTransicao`).
- **OwlCarousel2 (CDN)** — carrosséis.
- **jquery.touchSwipe** — gestos (fechar modais por swipe).
- **jquery.form** — upload de arquivos (`ajaxForm`, `formSerialize`).
- **inputmask** — máscaras de formulário.
- **Cordova plugins**: `cordova.js`, InAppBrowser, Camera.

---

## 12. Convenções e cuidados ao modificar

1. **Tudo é global**: `app`, `aviso`, `confirmacao`, etc. Não há imports; ao criar
   arquivo novo, registre o `<script>` no `index.html` na ordem correta.
2. **Uma tela = par view+model**: ao adicionar funcionalidade, crie o método em `Views`
   (HTML esqueleto) e o método em `Models` (AJAX + preenchimento), e um método em `App`
   que chame os dois.
3. **Strings de código em atributos**: `onclick="app.x()"` e `confirmacao(..., "app.y()", ...)`
   executam strings — cuidado com escapes de aspas em template literals e com parâmetros
   string (`'${valor}'`).
4. **Sem trailing de rotas/histórico**: "voltar" é re-renderizar a view anterior chamando
   o método correspondente.
5. **Typos históricos preservados** (não renomear sem grep completo):
   `categoiasAtendimento`, `paginaDeCmopra`, `commom/` (em vez de `common`),
   `viewPrincipalProfissionalAntigo/NovaTeste`.
6. **Ambiente** se troca editando o parâmetro `"HOMOLOGACAO"/"PRODUCAO"` em `app.js`.
