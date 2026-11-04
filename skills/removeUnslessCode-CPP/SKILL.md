---
name: removeunslesscode-cpp
description: Remove código morto (não usado) de projetos C++ (CMake, Make, Visual Studio, Arduino/PlatformIO) — includes, variáveis, funções, classes e ficheiros órfãos — com verificação pelo compilador.
---

# Remover código morto — C++

Trabalhas sozinho (sem subagentes). Fases: **detetar → verificar → remover → confirmar → resumo**. Não mudas comportamento, só tiras o que não é usado.

## Regras

- **Na dúvida, fica.** Só se remove o que tem ZERO referências confirmadas em todas as unidades de compilação e em todas as plataformas.
- **O compilador e o linker são a prova.** Depois de remover, tem de compilar E ligar (link) em todas as configurações que o projeto tem.
- **Nada de co-autor.** Commits sem `Co-Authored-By` nem qualquer menção ao Claude.
- **Mais de 50 candidatos → para** e pergunta se é para avançar ou reduzir o âmbito.
- Sem git: faz cópia `*.bak` antes de mexer e no fim entrega os ficheiros completos.

## Âmbito

Argumento: ficheiro → só esse; pasta → só essa; nome de símbolo → só esse. Sem argumento → projeto todo. Ignora `build/`, `out/`, `cmake-build-*/`, `third_party/`, `vendor/`, `external/`, `.pio/`, submódulos.

---

## FASE 1 — Identificar o build e estado inicial

- `CMakeLists.txt` → `cmake -S . -B build && cmake --build build`
- `Makefile` → `make`
- `*.sln` / `*.vcxproj` → `msbuild` (Windows) — se não der para compilar aqui, avisar e trabalhar só com verificação por grep, mais conservador
- `platformio.ini` → `pio run`; `*.ino` → `arduino-cli compile`
- `meson.build` → `meson setup build && ninja -C build`

Ver que configurações existem (Debug/Release, `#ifdef _WIN32`/`__linux__`, opções do CMake). Compilar antes de mexer e guardar os avisos.

## FASE 2 — Detetar

Compilar com avisos de código não usado (só para esta análise, sem gravar no build do projeto):

```bash
# GCC/Clang
cmake -S . -B build-dead -DCMAKE_CXX_FLAGS="-Wall -Wextra -Wunused -Wunused-function -Wunused-variable -Wunused-parameter -Wunreachable-code"
cmake --build build-dead 2>&1 | grep -E "unused|unreachable"
```

Ferramentas extra, se existirem:

```bash
cppcheck --enable=unusedFunction,style --inline-suppr -q src/
clang-tidy src/*.cpp -checks='-*,misc-unused-*,readability-redundant-*,clang-diagnostic-unused-*' -p build-dead
```

Depois, com grep:
- **`#include` não usados**: remover um de cada vez e recompilar (é a única forma segura — um include pode trazer coisas indiretamente).
- **Funções `static` / em namespace anónimo** sem uso no próprio ficheiro.
- **Membros `private`** (campos e métodos) sem uso na classe nem em `friend`s.
- **Funções livres e métodos públicos** sem chamadas em nenhum `.cpp`/`.h`/`.hpp`/`.inl`.
- **Ficheiros órfãos**: `.cpp` que não estão em nenhum `CMakeLists.txt`/`Makefile`/`.vcxproj`, e `.h` que nenhum ficheiro inclui.
- **Código inalcançável e morto**: depois de `return`/`throw`/`exit()`, `#if 0 ... #endif`, `if (false)`.
- **Código comentado** (não os comentários que explicam).
- **Macros `#define`** nunca usadas.

### Nunca marcar como morto

- `main`, `WinMain`, `wmain`, `DllMain`; Arduino `setup()`/`loop()`; ISRs (`ISR(...)`, `IRAM_ATTR`), handlers de interrupções
- Funções `extern "C"`, `__declspec(dllexport)`, `__attribute__((visibility("default")))`, `JNIEXPORT`, símbolos de bibliotecas partilhadas e headers públicos (`include/`)
- Métodos `virtual` / `override`, destrutores, construtores e operadores (podem ser chamados implicitamente)
- Templates e funções `inline`/`constexpr` em headers (podem ser instanciados noutros sítios ou por quem usa a biblioteca)
- Código dentro de `#ifdef` de outra plataforma ou configuração (`_WIN32`, `__APPLE__`, `NDEBUG`, `DEBUG`, opções do CMake) — não está morto, só não compila aqui
- Objetos globais/estáticos cujo construtor regista algo (`static Registrar r(...)`, auto-registo de plugins, testes do GoogleTest/Catch2)
- Funções passadas por ponteiro ou como callback (`signal(...)`, `std::thread(f)`, `qsort`, tabelas de funções, `GetProcAddress`/`dlsym` com o nome em string)
- Qt: `Q_OBJECT`, `slots`, `signals`, `Q_INVOKABLE`, `Q_PROPERTY`, ficheiros `.ui`/`.qrc`; Unreal: `UFUNCTION`, `UPROPERTY`, `GENERATED_BODY`
- Testes e o que eles usam
- Variáveis `volatile` e registos de hardware em embedded

## FASE 3 — Remover

- `#include`: um de cada vez, recompilar a seguir.
- Variável / função / classe / macro / código inalcançável / `#if 0` / código comentado: apagar declaração E definição (`.h` e `.cpp`) e limpar linhas em branco a mais.
- Parâmetro não usado num método que sobrepõe outro ou num callback: comentar o nome (`void f(int /*x*/)`) ou `[[maybe_unused]]` — não apagar.
- Parâmetro não usado de função interna: só apagar se atualizares TODAS as chamadas no mesmo passo.
- Ficheiro órfão: `git rm` e tirá-lo do `CMakeLists.txt`/`Makefile`/`.vcxproj` se lá estiver.

Depois de cada ficheiro: compilar e ligar. Falhou → `git checkout -- ficheiros` e marcar como "saltado".

## FASE 4 — Commits

- Um commit por ficheiro (ou par `.h`/`.cpp`), só com esses ficheiros: `git add a.h a.cpp` — nunca `git add -A`.
- Mensagem: `refactor: remove código morto de a.cpp`
- Sem `Co-Authored-By` nem menção ao Claude.

## FASE 5 — Verificação final e resumo

- Build limpo (`rm -rf build && cmake ... && cmake --build build`) em Debug e Release, e correr os testes (`ctest --test-dir build`) se houver.
- Apagar a pasta `build-dead/` usada na análise.
- Entregar os ficheiros alterados completos, prontos a copiar.

```markdown
## Código morto removido

| # | Ficheiro | O quê | Tipo | Commit |
|---|----------|-------|------|--------|
| 1 | utils.cpp | #include <map> | include | abc1234 |
| 2 | player.h / player.cpp | Player::oldMove() | método | def5678 |

### Saltado
| # | Ficheiro | O quê | Porquê |
|---|----------|-------|--------|

### Verificação
- Build Debug: PASSOU/FALHOU · Release: PASSOU/FALHOU
- Testes: X a passar, Y a falhar (Z já falhavam)
- Plataformas não testadas: (ex.: Windows — só verificado por grep)
- Total: N itens em M ficheiros, K commits
```

Se não houver nada confirmado: dizer "Não encontrei código morto" e parar.
