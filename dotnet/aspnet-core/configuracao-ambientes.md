# Configuração de Ambientes no ASP.NET Core

## 1. Objetivo

O ASP.NET Core permite executar uma mesma aplicação em diferentes ambientes:

- **Development:** desenvolvimento e depuração.
- **Staging:** homologação e validação antes da publicação definitiva.
- **Production:** ambiente de produção, utilizado nas operações reais.

O ambiente determina quais configurações são carregadas e permite habilitar ou desabilitar funcionalidades conforme o contexto de execução.

| Recurso | Responsabilidade |
|---|---|
| Máquina servidor | Define variáveis de ambiente no sistema operacional ou na infraestrutura. |
| `web.config` | Configura a hospedagem no IIS, incluindo variáveis de ambiente da aplicação. |
| `PublishProfile.pubxml` | Define configurações utilizadas durante a publicação. |
| `appsettings.json` | Armazena configurações gerais da aplicação. |
| `appsettings.{Environment}.json` | Armazena configurações específicas de cada ambiente. |
| `Program.cs` | Inicializa a aplicação e utiliza as configurações carregadas. |

## 2. Configuração da máquina servidor

O ambiente pode ser definido por variáveis de ambiente no Windows ou na infraestrutura de hospedagem.

Variáveis relevantes:

- `ASPNETCORE_ENVIRONMENT`
- `DOTNET_ENVIRONMENT`

Exemplo:

```text
ASPNETCORE_ENVIRONMENT = Staging
```

**Importante:** o ASP.NET Core não identifica automaticamente o ambiente pelo domínio ou por estar sendo executado em um servidor. Ele utiliza as configurações fornecidas durante a inicialização.

Quando várias aplicações são hospedadas no mesmo IIS, prefira configurações específicas por aplicação em vez de depender exclusivamente de uma variável global do Windows.

## 3. Configuração no `web.config`

O `web.config` é utilizado pelo IIS para configurar a hospedagem da aplicação ASP.NET Core.

Exemplo de configuração para homologação:

```xml
<aspNetCore processPath="dotnet"
            arguments=".\MinhaApi.dll"
            hostingModel="inprocess">

    <environmentVariables>
        <environmentVariable
            name="ASPNETCORE_ENVIRONMENT"
            value="Staging" />
    </environmentVariables>

</aspNetCore>
```

Para produção, altere o valor:

```xml
<environmentVariable
    name="ASPNETCORE_ENVIRONMENT"
    value="Production" />
```

O exemplo mostra apenas o trecho relevante. Preserve as demais configurações geradas no `web.config` durante a publicação.

Quando o IIS inicia a aplicação, as variáveis configuradas no `web.config` são fornecidas ao processo da aplicação. Para a variável `ASPNETCORE_ENVIRONMENT`, o valor no `web.config` prevalece sobre o valor da mesma variável definido no ambiente do sistema operacional.

## 4. Configuração no `PublishProfile.pubxml`

Os perfis de publicação normalmente ficam em:

```text
Properties/
└── PublishProfiles/
    ├── Homologacao.pubxml
    └── Producao.pubxml
```

### Perfil de homologação

Arquivo `Homologacao.pubxml`:

```xml
<Project>
  <PropertyGroup>
    <EnvironmentName>Staging</EnvironmentName>
  </PropertyGroup>
</Project>
```

### Perfil de produção

Arquivo `Producao.pubxml`:

```xml
<Project>
  <PropertyGroup>
    <EnvironmentName>Production</EnvironmentName>
  </PropertyGroup>
</Project>
```

A propriedade `EnvironmentName` pode fazer com que o processo de publicação gere a variável de ambiente correspondente no `web.config`.

O `.pubxml` é utilizado durante a publicação; ele não é, por si só, a configuração de ambiente consultada pela aplicação em execução. Depois de publicar, confira o `web.config` gerado e as configurações efetivas do servidor.

## 5. Arquivos `appsettings`

Estrutura comum:

```text
MinhaApi/
├── Program.cs
├── appsettings.json
├── appsettings.Development.json
├── appsettings.Staging.json
└── appsettings.Production.json
```

### `appsettings.json`

Armazena configurações gerais compartilhadas pelos ambientes:

```json
{
  "Application": {
    "Name": "MinhaApi"
  }
}
```

### `appsettings.Staging.json`

```json
{
  "Application": {
    "Environment": "Staging"
  }
}
```

### `appsettings.Production.json`

```json
{
  "Application": {
    "Environment": "Production"
  }
}
```

Com a configuração padrão de `WebApplication.CreateBuilder(args)`, o ASP.NET Core carrega `appsettings.json` e, em seguida, o arquivo específico do ambiente efetivo. Os valores do arquivo específico prevalecem sobre os valores correspondentes do arquivo geral.

