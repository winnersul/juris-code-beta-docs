# Dashboard Eleições — Dudu News

Pasta de trabalho do projeto **Dashboard de Cobertura Eleitoral em Tempo Real**, proposto para a Dudu News (Eleições 2026).

## Conteúdo desta pasta

| Arquivo | O que é |
|---|---|
| `proposta-comercial-dudu-news.pdf` | **Proposta comercial (2 páginas)** — escopo, limitações, arquitetura, ferramentas/custos, cronograma e investimento. Documento para enviar ao cliente. |
| `proposta-comercial-dudu-news.html` | Fonte editável da proposta (abrir no navegador ou reexportar em PDF). |
| `plano-tecnico-editorial-tse.md` | Plano técnico-editorial completo (pesquisa de fontes públicas do TSE, arquitetura detalhada, camada de IA, riscos, cronograma). Base de onde a proposta comercial foi resumida. |
| `design-telas-principais.html` | Protótipo de design das 3 telas principais (Nacional, Estadual, Mobile) — dados fictícios, apenas para demonstrar o layout. Abrir no navegador. |
| `design-telas-preview.png` | Print estático da tela "Painel Nacional" do protótipo, para referência rápida sem precisar abrir o HTML. |

## Versões navegáveis (compartilháveis por link)

- Proposta em formato de plano completo: https://claude.ai/code/artifact/ece621d9-12a0-4517-a22b-272e64499254
- Protótipo das telas (Nacional / Estadual / Mobile): https://claude.ai/code/artifact/3cffba75-81e1-4ed2-9991-e4683ba2a4a7

## Como sincronizar esta pasta com o seu computador (Windows/CMD)

Este projeto vive no repositório GitHub `winnersul/juris-code-beta-docs`, branch `claude/election-coverage-dashboard-vx6mw2`. Para trazer os arquivos para `C:\Users\winne.PC_ANGELO\Desktop\DASHBOARD ELEIÇÕES`, abra o **Prompt de Comando (CMD)** e rode:

**Se a pasta ainda não existe no seu computador (primeira vez):**
```
git clone --branch claude/election-coverage-dashboard-vx6mw2 https://github.com/winnersul/juris-code-beta-docs.git "C:\Users\winne.PC_ANGELO\Desktop\DASHBOARD ELEIÇÕES"
```

**Se a pasta já existe e você só quer trazer as atualizações mais recentes:**
```
cd /d "C:\Users\winne.PC_ANGELO\Desktop\DASHBOARD ELEIÇÕES" && git pull origin claude/election-coverage-dashboard-vx6mw2
```

Depois disso, os arquivos desta pasta (`dashboard-eleicoes-dudu-news/`) estarão dentro da sua pasta local, prontos para abrir, editar ou anexar a um e-mail.

> **Por que preciso rodar esse comando?** Esta sessão do Claude Code roda em um ambiente na nuvem, não diretamente no seu computador — por isso não é possível criar arquivos direto no seu `C:\`. O `git clone`/`git pull` é a forma de trazer o que foi produzido aqui para a sua máquina.
