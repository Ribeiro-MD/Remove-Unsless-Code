---
name: removeunslesscode-csharp
description: Remove código morto (não usado) de projetos C# (.NET, WinForms, WPF, ASP.NET, Unity) — usings, variáveis, membros, classes e ficheiros órfãos — com verificação pelo compilador.
---

# Remover código morto — C#

Trabalhas sozinho (sem subagentes). Fases: **detetar → verificar → remover → confirmar → resumo**. Não mudas comportamento, só tiras o que não é usado.

## Regras

- **Na dúvida, fica.** Só se remove o que tem ZERO referências confirmadas, incluindo no Designer, XAML e reflexão.
- **O compilador é a prova.** Depois de remover, `dotnet build` (ou `msbuild`) tem de passar.
- **Nunca editar `*.Designer.cs` à mão** além de remover uma linha `+=` de um evento cujo handler foi confirmado como morto — e mesmo isso só se pedido. O Designer do Visual Studio regenera esse ficheiro.
- **Nada de co-autor.** Commits sem `Co-Authored-By` nem qualquer menção ao Claude.
- **Mais de 50 candidatos → para** e pergunta se é para avançar ou reduzir o âmbito.
- Sem git: faz cópia `*.bak` antes de mexer e no fim entrega os ficheiros completos.

## Âmbito

Argumento: ficheiro → só esse; pasta → só essa; nome de símbolo → só esse. Sem argumento → solução toda. Ignora `bin/`, `obj/`, `packages/`, `Library/` e `Temp/` (Unity).

---

## FASE 1 — Identificar o projeto e estado inicial

- `*.sln` / `*.csproj` → `dotnet build` (projetos SDK) ou `msbuild` (.NET Framework antigo)
- WinForms/WPF fora do Windows: `dotnet build -p:EnableWindowsTargeting=true` (compila mas não corre). Se for .NET Framework antigo e não compilar aqui, avisar e trabalhar só com verificação por grep, mais conservador.
- Unity (`Assets/`, `ProjectSettings/`): não há build por linha de comandos simples — verificação só por grep e muito conservadora; avisar para abrir no Editor no fim.

Compilar antes de mexer e guardar os avisos.

## FASE 2 — Detetar

Avisos do compilador que interessam:

| Código | O quê |
|--------|-------|
| CS0168 | variável declarada e nunca usada |
| CS0219 | variável atribuída e nunca lida |
| CS0169 / CS0414 | campo privado nunca usado / atribuído e nunca lido |
| CS8321 | função local nunca usada |
| CS0162 | código inalcançável |
| IDE0051 / IDE0052 | membro privado nunca usado / nunca lido |
| IDE0005 | `using` desnecessário |
| IDE0060 | parâmetro não usado |

Os `IDE*` precisam de ser ativados só para a análise:

```bash
dotnet build -p:EnforceCodeStyleInBuild=true -p:GenerateDocumentationFile=true \
  -warnaserror- 2>&1 | grep -E "CS0168|CS0219|CS0169|CS0414|CS8321|CS0162|IDE0051|IDE0052|IDE0005|IDE0060"
```

Alternativa só para listar (sem alterar): `dotnet format analyzers --verify-no-changes --diagnostics IDE0051 IDE0052 IDE0005 IDE0060 --severity info`

Depois, com grep:
- **Classes e ficheiros órfãos**: nome da classe não aparece em mais nenhum `.cs`, `.xaml`, `.resx`, `.config`, `.json`, nem no `.csproj`.
- **Métodos públicos/internal** sem chamadas em nenhum projeto da solução.
- **Event handlers órfãos em WinForms**: método `xxx_Click(object sender, EventArgs e)` no `Form.cs` que NÃO aparece no `Form.Designer.cs` (`+= new ...(xxx_Click)` ou `+= xxx_Click`) nem noutro `+=`. Acontece quando se apaga um botão no Designer e o handler fica.
- **Código comentado** e blocos `#if false`.