**Atenção:** os arquivos `appsettings` não definem automaticamente o ambiente. O ambiente determina qual arquivo específico será carregado.

Não armazene senhas ou credenciais reais em arquivos versionados no Git. Utilize mecanismos seguros para gerenciar segredos.

## 6. Precedência das variáveis de ambiente

No modelo moderno, utilizando:

```csharp
var builder = WebApplication.CreateBuilder(args);
```

`DOTNET_ENVIRONMENT` tem precedência sobre `ASPNETCORE_ENVIRONMENT` quando ambas estão definidas.

Além disso, no IIS, o valor de `ASPNETCORE_ENVIRONMENT` configurado no `web.config` prevalece sobre o valor da mesma variável definido no ambiente do sistema operacional.

Exemplo:

| Local | Variável | Valor |
|---|---|---|
| Windows | `ASPNETCORE_ENVIRONMENT` | `Staging` |
| `web.config` | `ASPNETCORE_ENVIRONMENT` | `Production` |
| Resultado, se `DOTNET_ENVIRONMENT` não estiver definido com outro valor | Ambiente efetivo | `Production` |

Se `DOTNET_ENVIRONMENT` estiver definido como `Staging`, ele poderá prevalecer sobre `ASPNETCORE_ENVIRONMENT` no modelo moderno.

**Recomendação:** evite definir as duas variáveis com valores diferentes. Mantenha uma configuração consistente para reduzir ambiguidades.

## 7. Utilização no `Program.cs`

O ambiente pode ser consultado por meio de `builder.Environment` ou `app.Environment`:

```csharp
var builder = WebApplication.CreateBuilder(args);

Console.WriteLine(
    $"Ambiente: {builder.Environment.EnvironmentName}"
);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

if (app.Environment.IsStaging())
{
    Console.WriteLine("Executando em Homologação");
}

if (app.Environment.IsProduction())
{
    Console.WriteLine("Executando em Produção");
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

Se nenhuma configuração de ambiente for fornecida, o padrão será `Production`, desde que não exista outra configuração aplicável.

`IsDevelopment()` não verifica se a aplicação está sendo executada localmente; verifica se o ambiente efetivo é `Development`.

## 8. Fluxo de publicação no IIS

1. Selecione um perfil de publicação no Visual Studio.
2. O processo de publicação gera os arquivos necessários, incluindo o `web.config`.
3. Os arquivos são publicados no servidor IIS.
4. O IIS inicia a aplicação com as configurações de hospedagem.
5. O ASP.NET Core determina o ambiente efetivo.
6. A aplicação carrega as configurações gerais e específicas desse ambiente.
7. O `Program.cs` habilita ou desabilita funcionalidades conforme o ambiente.

Exemplo:

| Configuração | Homologação | Produção |
|---|---|---|
| Perfil | `Homologacao.pubxml` | `Producao.pubxml` |
| Ambiente | `Staging` | `Production` |
| Arquivo específico | `appsettings.Staging.json` | `appsettings.Production.json` |
| Banco de dados | Banco de homologação | Banco de produção |
| Swagger | Opcional, com acesso controlado | Normalmente desabilitado |

Os ambientes devem ter configurações e recursos operacionais separados conforme a necessidade.

## 9. Boas práticas

- Configure explicitamente o ambiente de cada aplicação publicada.
- Utilize perfis de publicação distintos para homologação e produção.
- Verifique o `web.config` gerado após a publicação.
- Mantenha os arquivos `appsettings` organizados por ambiente.
- Não utilize `Development` em aplicações públicas de produção.
- Não exponha detalhes internos de exceções em produção.
- Separe os bancos de dados de homologação e produção.
- Não versione credenciais reais em arquivos de configuração.
- Registre o ambiente efetivo nos logs de inicialização.
- Evite definir `DOTNET_ENVIRONMENT` e `ASPNETCORE_ENVIRONMENT` com valores conflitantes.

## 10. Referências oficiais

- [Ambientes do ASP.NET Core](https://learn.microsoft.com/pt-br/aspnet/core/fundamentals/environments)
- [Hospedagem no IIS e `web.config`](https://learn.microsoft.com/pt-br/aspnet/core/host-and-deploy/iis/web-config)
- [Configuração no ASP.NET Core](https://learn.microsoft.com/pt-br/aspnet/core/fundamentals/configuration)

## Conclusão

O ambiente efetivo é determinado durante a inicialização da aplicação. Ele orienta o carregamento das configurações específicas e permite adaptar o comportamento da API para Desenvolvimento, Homologação ou Produção.
