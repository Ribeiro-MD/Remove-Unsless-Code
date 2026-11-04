---
name: codigo-morto
description: Remove código morto (não usado) de projetos Java (Fabric/Paper/Gradle/Maven) e HTML/CSS com verificação antes de apagar. Usar em vez de remove-deadcode para estes stacks.
---

# Remover código morto — Java + HTML/CSS

Trabalhas sozinho (sem subagentes). Fases: **detetar → verificar → remover → confirmar → resumo**. Não mudas comportamento, só tiras o que não é usado.

## Regras

- **Na dúvida, fica.** Só se remove o que tem ZERO referências confirmadas.
- **O compilador / o browser é a prova.** Depois de remover, tem de compilar (Java) e a página tem de ficar igual (HTML/CSS).
- **Nada de co-autor.** Commits sem `Co-Authored-By` nem qualquer menção ao Claude.
- **Mais de 50 candidatos → para** e pergunta se é para avançar ou reduzir o âmbito.
- Se o projeto não tiver git, faz cópia `*.bak` dos ficheiros antes de mexer (ou trabalha numa cópia) e no fim entrega os ficheiros completos.

## Âmbito

Se vier argumento: ficheiro → só esse ficheiro; pasta → só essa pasta; nome de símbolo/classe CSS → só esse. Sem argumento → projeto todo.

---

## FASE 1 — Detetar o tipo de projeto

- `build.gradle(.kts)` / `pom.xml` / `*.java` → parte Java
- `*.html` / `*.css` (e `*.js` ligados) → parte web
- `fabric.mod.json` → mod Fabric; `plugin.yml` / `paper-plugin.yml` → plugin Paper

Corre o build ANTES de mexer em nada e guarda o resultado (avisos/erros já existentes não são culpa tua).

---

## FASE 2 — Java

### Detetar

1. Compilar com avisos:
   - Gradle: `./gradlew compileJava --warning-mode all` (se precisares, adiciona temporariamente `options.compilerArgs += ['-Xlint:all']` e tira no fim)
   - Maven: `mvn -q compile -Dmaven.compiler.showWarnings=true`
2. Procurar candidatos com grep:
   - **Imports não usados**: para cada `import a.b.Nome;`, verificar se `Nome` aparece no resto do ficheiro. Imports `*` e `static` com cuidado.
   - **Membros `private`** (campos, métodos, classes internas) sem uso no próprio ficheiro.
   - **Variáveis locais** atribuídas e nunca lidas.
   - **Classes/ficheiros órfãos**: nome da classe não aparece em mais nenhum `.java`, nem em `fabric.mod.json`, `*.mixins.json`, `plugin.yml`, `META-INF/services`, nem em recursos.
   - **Métodos públicos/package** sem chamadas em lado nenhum do projeto.
   - **Código inalcançável**: depois de `return`/`throw`, `if (false)`, constantes `DEBUG = false` com blocos mortos.
   - **Blocos de código comentado** (código antigo em `//` ou `/* */`, não comentários explicativos).

### Nunca marcar como morto (Java)

- `main`, construtores usados por reflexão, classes de entrypoint (`ModInitializer`, `ClientModInitializer`, `DataGeneratorEntrypoint`, classe `main` do `plugin.yml`, `JavaPlugin.onEnable/onDisable/onLoad`)
- Tudo o que esteja em `fabric.mod.json`, `*.mixins.json`, `*.accesswidener`, `plugin.yml`, `META-INF/services`
- Mixins e tudo com `@Mixin`, `@Inject`, `@Redirect`, `@ModifyVariable`, `@ModifyArg`, `@Shadow`, `@Accessor`, `@Invoker`, `@Unique`, `@Overwrite`
- `@Override` (implementa interface/superclasse), `@EventHandler`, `@SubscribeEvent`, métodos de `Listener`, `CommandExecutor`, `TabCompleter`
- Campos de registo (`ModItems`, `ModBlocks`, `ModSounds`, etc.) — o registo acontece no carregamento da classe, mesmo que ninguém leia o campo
- Campos usados em serialização/config (Gson, Jackson, YAML, records de config) — o nome é lido por reflexão
- `serialVersionUID`, constantes públicas de API, classes em `api/`
- Classes de teste e o que os testes usam
- Parâmetros de métodos que implementam/sobrepõem outro (não se remove; deixa ficar)

### Remover (Java)