### Nunca marcar como morto

- `Program.Main`, `static void Main`, top-level statements, `Startup`/`Program` do ASP.NET
- `*.Designer.cs`, `InitializeComponent`, controlos declarados no Designer, `.resx`, `Settings.settings`, `App.config`
- Event handlers ligados no Designer, em XAML (`Click="..."`) ou por `+=` em código
- WPF/MAUI: tudo usado em XAML (`x:Name`, `{Binding Prop}`, `Command=`, converters, `x:Class`), `INotifyPropertyChanged` e propriedades ligadas por binding
- ASP.NET: controllers e ações (`[HttpGet]`, `[Route]`), Razor Pages (`OnGet`, `OnPost`), middlewares, serviços registados em DI, `appsettings.json`
- Unity: `Awake`, `Start`, `Update`, `FixedUpdate`, `LateUpdate`, `OnEnable`, `OnCollisionEnter`, etc.; campos `public` ou `[SerializeField]` (configurados no Inspector); métodos chamados por `UnityEvent`/botões no Inspector ou por `SendMessage`/`Invoke("nome")`
- `override`, implementações de interface, `partial` methods, operadores, finalizers
- Reflexão e serialização: `[Serializable]`, `[DataMember]`, `[JsonProperty]`, propriedades de DTOs/modelos (EF Core, Dapper, OleDb mapeados por nome), `Activator.CreateInstance`, `GetMethod("nome")`, `nameof`
- `[DllImport]`/`[LibraryImport]`, `[ComVisible]`, atributos de teste (`[Test]`, `[Fact]`, `[TestMethod]`) e o que os testes usam
- APIs `public` de bibliotecas (outros projetos podem usar)

## FASE 3 — Remover

- `using` desnecessário: apagar a linha.
- Variável / campo / método privado / função local / código inalcançável / código comentado: apagar e limpar linhas em branco a mais.
- Event handler órfão (confirmado que não está no Designer nem em `+=`): apagar o método.
- Parâmetro não usado num `override`, implementação de interface ou event handler: deixar ficar (a assinatura é obrigatória). Noutros casos privados: renomear para `_` ou apagar se atualizares TODAS as chamadas no mesmo passo.
- Classe/ficheiro órfão: `git rm` e, em projetos antigos (não-SDK), tirar a linha `<Compile Include="...">` do `.csproj`.

Depois de cada ficheiro: `dotnet build`. Falhou → `git checkout -- ficheiro` e marcar como "saltado".

## FASE 4 — Commits

- Um commit por ficheiro (ou `Form.cs` + `Form.Designer.cs` juntos), só com esses ficheiros: `git add Form1.cs` — nunca `git add -A`.
- Mensagem: `refactor: remove código morto de Form1.cs`
- Sem `Co-Authored-By` nem menção ao Claude.

## FASE 5 — Verificação final e resumo

```bash
dotnet clean && dotnet build
dotnet test     # se houver projeto de testes
```

Comparar avisos com os de antes. Em WinForms/WPF compilado fora do Windows ou Unity, avisar que convém abrir no Visual Studio/Editor e confirmar que os formulários abrem.

Entregar os ficheiros alterados completos, prontos a copiar.

```markdown
## Código morto removido

| # | Ficheiro | O quê | Tipo | Commit |
|---|----------|-------|------|--------|
| 1 | Form1.cs | using System.Linq | using | abc1234 |
| 2 | Form1.cs | button3_Click | handler órfão | abc1234 |

### Saltado
| # | Ficheiro | O quê | Porquê |
|---|----------|-------|--------|

### Verificação
- Build: PASSOU/FALHOU
- Testes: X a passar, Y a falhar (Z já falhavam)
- Por confirmar no Windows/Editor: sim/não
- Total: N itens em M ficheiros, K commits
```

Se não houver nada confirmado: dizer "Não encontrei código morto" e parar.
