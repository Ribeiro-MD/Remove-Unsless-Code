# Remove-Unsless-Code

Skills para o teu agente de IA remover código não usado dos teus projetos.

## Skills

| Skill | Stack | Como funciona |
|-------|-------|---------------|
| [`removeUnslessCode-Java-HTML-CSS`](skills/removeUnslessCode-Java-HTML-CSS/SKILL.md) | Java (Fabric/Paper, Gradle/Maven) + HTML/CSS | Um só agente, verifica pelo compilador e por screenshots antes/depois |
| [`removeUnslessCode-TypeScript`](skills/removeUnslessCode-TypeScript/SKILL.md) | TypeScript/Bun | Original: vários agentes em paralelo com verificação por LSP |

## Instalar

**Claude Code** (Linux/macOS):

```bash
git clone https://github.com/Ribeiro-MD/Remove-Unsless-Code.git
mkdir -p ~/.claude/skills
cp -r Remove-Unsless-Code/skills/* ~/.claude/skills/
```

**claude.ai**: abre o `SKILL.md` que queres e adiciona-o como skill nas definições.

## Usar

Pede ao agente algo como "remove o código morto deste projeto" ou "limpa o CSS que não é usado". Podes limitar o âmbito a um ficheiro, uma pasta ou um nome.
