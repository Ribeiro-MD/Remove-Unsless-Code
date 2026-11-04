---
name: removeunslesscode-css
description: Remove CSS morto (seletores, regras, keyframes, variáveis e ficheiros não usados) de sites e apps web, incluindo SCSS/LESS, com verificação visual antes e depois.
---

# Remover código morto — CSS

Trabalhas sozinho (sem subagentes). Fases: **detetar → verificar → remover → confirmar → resumo**. A página tem de ficar exatamente igual.

## Regras

- **Na dúvida, fica.** CSS é ligado por JS, por estados e por media queries — só se remove o que tem ZERO referências confirmadas.
- **O browser é a prova.** Screenshots antes e depois têm de ser iguais.
- **Nada de co-autor.** Commits sem `Co-Authored-By` nem qualquer menção ao Claude.
- **Mais de 50 candidatos → para** e pergunta se é para avançar ou reduzir o âmbito.
- Sem git: faz cópia `*.bak` antes de mexer e no fim entrega os ficheiros completos.

## Âmbito

Argumento: ficheiro → só esse; pasta → só essa; seletor → só esse. Sem argumento → todo o CSS do projeto (`.css`, `.scss`, `.sass`, `.less`, `<style>` dentro de `.html`, `style` em componentes `.vue`/`.svelte`). Ignora `node_modules/`, `dist/`, `build/` e CSS minificado de terceiros (`*.min.css`).

---

## FASE 1 — Estado inicial

- Listar as páginas (`*.html`) e os ficheiros onde o markup é gerado (`.js`, `.ts`, `.jsx`, `.tsx`, `.vue`, `.php`, templates).
- Se houver browser (Playwright/Chromium ou o browser da sessão), tirar screenshots de cada página em **desktop (1280px)** e **telemóvel (390px)**, com os estados importantes abertos (menu, modais, separadores, tema escuro).
- Se houver build (`npm run build`, Sass), correr e guardar o resultado.

## FASE 2 — Detetar

Lista inicial opcional com PurgeCSS (só como pista, nunca como prova):

```bash
npx purgecss --css "**/*.css" --content "**/*.html" "**/*.js" --rejected -o /tmp/purge
```

Depois, para cada ficheiro de CSS:

1. **Seletores sem uso**: cada classe `.x` e id `#y` procurado em TODO o markup e JS: `class="... x ..."`, `className`, `classList.add/remove/toggle('x')`, `querySelector('.x')`, `getElementById('y')`, `:class=`, `class:x=`, `clsx(...)`, strings e template strings.
2. **Regras vazias** (`.a { }`).
3. **Propriedade repetida** na mesma regra (fica a última) e **regras duplicadas** (mesmo seletor + mesmas declarações → juntar numa).
4. **Declarações anuladas**: a mesma propriedade redefinida mais abaixo no mesmo seletor e mesma media query, sem nada entre elas que dependa da primeira.
5. **`@keyframes`** sem `animation`/`animation-name` a usá-las.
6. **Variáveis CSS** (`--x`) nunca usadas em `var(--x)` (procurar também no JS: `setProperty('--x')`, `getPropertyValue`).
7. **SCSS/LESS**: `$variaveis`, `@mixin`, `%placeholders` e `@function` nunca usados; partials (`_x.scss`) nunca importados.
8. **`@font-face`** cuja `font-family` não aparece em lado nenhum; `@import` de ficheiros que não existem.
9. **Ficheiros órfãos**: `.css`, fontes e imagens que só eram usados por CSS e que nenhum `href`, `src`, `url()`, `@import` ou `srcset` referencia.
10. **Código comentado** (`/* .antigo { ... } */`) — não os comentários que explicam.

### Nunca marcar como morto

- Classes/ids montados dinamicamente (`'btn-' + tipo`, `` `tema-${x}` ``) — se o prefixo aparece em JS, ficam todas as classes com esse prefixo
- Classes de estado que o JS liga e desliga (`.active`, `.open`, `.show`, `.hidden`, `.is-*`, `.has-*`, `.dark`, `.loading`)
- Pseudo-classes e pseudo-elementos (`:hover`, `:focus-visible`, `::before`, `::selection`, `::-webkit-scrollbar`)
- Regras dentro de `@media`, `@supports`, `@container`, `prefers-color-scheme`, `prefers-reduced-motion`, `@media print`
- Seletores de elemento e atributo (`body`, `a`, `input[type="checkbox"]`, `[data-theme="dark"]`, `[aria-expanded="true"]`)
- Classes de bibliotecas (Bootstrap, Tailwind, Font Awesome, etc.) e CSS que estiliza HTML gerado por bibliotecas ou CMS
- Ids usados como âncoras (`href="#y"`) e por `for=` de labels
- Classes usadas em ficheiros que não estás a ver (outro repositório, servidor, emails HTML) — se o projeto parecer ter isso, pergunta

## FASE 3 — Remover

- Um ficheiro de cada vez. Apagar só o que foi confirmado e limpar linhas em branco a mais.
- Se um seletor faz parte de uma lista (`.a, .b { }`) e só `.b` está morto, apagar só `.b`.
- Ficheiro órfão: `git rm` (ou apagar se não houver git) e remover o `<link>`/`@import` que lhe apontava, se existir.
- Depois de cada ficheiro: correr o build (se houver) e tirar screenshots outra vez. Mudou alguma coisa → repor esse ficheiro (`git checkout -- ficheiro`) e marcar como "saltado".
- Sem browser: verificar que não ficaram `href`/`src`/`url()` partidos e avisar que convém abrir as páginas para confirmar.

## FASE 4 — Commits

- Um commit por ficheiro, só com esse ficheiro: `git add style.css` — nunca `git add -A`.
- Mensagem: `refactor: remove CSS morto de style.css`
- Sem `Co-Authored-By` nem menção ao Claude.

## FASE 5 — Verificação final e resumo

- Build a passar (se houver) e screenshots finais iguais aos iniciais em desktop e telemóvel.
- Entregar os ficheiros alterados completos, prontos a copiar.

```markdown
## CSS morto removido

| # | Ficheiro | O quê | Tipo | Commit |
|---|----------|-------|------|--------|
| 1 | style.css | .card-old | seletor | abc1234 |
| 2 | style.css | @keyframes pulse | keyframes | abc1234 |

### Saltado
| # | Ficheiro | O quê | Porquê |
|---|----------|-------|--------|

### Verificação
- Build: PASSOU/FALHOU/sem build
- Páginas: iguais / por confirmar
- Tamanho do CSS: antes X KB → depois Y KB
- Total: N itens em M ficheiros, K commits
```

Se não houver nada confirmado: dizer "Não encontrei CSS morto" e parar.
