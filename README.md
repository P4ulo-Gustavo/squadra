=# Routes-PHP 🚦

Gerenciador de rotas dinâmico construído com **PHP puro, sem frameworks**. O projeto implementa do zero os pilares de um mini-framework MVC: roteamento, controllers, sistema de templates e separação clara entre camadas — com o objetivo de entender como frameworks como o Laravel funcionam por baixo dos panos.

---

## Sumário

- [Visão geral](#visão-geral)
- [Como funciona](#como-funciona)
- [Estrutura do projeto](#estrutura-do-projeto)
- [Classes e responsabilidades](#classes-e-responsabilidades)
- [Sistema de templates](#sistema-de-templates)
- [Configuração e instalação](#configuração-e-instalação)
- [Como adicionar uma nova rota](#como-adicionar-uma-nova-rota)
- [Requisitos](#requisitos)

---

## Visão geral

A maioria dos projetos PHP iniciantes cria um arquivo `.php` para cada página — `home.php`, `login.php`, `contato.php`. Essa abordagem escala mal, é difícil de manter e expõe arquivos diretamente na URL.

Este projeto resolve isso com um **front controller**: toda requisição passa por um único ponto de entrada (`index.php`), que delega a execução para o controller correto com base na URL. O resultado é um sistema organizado, seguro e fácil de expandir.

---

## Como funciona

```
Navegador
    │
    │  GET /login
    ▼
 .htaccess  ──────────────────────────────────────────────────────────
    │  Redireciona toda requisição que não seja                       │
    │  arquivo ou diretório real para index.php                       │
    ▼                                                                 │
 index.php                                                            │
    │  1. Instancia Request (captura método, URI, headers, body)      │
    │  2. Instancia Router (recebe o Request)                         │
    │  3. Carrega router/page.php (registra as rotas)  ◄──────────────┘
    │  4. Chama $router->dispatch()
    ▼
 Router::dispatch()
    │  Busca a rota no array interno
    │  Se não achar → retorna Response 404
    │  Se achar     → instancia o Controller e chama o método
    ▼
 Controller (ex: LoginController)
    │  Usa View::render() para montar o HTML
    │  Retorna new Response(200, $html)
    ▼
 Response::send()
    │  Envia o status HTTP
    │  Envia os headers
    │  Envia o corpo (HTML, JSON, redirect...)
    ▼
Navegador recebe a resposta
```

---

## Estrutura do projeto

```
Routes-PHP/
├── App/
│   ├── Controller/
│   │   ├── Http/
│   │   │   ├── Request.php          # Captura e normaliza os dados da requisição
│   │   │   ├── Response.php         # Monta e envia a resposta HTTP
│   │   │   └── Router.php           # Registra e despacha as rotas
│   │   └── View/
│   │       ├── HomeController.php   # Controller da página inicial
│   │       ├── LoginController.php  # Controller da página de login
│   │       └── SiginController.php  # Controller da página de cadastro
│   ├── Utils/
│   │   └── View.php                 # Motor de templates com {{variáveis}}
│   ├── public/
│   │   ├── index.php                # Front controller — único ponto de entrada
│   │   └── .htaccess                # Redireciona tudo para index.php
│   ├── resources/
│   │   ├── css/                     # Estilos por página
│   │   └── js/                      # Scripts por página
│   ├── router/
│   │   └── page.php                 # Definição de todas as rotas da aplicação
│   └── view/
│       ├── header.html              # Template do cabeçalho
│       ├── footer.html              # Template do rodapé
│       ├── home.html                # View da página inicial
│       ├── login.html               # View da página de login
│       └── sigin.html               # View da página de cadastro
├── vendor/                          # Dependências do Composer (gerado automaticamente)
└── composer.json
```

---

## Classes e responsabilidades

### `Request` — `App/Controller/Http/Request.php`

Responsável por capturar e organizar tudo que chegou na requisição HTTP. Ao ser instanciada, já lê automaticamente:

| Dado | Origem |
|---|---|
| Método HTTP | `$_SERVER['REQUEST_METHOD']` |
| URI | `$_SERVER['REQUEST_URI']` |
| Query params | `parse_url` + `parse_str` |
| Headers | `getallheaders()` |
| Body POST | `$_POST` ou JSON via `php://input` |

Todos os dados são expostos via getters (`getMethod()`, `getUri()`, `getPostVars()`...).

---

### `Router` — `App/Controller/Http/Router.php`

Gerencia o registro e o despacho das rotas. Internamente mantém um array no formato:

```php
$routes['GET']['/login'] = [LoginController::class, 'index'];
```

**Registro de rotas:**
```php
$router->get('/login', [LoginController::class, 'index']);
$router->post('/login', [AuthController::class, 'handle']);
```

**Despacho:** ao chamar `dispatch()`, o Router pega o método e a URI do `Request`, localiza a ação correspondente no array e instancia o controller para executá-la. Se a rota não existir, retorna automaticamente um `Response` com status 404.

---

### `Response` — `App/Controller/Http/Response.php`

Encapsula a resposta HTTP que será enviada ao navegador. O método `send()` dispara o status code, os headers e o corpo da resposta.

Possui métodos estáticos de atalho para os casos mais comuns:

```php
// Resposta JSON (API)
return Response::json(['status' => 'ok']);

// Redirecionamento
return Response::redirect('/login');
```

---

### `View` — `App/Utils/View.php`

Motor de templates minimalista. Lê o arquivo `.html` correspondente da pasta `view/` e substitui as marcações `{{chave}}` pelos valores passados pelo controller.

```php
// No controller
View::render('home', [
    'header' => View::render('header'),
    'footer' => View::render('footer'),
]);
```

```html
<!-- Em home.html -->
{{header}}
<main>Conteúdo da página</main>
{{footer}}
```

Isso permite reutilizar componentes como header e footer em qualquer página, mantendo as views modulares e limpas.

---

### Controllers — `App/Controller/View/`

Cada controller representa uma página da aplicação. Recebem o `Request`, montam o HTML via `View::render()` e retornam um `Response`.

```php
class LoginController
{
    public function index(Request $request): Response
    {
        return new Response(200, View::render('login'));
    }
}
```

---

### `router/page.php`

Arquivo de configuração de rotas. É o único arquivo a ser editado quando uma nova página é adicionada ao projeto:

```php
$router->get('/',       [HomeController::class,  'index']);
$router->get('/login',  [LoginController::class, 'index']);
$router->get('/sigin',  [SiginController::class, 'index']);
```

---

## Sistema de templates

O projeto utiliza um sistema de templates próprio baseado em substituição de marcações `{{chave}}`. Os arquivos de view ficam em `App/view/` e são arquivos `.html` comuns — sem sintaxe especial de PHP, sem `<?php echo ?>` espalhados no HTML.

Isso promove uma separação real entre lógica e apresentação, e facilita a colaboração com designers que não precisam lidar com código PHP.

---

## Configuração e instalação

**1. Clone o repositório**
```bash
git clone https://github.com/seu-usuario/routes-php.git
cd routes-php
```

**2. Instale as dependências**
```bash
composer install
```

**3. Configure o Virtual Host no Apache**

Aponte o `DocumentRoot` para a pasta `App/public/`:

```apache
<VirtualHost *:80>
    ServerName routes.local
    DocumentRoot /var/www/routes-php/App/public

    <Directory /var/www/routes-php/App/public>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

**4. Ative o `mod_rewrite`**
```bash
sudo a2enmod rewrite
sudo systemctl restart apache2
```

**5. (Opcional) Exponha para outras máquinas com ngrok**
```bash
ngrok http 80
```

---

## Como adicionar uma nova rota

**1. Crie o controller** em `App/Controller/View/`:

```php
<?php

namespace App\Controller\View;

use App\Controller\Http\Request;
use App\Controller\Http\Response;
use App\Utils\View;

class ContatoController
{
    public function index(Request $request): Response
    {
        return new Response(200, View::render('contato'));
    }
}
```

**2. Crie a view** em `App/view/contato.html`:

```html
{{header}}
<main>
    <h1>Entre em contato</h1>
</main>
{{footer}}
```

**3. Registre a rota** em `App/router/page.php`:

```php
$router->get('/contato', [ContatoController::class, 'index']);
```

Pronto. Nenhuma outra parte do sistema precisa ser alterada.

---

## Requisitos

- PHP 8.0 ou superior
- Apache com `mod_rewrite` ativo
- Composer

---

> Projeto desenvolvido por **Gustavo** como estudo de arquitetura MVC e roteamento em PHP puro.
# squadra
