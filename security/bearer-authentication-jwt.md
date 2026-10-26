# Autenticação Bearer com JWT (JSON Web Token)

> **Categoria:** Segurança de aplicações e APIs
> **Tecnologias:** HTTP, JWT, OAuth 2.0, OpenID Connect, ASP.NET Core e React
> **Idioma:** Português (Brasil)
> **Objetivo:** Documentar os conceitos, a implementação, o consumo e as boas práticas de segurança relacionadas à autenticação Bearer com tokens JWT.

---

## Sumário

1. [Introdução](#1-introdução)
2. [O que é autenticação Bearer?](#2-o-que-é-autenticação-bearer)
3. [O que é JWT?](#3-o-que-é-jwt)
4. [Diferença entre Bearer, JWT, OAuth 2.0 e OpenID Connect](#4-diferença-entre-bearer-jwt-oauth-20-e-openid-connect)
5. [Como funciona o fluxo de autenticação](#5-como-funciona-o-fluxo-de-autenticação)
6. [Estrutura de um JWT](#6-estrutura-de-um-jwt)
7. [Enviando um Bearer Token via HTTP](#7-enviando-um-bearer-token-via-http)
8. [Implementação no ASP.NET Core](#8-implementação-no-aspnet-core)
9. [Protegendo endpoints](#9-protegendo-endpoints)
10. [Autorização por roles e políticas](#10-autorização-por-roles-e-políticas)
11. [Consumo da API com React e TypeScript](#11-consumo-da-api-com-react-e-typescript)
12. [Autenticação no Swagger](#12-autenticação-no-swagger)
13. [Códigos de resposta HTTP](#13-códigos-de-resposta-http)
14. [Armazenamento e gerenciamento de tokens](#14-armazenamento-e-gerenciamento-de-tokens)
15. [Boas práticas de segurança](#15-boas-práticas-de-segurança)
16. [Erros comuns e soluções](#16-erros-comuns-e-soluções)
17. [Checklist para produção](#17-checklist-para-produção)
18. [Glossário](#18-glossário)
19. [Referências oficiais](#19-referências-oficiais)

---

# 1. Introdução

A autenticação é um dos componentes fundamentais da segurança de uma aplicação. Ela permite identificar o usuário ou sistema que está realizando uma requisição e estabelecer uma identidade que poderá ser utilizada para controlar o acesso aos recursos disponíveis.

Em aplicações modernas, especialmente aquelas que utilizam APIs REST, é comum empregar tokens de acesso para autenticar requisições HTTP.

Um dos mecanismos mais utilizados é a **autenticação Bearer**, frequentemente implementada com tokens no formato **JWT (JSON Web Token)**.

Essa abordagem permite que aplicações web, aplicações mobile, serviços de integração e outros clientes consumam APIs protegidas sem precisar enviar o nome de usuário e a senha em todas as requisições.

### Objetivos desta documentação

Ao concluir a leitura, será possível compreender:

- O funcionamento da autenticação Bearer.
- A estrutura e a finalidade de um JWT.
- A diferença entre autenticação e autorização.
- Como configurar a validação de JWT em uma API ASP.NET Core.
- Como proteger controllers e endpoints.
- Como enviar tokens por meio do cabeçalho HTTP `Authorization`.
- Como consumir endpoints protegidos utilizando React e TypeScript.
- Como configurar a autenticação no Swagger.
- Quais medidas devem ser adotadas para proteger tokens em produção.

> **Importante:** os exemplos de emissão manual de JWT apresentados nesta documentação são didáticos. Em aplicações de produção, prefira um provedor de identidade que implemente os padrões OAuth 2.0 e OpenID Connect, em vez de criar um sistema de emissão de tokens sem uma arquitetura de segurança adequada.

# 2. O que é autenticação Bearer?

Bearer é um esquema de autenticação HTTP utilizado para apresentar uma credencial, geralmente um token de acesso, ao servidor que protege determinado recurso.

A palavra *Bearer* pode ser entendida como "portador". Nesse modelo, quem apresenta um token válido pode obter acesso aos recursos associados a ele, desde que as regras de autorização também sejam atendidas.

O token normalmente é enviado no cabeçalho HTTP `Authorization`.

### Exemplo

```http
GET /api/usuarios HTTP/1.1
Host: localhost:7028
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

A estrutura do cabeçalho é:

```http
Authorization: Bearer <access_token>
```

Onde:

| Elemento         | Descrição                                     |
| ---------------- | --------------------------------------------- |
| `Authorization`  | Cabeçalho HTTP que transporta as credenciais. |
| `Bearer`         | Esquema de autenticação utilizado.            |
| `<access_token>` | Token de acesso apresentado pelo cliente.     |

A API recebe a requisição, valida o token de acordo com sua configuração e determina se o cliente está autenticado.

**Atenção:** um Bearer Token deve ser tratado como uma credencial secreta. Quem obtiver um token utilizável poderá tentar acessar os recursos associados a ele. Por isso, sua transmissão e seu armazenamento precisam ser protegidos.

Referência: [RFC 6750 — The OAuth 2.0 Authorization Framework: Bearer Token Usage](https://datatracker.ietf.org/doc/html/rfc6750).

## 2.1. Principais características

- A autenticação é realizada por meio de um token.
- O token pode ser enviado em requisições HTTP para diferentes endpoints autorizados.
- O cliente normalmente não precisa reenviar suas credenciais de login a cada requisição.
- O token pode possuir prazo de validade.
- O servidor precisa validar a credencial antes de permitir o acesso.
- A posse do token não garante acesso irrestrito: as políticas de autorização continuam sendo aplicadas.

## 2.2. Bearer não é um formato de token

É importante distinguir o esquema de autenticação do formato da credencial.

Bearer define como o token é apresentado na requisição. O token, por sua vez, pode utilizar diferentes formatos.

Por exemplo:

```http
Authorization: Bearer abc123xyz
```

O valor `abc123xyz` poderia ser um token opaco, cuja validade é consultada em um servidor, ou poderia ser outro tipo de credencial aceito pela API.

Quando esse valor é um JWT, temos uma implementação conhecida como **JWT Bearer Authentication**.

# 3. O que é JWT?

JWT significa **JSON Web Token**.

É um formato compacto, definido pela RFC 7519, utilizado para transportar declarações (*claims*) entre partes.

Essas declarações podem representar informações como:

- Identificador do usuário.
- Emissor do token.
- Destinatário previsto.
- Data de emissão.
- Data de expiração.
- Identificador do token.
- Funções ou permissões, quando apropriado.

Um JWT assinado permite verificar a integridade do conteúdo e a autenticidade da assinatura em relação à chave utilizada.

Entretanto, **um JWT assinado não é necessariamente criptografado**. Seu conteúdo pode ser decodificado por quem tiver acesso ao token.

Referência: [RFC 7519 — JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519).

## 3.1. Principais vantagens

**Ausência de sessão obrigatória no servidor**

Um JWT pode permitir que a API valide a identidade e as declarações do usuário sem consultar um registro de sessão em cada requisição.

Isso não significa que toda arquitetura JWT seja completamente *stateless*: mecanismos de revogação, sessões, permissões e outros controles podem exigir consultas adicionais.

**Integração entre aplicações**

Uma aplicação web, um aplicativo mobile ou outro serviço pode apresentar um token emitido por um provedor confiável para acessar uma API.

**Validação padronizada**

Bibliotecas de segurança implementam os mecanismos necessários para validar a assinatura, a validade temporal e outras propriedades do token.

**Transporte simples**

O token pode ser enviado em cabeçalhos HTTP convencionais.

## 3.2. Limitações

JWT não elimina os desafios de segurança de uma aplicação.

Entre suas limitações estão:

- Um token roubado pode ser utilizado até expirar ou ser invalidado por mecanismos apropriados.
- Um token autocontido pode continuar sendo aceito após a desativação de um usuário, caso a API não consulte mecanismos adicionais de revogação ou autorização.
- O tamanho do token pode aumentar conforme a quantidade de declarações.
- A assinatura não oculta os dados do payload.
- A validação incorreta do emissor, do destinatário ou da assinatura pode comprometer a segurança.

JWT deve ser empregado de acordo com as necessidades da arquitetura, e não apenas por ser uma tecnologia popular.

# 4. Diferença entre Bearer, JWT, OAuth 2.0 e OpenID Connect

Esses conceitos são relacionados, mas não representam a mesma coisa.

| Conceito              | Finalidade                                                                                                    |
| --------------------- | ------------------------------------------------------------------------------------------------------------- |
| Bearer                | Define o esquema de apresentação de uma credencial em uma requisição HTTP.                                    |
| JWT                   | Define um formato compacto para transportar declarações.                                                      |
| Access Token          | Representa a credencial utilizada para acessar recursos protegidos.                                           |
| OAuth 2.0             | Define um framework de autorização e obtenção de tokens de acesso.                                            |
| OpenID Connect (OIDC) | Adiciona uma camada de identidade e autenticação sobre OAuth 2.0.                                             |
| Refresh Token         | Permite solicitar novos tokens de acesso conforme as regras do provedor de identidade.                        |
| ID Token              | Contém informações de autenticação destinadas ao cliente OIDC; não deve ser usado como token de acesso à API. |

### Exemplo de relacionamento

Uma arquitetura pode utilizar OpenID Connect para autenticar o usuário e OAuth 2.0 para obter um token de acesso.

O provedor de identidade emite um access token no formato JWT.

A aplicação cliente envia esse token para a API utilizando o esquema Bearer:

```http
Authorization: Bearer <access_token>
```

A API valida o token e aplica suas políticas de autorização.

Nesse fluxo, cada tecnologia possui uma responsabilidade específica.

> **Recomendação:** para aplicações de produção, utilize fluxos padronizados de OAuth 2.0 e OpenID Connect. A validação do JWT na API não substitui um processo seguro de autenticação e emissão de tokens.

# 5. Como funciona o fluxo de autenticação

Considere uma arquitetura composta por:

- **Frontend:** React com TypeScript.
- **Backend:** API ASP.NET Core.
- **Provedor de identidade:** responsável por autenticar usuários e emitir tokens.
- **Recurso protegido:** endpoint da API que exige autenticação.

O fluxo pode ser representado da seguinte forma:

```text
┌─────────────────────┐
│     Usuário         │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Aplicação React     │
│ Formulário de login │
└──────────┬──────────┘
           │
           │ 1. Inicia o processo de autenticação
           ▼
┌─────────────────────┐
│ Provedor de         │
│ identidade          │
└──────────┬──────────┘
           │
           │ 2. Emite o access token
           ▼
┌─────────────────────┐
│ Aplicação cliente   │
└──────────┬──────────┘
           │
           │ 3. Requisição HTTP
           │ Authorization: Bearer <token>
           ▼
┌─────────────────────┐
│ API ASP.NET Core    │
│                     │
│ 4. Valida o token   │
│ 5. Avalia acesso    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Recurso protegido   │
│ Resposta HTTP       │
└─────────────────────┘
```

## 5.1. Etapas detalhadas

**1. Autenticação**

O usuário inicia o processo de login por meio de um fluxo apropriado, como o fluxo de código de autorização com PKCE para aplicações públicas.

**2. Emissão do token**

O provedor de identidade emite um token de acesso conforme as regras do fluxo utilizado.

**3. Requisição autenticada**

O cliente apresenta o token no cabeçalho `Authorization`.

**4. Validação do token**

A API verifica a assinatura, o emissor, o destinatário, a expiração e os demais requisitos configurados.

**5. Autorização**

A API verifica se o usuário autenticado possui a permissão necessária para acessar o recurso solicitado.

**6. Resposta**

Se a autenticação e a autorização forem bem-sucedidas, a API processa a requisição. Caso contrário, retorna uma resposta HTTP apropriada.

# 6. Estrutura de um JWT

Um JWT assinado no formato JWS normalmente possui três partes separadas por pontos:

```text
HEADER.PAYLOAD.SIGNATURE
```

Exemplo ilustrativo:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJzdWIiOiIxMjMiLCJyb2xlIjoiQWRtaW4ifQ
.
assinatura_exemplificativa
```

O exemplo é apenas ilustrativo e não representa um token válido.

## 6.1. Header

O cabeçalho informa metadados do token, como o algoritmo de assinatura.

Exemplo:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

| Propriedade | Descrição                              |
| ----------- | -------------------------------------- |
| `alg`       | Algoritmo utilizado para a assinatura. |
| `typ`       | Tipo do objeto, normalmente `JWT`.     |

O algoritmo indicado no token não deve ser aceito indiscriminadamente. A API precisa configurar os algoritmos e as chaves confiáveis de acordo com sua política de segurança.

## 6.2. Payload

O payload contém as declarações do token.

Exemplo:

```json
{
  "sub": "123",
  "iss": "https://identity.exemplo.com",
  "aud": "api-gestao-pedidos",
  "iat": 1792958400,
  "exp": 1792962000,
  "jti": "a7d54f8e-6f7b-4e19-90f7-123456789abc",
  "role": "Admin"
}
```

As datas são valores numéricos ilustrativos no formato Unix time.

| Claim  | Significado                                            |
| ------ | ------------------------------------------------------ |
| `sub`  | Identificador do sujeito, como o usuário autenticado.  |
| `iss`  | Emissor do token (*issuer*).                           |
| `aud`  | Destinatário previsto (*audience*).                    |
| `iat`  | Data de emissão (*issued at*).                         |
| `exp`  | Data de expiração (*expiration time*).                 |
| `jti`  | Identificador único do token.                          |
| `role` | Declaração de função, quando suportada pela aplicação. |

A interpretação de claims de autorização depende da configuração do provedor de identidade e da API.

**Nunca inclua senhas, chaves privadas ou outros segredos no payload.** Em um JWT apenas assinado, as informações podem ser lidas por quem obtiver o token.

## 6.3. Signature

A assinatura permite detectar alterações no conteúdo assinado e verificar se o token foi produzido por uma entidade que possui a chave apropriada.

No caso do HS256, a assinatura utiliza uma chave secreta compartilhada.

Em uma implementação com RS256, a assinatura é produzida com uma chave privada e pode ser validada com a chave pública correspondente.

Em arquiteturas com um provedor de identidade centralizado e várias APIs, a assinatura assimétrica costuma facilitar a distribuição das chaves públicas sem compartilhar a chave privada de emissão.

## 6.4. Token assinado não é token criptografado

A assinatura protege a integridade do conteúdo, mas não garante sua confidencialidade.

Portanto:

- Não coloque senhas no payload.
- Não coloque chaves privadas no payload.
- Não utilize claims como substituto de consultas a dados que precisam permanecer atualizados.
- Evite incluir informações pessoais desnecessárias.

Se houver necessidade de confidencialidade do conteúdo, será necessário adotar um mecanismo de criptografia apropriado. A criptografia, por si só, também não substitui a validação correta do token.

# 7. Enviando um Bearer Token via HTTP

A maneira convencional de enviar um token de acesso Bearer é utilizar o cabeçalho `Authorization`.

## 7.1. Exemplo com GET

```http
GET /api/usuarios HTTP/1.1
Host: localhost:7028
Authorization: Bearer <access_token>
Accept: application/json
```

## 7.2. Exemplo com POST

```http
POST /api/pedidos HTTP/1.1
Host: localhost:7028
Authorization: Bearer <access_token>
Content-Type: application/json
```

Corpo da requisição:

```json
{
  "clienteId": 10,
  "valorTotal": 250.90
}
```

## 7.3. Exemplo com cURL

```bash
curl -X GET "https://localhost:7028/api/usuarios" \
  -H "Authorization: Bearer SEU_ACCESS_TOKEN" \
  -H "Accept: application/json"
```

Substitua `SEU_ACCESS_TOKEN` por um token válido emitido para a API.

No Windows, o PowerShell pode utilizar uma sintaxe diferente para quebra de linhas. Para evitar problemas, execute o comando em uma única linha quando necessário.

## 7.4. Onde não enviar o token

Evite transmitir tokens de acesso em parâmetros da URL:

```http
GET /api/usuarios?access_token=SEU_TOKEN
```

URLs podem aparecer em logs, histórico do navegador, ferramentas de monitoramento e cabeçalhos de referência.

Prefira o cabeçalho `Authorization` e utilize HTTPS em todas as comunicações que transportam credenciais.

# 8. Implementação no ASP.NET Core

Esta seção apresenta uma implementação didática de validação de JWT em uma API ASP.NET Core.

A solução será organizada em:

1. Instalação do pacote.
2. Configuração da chave e das opções.
3. Registro dos serviços de autenticação.
4. Configuração do middleware.
5. Proteção de endpoints.

> **Escopo do exemplo:** esta implementação demonstra a validação de JWT assinado com HS256 usando uma chave compartilhada. Ela não constitui, isoladamente, um sistema de autenticação completo para produção. A emissão de tokens em produção deve seguir um fluxo padronizado de OAuth 2.0/OpenID Connect, preferencialmente por meio de um provedor de identidade confiável.

## 8.1. Instalar o pacote NuGet

No projeto da API, instale o pacote:

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

Utilize uma versão compatível com o framework de destino do projeto.

O pacote fornece o manipulador de autenticação Bearer utilizado para validar tokens recebidos pela API.

## 8.2. Configurar os parâmetros JWT

A API precisa conhecer os valores esperados para validar os tokens.

Exemplo de configuração:

```json
{
  "Jwt": {
    "Issuer": "https://identity.exemplo.com",
    "Audience": "api-gestao-pedidos"
  }
}
```

Nesse exemplo:

- `Issuer`: identifica o emissor confiável.
- `Audience`: identifica a API para a qual o token foi emitido.

A chave secreta não deve ser armazenada nesse arquivo quando ele fizer parte do repositório.

Para fins didáticos, os exemplos seguintes pressupõem uma chave de assinatura disponibilizada por configuração segura.

## 8.3. Armazenar a chave de assinatura

Durante o desenvolvimento local, o mecanismo de User Secrets pode ser utilizado para fornecer a chave sem incluí-la no código-fonte versionado.

Inicialize os User Secrets no projeto:

```bash
dotnet user-secrets init
```

Configure uma chave de teste com comprimento adequado para HS256:

```bash
dotnet user-secrets set "Jwt:Key" "CHAVE_DE_TESTE_LONGA_E_ALEATORIA_COM_NO_MINIMO_32_BYTES"
```

O valor acima é um exemplo didático, não uma chave recomendada para uso real. Em desenvolvimento, utilize uma chave aleatória própria, com pelo menos 256 bits de entropia para HS256.

Em produção, utilize um gerenciador de segredos ou outro mecanismo seguro de provisionamento de chaves. Não reutilize a chave de desenvolvimento.

## 8.4. Configurar a autenticação no Program.cs

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var jwtKey = builder.Configuration["Jwt:Key"]
    ?? throw new InvalidOperationException(
        "A chave de assinatura JWT não foi configurada."
    );

var jwtIssuer = builder.Configuration["Jwt:Issuer"]
    ?? throw new InvalidOperationException(
        "O emissor JWT não foi configurado."
    );

var jwtAudience = builder.Configuration["Jwt:Audience"]
    ?? throw new InvalidOperationException(
        "A audiência JWT não foi configurada."
    );

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters =
            new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidIssuer = jwtIssuer,

                ValidateAudience = true,
                ValidAudience = jwtAudience,

                ValidateLifetime = true,

                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes(jwtKey)
                ),

                ValidAlgorithms =
                [
                    SecurityAlgorithms.HmacSha256
                ],

                ClockSkew = TimeSpan.FromSeconds(30)
            };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

app.UseHttpsRedirection();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

### Entendendo as opções

| Propriedade                | Finalidade                                                  |
| -------------------------- | ----------------------------------------------------------- |
| `ValidateIssuer`           | Exige a validação do emissor.                               |
| `ValidIssuer`              | Define o emissor esperado.                                  |
| `ValidateAudience`         | Exige a validação da audiência.                             |
| `ValidAudience`            | Define a API destinatária esperada.                         |
| `ValidateLifetime`         | Valida o período de validade do token.                      |
| `ValidateIssuerSigningKey` | Exige a validação da chave de assinatura.                   |
| `IssuerSigningKey`         | Informa a chave usada para validar a assinatura.            |
| `ValidAlgorithms`          | Restringe os algoritmos de assinatura aceitos.              |
| `ClockSkew`                | Define uma tolerância de relógio para a validação temporal. |

O exemplo utiliza `ClockSkew` de 30 segundos para tolerar pequenas diferenças entre relógios. Essa tolerância deve ser definida conscientemente conforme os requisitos da aplicação.

**A validação da assinatura, do emissor, da audiência e da expiração não deve ser desabilitada para fazer um token ser aceito.**

## 8.5. Ordem do middleware

A ordem das chamadas é importante:

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

O middleware de autenticação identifica o usuário a partir das credenciais apresentadas.

O middleware de autorização aplica as regras de acesso ao endpoint solicitado.

A autenticação deve ser executada antes da autorização.

## 8.6. Diferença entre a API emissora e a API consumidora

Em uma arquitetura real, a aplicação responsável por emitir tokens pode ser diferente da API que valida esses tokens.

**Provedor de identidade:**

- Autentica o usuário.
- Emite tokens conforme o fluxo autorizado.
- Gerencia chaves, expiração e renovação.

**API protegida:**

- Recebe o token.
- Valida a assinatura e as declarações.
- Identifica o usuário.
- Aplica as políticas de autorização.

Essa separação é recomendada quando a aplicação possui múltiplos clientes ou APIs.

# 9. Protegendo endpoints

Depois de configurar a autenticação, é possível exigir credenciais em controllers e actions utilizando o atributo `[Authorize]`.

## 9.1. Endpoint público

```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/publico")]
public class PublicoController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return Ok(new
        {
            mensagem = "Este endpoint é público."
        });
    }
}
```

Esse endpoint não exige autenticação explicitamente.

## 9.2. Endpoint protegido

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/usuarios")]
public class UsuariosController : ControllerBase
{
    [HttpGet]
    [Authorize]
    public IActionResult Get()
    {
        return Ok(new
        {
            mensagem = "Acesso autorizado."
        });
    }
}
```

Se o cliente não apresentar um token válido, a requisição normalmente será rejeitada com `401 Unauthorized`.

## 9.3. Proteger um controller inteiro

```csharp
[ApiController]
[Route("api/pedidos")]
[Authorize]
public class PedidosController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return Ok();
    }

    [HttpPost]
    public IActionResult Post()
    {
        return Ok();
    }
}
```

Nesse caso, todas as actions do controller exigem autenticação, salvo quando alguma regra específica altera esse comportamento.

## 9.4. Liberar uma action específica

O atributo `[AllowAnonymous]` permite acesso anônimo a um endpoint, mesmo quando existe uma exigência de autorização herdada do controller ou de uma política aplicável.

```csharp
[ApiController]
[Route("api/auth")]
[Authorize]
public class AuthController : ControllerBase
{
    [HttpGet("perfil")]
    public IActionResult Perfil()
    {
        return Ok();
    }

    [HttpPost("login")]
    [AllowAnonymous]
    public IActionResult Login()
    {
        return Ok(new
        {
            mensagem = "Endpoint público de login."
        });
    }
}
```

Em uma aplicação real, a action de login precisa implementar um fluxo de autenticação seguro. O exemplo acima demonstra somente o uso dos atributos.

## 9.5. Exigir autenticação por padrão

Em APIs que devem proteger quase todos os endpoints, é possível configurar uma política de fallback:

```csharp
using Microsoft.AspNetCore.Authorization;

builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(
        new AuthorizationPolicyBuilder()
            .RequireAuthenticatedUser()
            .Build()
    );
```

Com essa configuração, os endpoints que não possuem uma política explícita de autorização exigem um usuário autenticado.

Endpoints públicos devem ser declarados intencionalmente, utilizando `[AllowAnonymous]` quando apropriado.

Essa abordagem reduz o risco de esquecer acidentalmente a proteção de um endpoint.

# 10. Autorização por roles e políticas

Autenticação e autorização são etapas diferentes.

A autenticação verifica a identidade apresentada pelo cliente. A autorização determina se essa identidade pode realizar determinada operação.

## 10.1. Autorização por role

Considere um endpoint administrativo:

```csharp
[HttpDelete("{id:int}")]
[Authorize(Roles = "Admin")]
public IActionResult Delete(int id)
{
    return NoContent();
}
```

Nesse exemplo, o usuário precisa estar autenticado e possuir a role `Admin`, reconhecida pelo mecanismo de autorização configurado.

Um usuário autenticado sem essa permissão normalmente recebe `403 Forbidden`.

O nome e o formato da claim de role precisam ser compatíveis com o token emitido e com a configuração do ASP.NET Core.

## 10.2. Autorização por política

Para regras mais específicas, utilize políticas.

Registro da política:

```csharp
builder.Services.AddAuthorizationBuilder()
    .AddPolicy(
        "PedidosLeitura",
        policy => policy.RequireClaim(
            "permission",
            "pedidos:read"
        )
    );
```

Aplicação da política:

```csharp
[HttpGet]
[Authorize(Policy = "PedidosLeitura")]
public IActionResult Get()
{
    return Ok();
}
```

Nesse caso, o token precisa conter a claim `permission` com o valor `pedidos:read`, e a API precisa confiar na origem dessa declaração.

**Não confie em permissões fornecidas livremente pelo cliente.** As roles e as permissões devem ser emitidas ou validadas por uma autoridade confiável.

# 11. Consumo da API com React e TypeScript

Esta seção demonstra como consumir uma API protegida com Bearer Token.

Os exemplos pressupõem que a aplicação já obteve um access token por um processo de autenticação apropriado.

## 11.1. Realizando uma requisição com fetch

```typescript
const token = obterAccessToken();

const response = await fetch(
    "https://localhost:7028/api/usuarios",
    {
        method: "GET",
        headers: {
            Authorization: `Bearer ${token}`,
            Accept: "application/json"
        }
    }
);

if (!response.ok) {
    throw new Error(
        `Falha ao consultar usuários: HTTP ${response.status}`
    );
}

const usuarios = await response.json();
```

A função `obterAccessToken()` é ilustrativa. Sua implementação depende da arquitetura de autenticação utilizada.

Não armazene o token em uma variável global permanente nem presuma que um token salvo no navegador continua válido indefinidamente.

## 11.2. Criando um serviço reutilizável

Exemplo de arquivo `src/services/api.ts`:

```typescript
const API_BASE_URL = "https://localhost:7028";

export async function apiGet<T>(
    endpoint: string,
    accessToken: string
): Promise<T> {
    const response = await fetch(
        `${API_BASE_URL}${endpoint}`,
        {
            method: "GET",
            headers: {
                Authorization: `Bearer ${accessToken}`,
                Accept: "application/json"
            }
        }
    );

    if (!response.ok) {
        throw new Error(
            `Erro na API: HTTP ${response.status}`
        );
    }

    return response.json() as Promise<T>;
}
```

Exemplo de utilização:

```typescript
interface Usuario {
    id: number;
    nome: string;
    email: string;
}

const usuarios = await apiGet<Usuario[]>(
    "/api/usuarios",
    accessToken
);
```

O serviço centraliza a construção da requisição e facilita a manutenção dos cabeçalhos.

Em aplicações maiores, considere uma camada HTTP que trate erros, renovação de tokens quando suportada e encerramento da sessão.

## 11.3. Exemplo com Axios

Se a aplicação utiliza Axios, o token pode ser informado no cabeçalho da requisição:

```typescript
import axios from "axios";

const api = axios.create({
    baseURL: "https://localhost:7028"
});

export async function buscarUsuarios(
    accessToken: string
) {
    const response = await api.get(
        "/api/usuarios",
        {
            headers: {
                Authorization: `Bearer ${accessToken}`
            }
        }
    );

    return response.data;
}
```

Em projetos reais, o token deve ser obtido a partir do mecanismo de autenticação adotado, e não inserido diretamente no código.

## 11.4. Tratamento de erros HTTP

O frontend deve diferenciar os principais códigos de resposta:

```typescript
if (response.status === 401) {
    // Credencial ausente, inválida ou expirada.
}

if (response.status === 403) {
    // Usuário autenticado sem autorização suficiente.
}
```

Um erro `401` não significa necessariamente que a senha do usuário está incorreta. Ele pode indicar que o token expirou, está inválido ou não foi enviado.

Um erro `403` indica que o acesso foi negado, normalmente porque o usuário não possui a permissão necessária.

## 11.5. Armazenamento do token no navegador

Evite considerar `localStorage` a opção padrão para guardar tokens de acesso sensíveis.

Scripts maliciosos executados na origem da aplicação podem ler valores armazenados nesse mecanismo, o que torna ataques XSS especialmente relevantes.

Dependendo da arquitetura, uma alternativa é utilizar o padrão **Backend for Frontend (BFF)**, mantendo os tokens no servidor e utilizando cookies de sessão configurados com `HttpOnly`, `Secure` e uma política `SameSite` apropriada.

Cookies exigem análise e proteção contra CSRF, conforme o modelo de sessão adotado.

Para aplicações SPA que utilizam um provedor de identidade, siga as orientações oficiais desse provedor para aquisição, armazenamento em memória e renovação de tokens.

# 12. Autenticação no Swagger

O Swagger UI pode ser configurado para permitir que o desenvolvedor informe um Bearer Token e teste endpoints protegidos.

Em uma API ASP.NET Core que utiliza Swashbuckle, instale o pacote caso ainda não esteja presente:

```bash
dotnet add package Swashbuckle.AspNetCore
```

## 12.1. Configuração no Program.cs

Adicione a configuração de segurança ao registro do Swagger:

```csharp
using Microsoft.OpenApi.Models;

builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc(
        "v1",
        new OpenApiInfo
        {
            Title = "API Gestão de Pedidos",
            Version = "v1"
        }
    );

    options.AddSecurityDefinition(
        "Bearer",
        new OpenApiSecurityScheme
        {
            Name = "Authorization",
            Type = SecuritySchemeType.Http,
            Scheme = "bearer",
            BearerFormat = "JWT",
            In = ParameterLocation.Header,
            Description =
                "Informe o access token no formato Bearer."
        }
    );

    options.AddSecurityRequirement(
        new OpenApiSecurityRequirement
        {
            {
                new OpenApiSecurityScheme
                {
                    Reference = new OpenApiReference
                    {
                        Type = ReferenceType.SecurityScheme,
                        Id = "Bearer"
                    }
                },
                Array.Empty<string>()
            }
        }
    );
});
```

No pipeline da aplicação, configure o Swagger UI conforme o modelo utilizado pelo projeto.

Exemplo:

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();

    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint(
            "/swagger/v1/swagger.json",
            "API Gestão de Pedidos v1"
        );
    });
}
```

Essa configuração pressupõe que o documento OpenAPI seja servido em `/swagger/v1/swagger.json`.

## 12.2. Como utilizar

1. Execute a API.
2. Abra a interface Swagger UI.
3. Clique em **Authorize**.
4. Informe o token de acesso válido.
5. Autorize a configuração.
6. Execute um endpoint protegido.

Como o esquema foi declarado como HTTP Bearer, normalmente basta informar o token, sem adicionar manualmente o prefixo `Bearer`.

A interface incluirá o cabeçalho de autorização nas requisições aplicáveis.

**Segurança:** restrinja o Swagger UI conforme as necessidades do ambiente. Não exponha credenciais reais em capturas de tela, logs ou documentação pública.

# 13. Códigos de resposta HTTP

Os códigos HTTP ajudam a identificar o resultado de uma requisição autenticada.

| Código                      | Significado                                       | Situação comum                         |
| --------------------------- | ------------------------------------------------- | -------------------------------------- |
| `200 OK`                    | Requisição processada com sucesso.                | Consulta realizada.                    |
| `201 Created`               | Recurso criado.                                   | Novo registro cadastrado.              |
| `204 No Content`            | Requisição bem-sucedida sem conteúdo de resposta. | Exclusão realizada.                    |
| `400 Bad Request`           | Requisição inválida.                              | Dados enviados incorretamente.         |
| `401 Unauthorized`          | Credenciais ausentes ou inválidas.                | Token expirado ou assinatura inválida. |
| `403 Forbidden`             | Acesso negado.                                    | Usuário sem a permissão exigida.       |
| `404 Not Found`             | Recurso não encontrado.                           | Endpoint ou registro inexistente.      |
| `500 Internal Server Error` | Erro interno do servidor.                         | Falha inesperada no processamento.     |

## 13.1. Diferença entre 401 e 403

**401 Unauthorized**

O cliente não apresentou uma credencial válida para o esquema de autenticação utilizado.

Exemplos:

- Token ausente.
- Token expirado.
- Assinatura inválida.
- Emissor incorreto.
- Audiência incorreta.

**403 Forbidden**

O cliente foi autenticado, mas não possui a autorização necessária para acessar o recurso.

Exemplo:

Um usuário autenticado tenta excluir um pedido, mas não possui a role administrativa exigida pelo endpoint.

Essa distinção facilita o diagnóstico de problemas e a implementação de tratamentos adequados no frontend.

# 14. Armazenamento e gerenciamento de tokens

Uma estratégia de autenticação deve considerar o ciclo de vida completo dos tokens.

## 14.1. Access Token

O access token é apresentado à API para acessar os recursos autorizados.

Boas práticas:

- Definir um prazo de validade apropriado.
- Restringir o token à audiência prevista.
- Solicitar somente as permissões necessárias.
- Evitar exposição em logs e ferramentas de monitoramento.
- Proteger o armazenamento e o transporte.
- Implementar estratégias de renovação compatíveis com o provedor de identidade.

## 14.2. Refresh Token

Um refresh token permite solicitar novos access tokens conforme as regras do provedor de identidade.

Ele não deve ser enviado indiscriminadamente às APIs de recursos como se fosse um access token.

Em arquiteturas com aplicações públicas, a renovação precisa seguir as recomendações do provedor de identidade, incluindo rotação de refresh tokens e mecanismos de detecção de reutilização, quando suportados.

## 14.3. Expiração

A claim `exp` define o instante após o qual o token não deve mais ser aceito.

Exemplo ilustrativo:

```json
{
  "sub": "123",
  "exp": 1792962000
}
```

A API deve validar o período de validade. A aplicação cliente não deve presumir que um token continua válido apenas porque ainda está armazenado.

## 14.4. Revogação

JWTs autocontidos podem continuar válidos até a expiração, mesmo após determinadas alterações no usuário ou em suas permissões.

Se a aplicação exigir revogação imediata, será necessário adotar uma estratégia apropriada, como:

- Tokens de curta duração.
- Revogação ou introspecção oferecida pelo provedor de identidade.
- Controle de sessões.
- Lista de bloqueio de tokens, quando justificada.
- Verificação de estado ou permissões em tempo de execução.

A escolha depende dos requisitos de segurança, desempenho e disponibilidade.

## 14.5. Rotação de chaves

As chaves utilizadas para assinar tokens devem possuir um processo controlado de rotação.

Em arquiteturas com assinatura assimétrica, o provedor de identidade pode publicar chaves públicas por meio de um endpoint JWKS. A API pode utilizar essas chaves para validar os tokens, conforme a configuração do provedor.

A rotação deve considerar o período em que tokens antigos ainda precisam ser validados.

# 15. Boas práticas de segurança

A implementação de Bearer Authentication deve considerar a segurança da emissão, da validação, do transporte e do armazenamento dos tokens.

## 15.1. Utilize HTTPS

Tokens Bearer são credenciais de acesso. Transmiti-los sem proteção pode permitir sua captura por terceiros.

Utilize HTTPS em todos os ambientes que transportam credenciais, incluindo produção.

## 15.2. Valide a assinatura

Não confie apenas no conteúdo decodificado do JWT.

A API deve verificar a assinatura com uma chave confiável e um algoritmo permitido.

## 15.3. Valide o emissor

Configure a API para aceitar somente tokens emitidos por autoridades confiáveis.

## 15.4. Valide a audiência

O token deve ser destinado à API que está recebendo a requisição.

Um token emitido para outro recurso não deve ser aceito indiscriminadamente.

## 15.5. Valide a expiração

Tokens expirados não devem ser aceitos como credenciais válidas.

## 15.6. Restrinja permissões

Utilize o princípio do menor privilégio.

Um usuário ou sistema deve receber apenas as permissões necessárias para realizar suas operações.

## 15.7. Não armazene segredos no payload

O payload de um JWT assinado não é necessariamente confidencial.

Nunca coloque senhas, chaves privadas ou outros segredos nesse conteúdo.

## 15.8. Não registre tokens em logs

Evite registrar:

- Access tokens.
- Refresh tokens.
- Senhas.
- Chaves de assinatura.
- Cabeçalhos `Authorization` completos.

Quando necessário para diagnóstico, registre apenas informações não sensíveis, como o identificador de correlação da requisição e o código HTTP.

## 15.9. Evite chaves fracas

Para algoritmos de assinatura simétrica, utilize chaves aleatórias fortes e compatíveis com os requisitos do algoritmo.

Não utilize senhas humanas como chaves de assinatura.

## 15.10. Prefira provedores de identidade confiáveis

Para produção, prefira um provedor de identidade que implemente padrões reconhecidos, como OAuth 2.0 e OpenID Connect.

Isso reduz a necessidade de implementar manualmente fluxos sensíveis, emissão de tokens, renovação, rotação de chaves e mecanismos de segurança.

## 15.11. Considere ataques XSS e CSRF

A estratégia de armazenamento deve considerar o tipo de aplicação.

- Tokens acessíveis ao JavaScript podem ser expostos em ataques XSS.
- Cookies enviados automaticamente pelo navegador exigem atenção a CSRF.
- Cookies de sessão devem utilizar atributos de segurança adequados.
- A política de origem cruzada (CORS) não substitui a autenticação ou a autorização.

## 15.12. Não desabilite validações para corrigir erros

Se a API rejeitar um token, investigue o motivo.

Não desabilite a validação de assinatura, expiração, emissor ou audiência apenas para eliminar um erro durante os testes.

# 16. Erros comuns e soluções

## 16.1. HTTP 401 — Unauthorized

**Possíveis causas:**

- Token não enviado.
- Token expirado.
- Assinatura inválida.
- Emissor ou audiência incompatível.
- Chave de validação incorreta.
- Cabeçalho `Authorization` malformado.

**Como investigar:**

1. Verifique se o cabeçalho foi enviado.
2. Confirme se o token foi emitido para a API correta.
3. Confira a expiração.
4. Valide os parâmetros de assinatura.
5. Consulte os logs de autenticação sem registrar o token completo.

## 16.2. HTTP 403 — Forbidden

**Possíveis causas:**

- Usuário sem a role exigida.
- Claim de permissão ausente.
- Política de autorização não atendida.
- Regras de acesso ao recurso não satisfeitas.

**Como investigar:**

1. Confirme se a autenticação foi bem-sucedida.
2. Verifique as claims efetivamente reconhecidas pela API.
3. Confira as roles e as políticas configuradas.
4. Revise as permissões atribuídas ao usuário.

## 16.3. Token válido em uma API e rejeitado em outra

Uma API não deve aceitar automaticamente qualquer JWT apenas porque ele é válido em outra aplicação.

Verifique:

- `iss`: emissor.
- `aud`: audiência.
- Algoritmo e assinatura.
- Chaves de validação.
- Prazo de validade.
- Políticas de autorização.

É comum que cada API tenha uma audiência específica.

## 16.4. Claim de role não reconhecida

O ASP.NET Core pode mapear claims de entrada conforme a configuração do esquema de autenticação.

Se uma role não for reconhecida, verifique:

- O nome da claim emitida pelo provedor.
- A configuração de `RoleClaimType`.
- O mapeamento de claims.
- O valor exato da role.

A configuração deve ser compatível com o formato do token.

## 16.5. Erro de CORS

CORS e autenticação são mecanismos diferentes.

Se uma aplicação React estiver em uma origem diferente da API, o navegador poderá exigir uma resposta CORS apropriada, especialmente em requisições que incluem o cabeçalho `Authorization`.

Uma API pode retornar `401` corretamente e, ainda assim, apresentar um problema de CORS se os cabeçalhos necessários não estiverem configurados.

Da mesma forma, configurar CORS não torna um endpoint autenticado.

## 16.6. Diferença de relógio

Diferenças entre os relógios do emissor e da API podem causar falhas na validação temporal.

Mantenha os servidores sincronizados e configure uma tolerância de relógio adequada, sem utilizá-la para compensar problemas permanentes.

# 17. Checklist para produção

Antes de disponibilizar uma API protegida com JWT Bearer, verifique os itens abaixo.

### Emissão e identidade

- [ ] Utilizar um fluxo de autenticação padronizado.
- [ ] Preferir um provedor de identidade confiável.
- [ ] Emitir tokens destinados à API correta.
- [ ] Definir uma política de expiração apropriada.
- [ ] Implementar renovação e revogação conforme os requisitos.

### Validação na API

- [ ] Validar a assinatura.
- [ ] Validar o emissor.
- [ ] Validar a audiência.
- [ ] Validar a expiração.
- [ ] Restringir os algoritmos aceitos.
- [ ] Aplicar políticas de autorização.
- [ ] Proteger endpoints por padrão quando apropriado.

### Proteção de credenciais

- [ ] Utilizar HTTPS.
- [ ] Armazenar chaves e segredos fora do código-fonte.
- [ ] Não registrar tokens completos em logs.
- [ ] Não incluir informações sensíveis no payload.
- [ ] Definir uma estratégia de armazenamento seguro no cliente.
- [ ] Avaliar riscos de XSS e CSRF.
- [ ] Implementar rotação de chaves.

### Operação e manutenção

- [x] Monitorar falhas de autenticação.
- [ ] Evitar exposição de detalhes sensíveis em mensagens de erro.
- [ ] Testar tokens expirados e inválidos.
- [ ] Testar usuários sem permissões.
- [ ] Documentar o processo de renovação.
- [ ] Revisar periodicamente as políticas de acesso.
- [ ] Manter os pacotes de segurança atualizados.

# 18. Glossário

| Termo          | Definição                                                                                    |
| -------------- | -------------------------------------------------------------------------------------------- |
| API            | Interface que permite a comunicação entre aplicações.                                        |
| Access Token   | Credencial utilizada para acessar recursos protegidos.                                       |
| Authentication | Processo de identificação e validação de uma identidade.                                     |
| Authorization  | Processo de determinar quais recursos uma identidade pode acessar.                           |
| Bearer         | Esquema HTTP utilizado para apresentar um token de acesso.                                   |
| Claim          | Declaração sobre um usuário, sistema ou contexto.                                            |
| CORS           | Mecanismo que controla o acesso a recursos entre diferentes origens no navegador.            |
| JWT            | Formato compacto de token que transporta declarações.                                        |
| OAuth 2.0      | Framework de autorização e obtenção de tokens.                                               |
| OpenID Connect | Camada de identidade sobre OAuth 2.0.                                                        |
| Payload        | Conteúdo do JWT que contém suas declarações.                                                 |
| Refresh Token  | Credencial utilizada para solicitar novos access tokens conforme as regras do provedor.      |
| Role           | Função utilizada para agrupar permissões ou responsabilidades.                               |
| Signature      | Assinatura que permite verificar a integridade e a origem do token.                          |
| Token          | Credencial utilizada para representar uma autorização ou identidade em determinado contexto. |

# 19. Referências oficiais

As referências abaixo são recomendadas para aprofundamento e consulta durante a implementação.

### Microsoft Learn

- [Configurar a autenticação JWT Bearer no ASP.NET Core](https://learn.microsoft.com/pt-br/aspnet/core/security/authentication/configure-jwt-bearer-authentication)
- [Visão geral da autenticação no ASP.NET Core](https://learn.microsoft.com/pt-br/aspnet/core/security/authentication/)
- [Autorização baseada em funções no ASP.NET Core](https://learn.microsoft.com/pt-br/aspnet/core/security/authorization/roles)
- [Políticas de autorização no ASP.NET Core](https://learn.microsoft.com/pt-br/aspnet/core/security/authorization/policies)

### Padrões e especificações

- [RFC 6750 — OAuth 2.0 Bearer Token Usage](https://datatracker.ietf.org/doc/html/rfc6750)
- [RFC 7519 — JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
- [OAuth 2.0 — RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)
- [OpenID Connect](https://openid.net/developers/how-connect-works/)

---

## Conclusão

A autenticação Bearer permite que clientes apresentem tokens de acesso em requisições HTTP para consumir recursos protegidos.

Quando combinada com JWT, pode transportar declarações assinadas que a API valida para estabelecer a identidade do solicitante e aplicar regras de autorização.

Uma implementação segura exige mais do que receber um token: é necessário validar sua assinatura, seu emissor, sua audiência e sua validade temporal, além de proteger as credenciais durante a emissão, o transporte, o armazenamento e a renovação.

Para aplicações de produção, a recomendação é utilizar OAuth 2.0 e OpenID Connect por meio de um provedor de identidade confiável, mantendo a API responsável pela validação dos tokens e pela aplicação das políticas de acesso.

**Princípio fundamental:** um token válido comprova somente aquilo que sua validação e suas declarações confiáveis permitem concluir; a autorização para acessar cada recurso deve continuar sendo aplicada pela API.