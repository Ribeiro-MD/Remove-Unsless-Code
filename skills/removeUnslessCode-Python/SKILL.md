---
name: removeunslesscode-python
description: Remove código morto (não usado) de projetos Python — imports, variáveis, funções, classes e ficheiros órfãos — com verificação antes de apagar.
---

# Remover código morto — Python

Trabalhas sozinho (sem subagentes). Fases: **detetar → verificar → remover → confirmar → resumo**. Não mudas comportamento, só tiras o que não é usado.

## Regras

- **Na dúvida, fica.** Python é dinâmico: só se remove o que tem ZERO referências confirmadas, incluindo em strings.
- **Os testes / o arranque são a prova.** Depois de remover, tem de compilar, passar no linter e arrancar.
- **Nada de co-autor.** Commits sem `Co-Authored-By` nem qualquer menção ao Claude.
- **Mais de 50 candidatos → para** e pergunta se é para avançar ou reduzir o âmbito.
- Sem git: faz cópia `*.bak` antes de mexer e no fim entrega os ficheiros completos.

## Âmbito

Argumento: ficheiro → só esse; pasta → só essa; nome de símbolo → só esse. Sem argumento → projeto todo (ignora `venv/`, `.venv/`, `__pycache__/`, `build/`, `dist/`, `site-packages/`).

---

## FASE 1 — Estado inicial

Antes de mexer, corre e guarda o resultado:

```bash
python -m compileall -q .
pytest -q            # se houver testes
```

Avisos e falhas que já existiam não são culpa tua.

## FASE 2 — Detetar

Ferramentas (instala se faltar: `pip install vulture ruff`):

```bash
ruff check . --select F401,F811,F841,ERA001 --output-format concise
vulture . --min-confidence 80 --exclude venv,.venv,build,dist
```

- `F401` import não usado · `F811` redefinido sem uso · `F841` variável local nunca lida · `ERA001` código comentado
- `vulture` encontra funções, classes, métodos, atributos e variáveis sem uso. Abaixo de 80% de confiança, só com verificação manual muito cuidadosa.

Depois, com grep:
- **Ficheiros órfãos**: módulos `.py` que nenhum outro importa (`import x`, `from x import`, `from . import x`) nem aparecem em `pyproject.toml`, `setup.py`, `setup.cfg`, `Procfile`, `Dockerfile`, scripts de shell ou CI.
- **Código inalcançável**: depois de `return`/`raise`/`sys.exit()`, `if False:`, `if DEBUG:` com `DEBUG = False` fixo.
- **Funções definidas duas vezes** no mesmo ficheiro (fica a última).

### Verificar cada candidato

Procurar o nome em TODO o projeto, incluindo strings: `grep -rn "nome" --include=*.py --include=*.toml --include=*.cfg --include=*.ini --include=*.yml --include=*.json --include=*.html .`

### Nunca marcar como morto

- `if __name__ == "__main__":`, `main()`, entry points em `[project.scripts]` / `console_scripts` / `[tool.poetry.scripts]`
- `__init__.py` que reexporta, nomes em `__all__`
- Métodos especiais (`__init__`, `__str__`, `__repr__`, `__eq__`, `__enter__`, `__exit__`, etc.) e `@property`/setters
- Tudo registado por decorador: `@app.route`, `@router.get`, `@bot.command`, `@client.event`, `@click.command`, `@pytest.fixture`, `@receiver`, `@celery.task`, `@dataclass` (campos)
- Django: `models.py`, `views.py` ligados em `urls.py`, `admin.py`, `apps.py`, `migrations/`, `settings.py`, campos de modelo, `Meta`, `management/commands/`
- Testes (`test_*.py`, `*_test.py`, `conftest.py`) e o que eles usam
- Nomes usados com `getattr`, `importlib`, `__import__`, `globals()`, `eval`, `pickle`, ou montados em strings (`f"handle_{tipo}"` → ficam todos os `handle_*`)
- Métodos que sobrepõem uma classe base ou implementam um protocolo/ABC
- Callbacks passados como argumento (`button.config(command=...)`, `signal.signal(...)`, `threading.Thread(target=...)`)
- APIs públicas de bibliotecas (o código é usado por outros projetos)

## FASE 3 — Remover

- Import não usado: apagar a linha, ou só esse nome se houver vários (`from x import a, b`).
- Variável local / função / classe / código inalcançável / código comentado: apagar e limpar linhas em branco a mais.
- Parâmetro não usado num callback ou num método que sobrepõe outro: renomear para `_nome` (não apagar).
- Parâmetro não usado de função interna: só apagar se atualizares TODAS as chamadas no mesmo passo.
- Ficheiro órfão: `git rm` (ou apagar se não houver git).

Depois de cada ficheiro:

```bash
python -m py_compile ficheiro.py && ruff check ficheiro.py --select F
```

Falhou → `git checkout -- ficheiro.py` e marcar como "saltado".

## FASE 4 — Commits

- Um commit por ficheiro (ou grupo pequeno), só com esses ficheiros: `git add ficheiro.py` — nunca `git add -A`.
- Mensagem: `refactor: remove código morto de ficheiro.py`
- Sem `Co-Authored-By` nem menção ao Claude.

## FASE 5 — Verificação final e resumo

```bash
python -m compileall -q .
pytest -q                      # comparar com o estado inicial
python -m nome_do_pacote --help   # ou arrancar o script principal, se for seguro
```

Entregar os ficheiros alterados completos, prontos a copiar.

```markdown
## Código morto removido

| # | Ficheiro | O quê | Tipo | Commit |
|---|----------|-------|------|--------|
| 1 | utils.py | import os | import | abc1234 |

### Saltado
| # | Ficheiro | O quê | Porquê |
|---|----------|-------|--------|

### Verificação
- Compilação: PASSOU/FALHOU
- Testes: X a passar, Y a falhar (Z já falhavam)
- Total: N itens em M ficheiros, K commits
```

Se não houver nada confirmado: dizer "Não encontrei código morto" e parar.