- Import não usado: apagar a linha.
- Membro privado / local / classe órfã / código inalcançável / código comentado: apagar e limpar linhas em branco a mais.
- Parâmetro não usado de método `private` (que não sobrepõe nada): só remover se atualizares TODAS as chamadas no mesmo passo; senão deixa.
- Ficheiro órfão: `git rm` (ou apagar se não houver git).

Depois de cada ficheiro: `./gradlew compileJava` (ou `mvn -q compile`). Falhou → reverter esse ficheiro (`git checkout -- ficheiro`) e marcar como "saltado".

---

## FASE 3 — HTML / CSS

### Detetar

1. **Seletores CSS não usados**: para cada classe `.x` e id `#y` nos `.css` e nos `<style>`, procurar em TODOS os `.html` e `.js` do projeto: `class="... x ..."`, `id="y"`, `classList.add/toggle/remove('x')`, `className`, `querySelector('.x')`, `getElementById('y')`, strings e template strings com o nome.
2. **Regras vazias** (`.a { }`) e **declarações sobrepostas** na mesma regra (a mesma propriedade duas vezes — fica a última).
3. **Regras duplicadas** (mesmo seletor e mesmas declarações) → juntar numa.
4. **`@keyframes`** sem `animation`/`animation-name` a usá-las; **variáveis CSS** (`--x`) definidas e nunca usadas em `var(--x)`; **`@font-face`** com fonte nunca usada.
5. **Ficheiros órfãos**: `.css`, `.js`, imagens, fontes e ícones na pasta que não são referenciados por nenhum `href`, `src`, `url()`, `import`, `srcset` nem pelo `manifest`.
6. **HTML**: blocos comentados `<!-- ... -->` com markup antigo, `<link>`/`<script>` para ficheiros que não existem, atributos repetidos, funções em `<script>` nunca chamadas (nem por `onclick=`, `addEventListener`, nem expostas em `window`).

### Nunca marcar como morto (web)

- Classes/ids montados dinamicamente (`'btn-' + tipo`, `` `tema-${x}` ``) — se o prefixo aparece em JS, ficam todas as classes com esse prefixo
- Estados e pseudo-classes (`:hover`, `:focus`, `.active`, `.open`, `.hidden`, `.dark`) que o JS liga/desliga
- Ids usados como âncoras (`href="#y"`), `for=` de labels, `aria-*`, `name` de formulários
- Regras dentro de `@media`, `@supports`, `prefers-color-scheme`, `@media print` (só parecem não fazer nada no ecrã atual)
- Classes de bibliotecas externas (Bootstrap, Tailwind, ícones) e CSS que estiliza HTML gerado por JS
- `index.html`, `favicon`, `manifest.json`, `robots.txt`, `404.html`, `CNAME`
- Comentários que explicam o código (só se tira código comentado)

### Remover e confirmar (web)

- Apagar o que foi confirmado, um ficheiro de cada vez.
- Confirmar que a página fica igual: se houver browser disponível (Playwright/Chromium ou o browser da sessão), tirar screenshot da página **antes e depois** em largura de desktop e de telemóvel e comparar; testar menus, separadores e modais que dependem de JS. Se não houver browser, validar que não ficam `href`/`src` partidos e avisar que convém abrir a página para confirmar.
- Se algo mudou visualmente → repor esse ficheiro e marcar como "saltado".

---

## FASE 4 — Commits

Se houver git:
- Um commit por ficheiro (ou por grupo pequeno de ficheiros relacionados), só com os ficheiros desse passo: `git add ficheiro1 ficheiro2` — nunca `git add -A`.
- Mensagem: `refactor: remove código morto de NomeDoFicheiro`
- Sem `Co-Authored-By` nem menção ao Claude.

---

## FASE 5 — Verificação final e resumo

- Java: `./gradlew build` (ou `mvn -q package`) tem de passar; comparar avisos com os de antes.
- Web: todas as páginas abrem sem recursos partidos (e screenshots iguais, se houve browser).
- Entregar os ficheiros alterados completos, prontos a copiar.

Resumo:

```markdown
## Código morto removido

| # | Ficheiro | O quê | Tipo | Commit |
|---|----------|-------|------|--------|
| 1 | ModItems.java | import java.util.List | import | abc1234 |
| 2 | style.css | .card-old | seletor CSS | def5678 |

### Saltado
| # | Ficheiro | O quê | Porquê |
|---|----------|-------|--------|

### Verificação
- Build: PASSOU/FALHOU
- Páginas: iguais / por confirmar
- Total: N itens em M ficheiros, K commits
```

Se não houver nada confirmado: dizer "Não encontrei código morto" e parar.
