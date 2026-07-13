# Guia Completo de Git

> Guia de referência em português sobre **Git**: dos conceitos fundamentais aos fluxos de trabalho profissionais. Cada seção combina explicação conceitual, comandos comentados e exemplos práticos do dia a dia de desenvolvimento.

---

## 📑 Índice

1. [O que é Git e como ele pensa](#1-o-que-é-git-e-como-ele-pensa)
2. [Instalação e configuração inicial](#2-instalação-e-configuração-inicial)
3. [As três áreas do Git](#3-as-três-áreas-do-git)
4. [Ciclo básico: init, add, commit](#4-ciclo-básico-init-add-commit)
5. [Explorando o histórico](#5-explorando-o-histórico)
6. [Branches: trabalhando em paralelo](#6-branches-trabalhando-em-paralelo)
7. [Merge e resolução de conflitos](#7-merge-e-resolução-de-conflitos)
8. [Rebase: reescrevendo a linha do tempo](#8-rebase-reescrevendo-a-linha-do-tempo)
9. [Repositórios remotos e GitHub](#9-repositórios-remotos-e-github)
10. [Desfazendo coisas](#10-desfazendo-coisas)
11. [Stash: guardando trabalho temporariamente](#11-stash-guardando-trabalho-temporariamente)
12. [Tags e versionamento](#12-tags-e-versionamento)
13. [.gitignore](#13-gitignore)
14. [Boas práticas de commit (Conventional Commits)](#14-boas-práticas-de-commit-conventional-commits)
15. [Fluxos de trabalho: GitHub Flow e Git Flow](#15-fluxos-de-trabalho-github-flow-e-git-flow)
16. [Pull Requests na prática](#16-pull-requests-na-prática)
17. [Comandos de inspeção e diagnóstico](#17-comandos-de-inspeção-e-diagnóstico)
18. [Situações de emergência (troubleshooting)](#18-situações-de-emergência-troubleshooting)
19. [Cheat sheet — referência rápida](#19-cheat-sheet--referência-rápida)

---

## 1. O que é Git e como ele pensa

Git é um **sistema de controle de versão distribuído**. Três palavras, três conceitos:

- **Controle de versão** — registra a evolução dos arquivos ao longo do tempo. Cada "foto" do projeto é um **commit**, e você pode voltar a qualquer uma delas;
- **Distribuído** — cada desenvolvedor tem uma **cópia completa** do repositório, com todo o histórico. Você trabalha offline; o GitHub é só um ponto de sincronização, não o "dono" do código;
- **Sistema** — internamente, o Git é um banco de dados de *snapshots*. Cada commit aponta para o estado completo do projeto naquele momento (não para "diferenças") e para o commit anterior, formando um **grafo**.

```
A ← B ← C ← D   (main)
         ↖
           E ← F   (feature/login)
```

Entender que **commits formam um grafo e branches são apenas ponteiros para commits** é o insight que faz todo o resto do Git fazer sentido.

> 💡 **Analogia:** pense em commits como *saves* de um jogo. Branches são marcadores de página apontando para um save. `HEAD` é "onde você está agora".

---

## 2. Instalação e configuração inicial

```bash
# Verificar se o Git está instalado
git --version

# Identidade — vai em TODO commit que você fizer (obrigatório)
git config --global user.name "João Vítor Marins Corrêa"
git config --global user.email "jmarinsti@gmail.com"

# Branch inicial padrão como 'main' (padrão moderno)
git config --global init.defaultBranch main

# Editor padrão para mensagens de commit (opcional)
git config --global core.editor "code --wait"    # VS Code

# Ver todas as configurações ativas
git config --list
```

Os níveis de configuração:

| Nível | Flag | Onde vale | Arquivo |
| ----- | ---- | --------- | ------- |
| Sistema | `--system` | Todos os usuários da máquina | `/etc/gitconfig` |
| Global | `--global` | Todos os seus repositórios | `~/.gitconfig` |
| Local | *(padrão)* | Apenas o repositório atual | `.git/config` |

> 💡 Config local sobrescreve a global. Útil quando você usa um e-mail pessoal em projetos próprios e outro corporativo no trabalho.

---

## 3. As três áreas do Git

O modelo mental mais importante do Git. Todo arquivo transita por três áreas:

```
Working Directory  ──git add──▶  Staging Area (Index)  ──git commit──▶  Repositório (.git)
 (seus arquivos)                  (o que vai no                          (histórico
                                   próximo commit)                        permanente)
```

- **Working Directory** — os arquivos como estão no seu disco, onde você edita;
- **Staging Area (index)** — a "área de preparação": você escolhe *o que* entra no próximo commit. É isso que permite commits pequenos e focados mesmo quando você mexeu em dez arquivos;
- **Repositório** — o histórico imutável de commits dentro da pasta oculta `.git/`.

```bash
git status    # mostra em que área está cada mudança
```

Saída típica e como ler:

```
Changes to be committed:        ← já no staging, entram no próximo commit
	modified:   src/Main.java

Changes not staged for commit:  ← modificados, mas fora do staging
	modified:   pom.xml

Untracked files:                ← arquivos novos que o Git ainda não conhece
	src/NovoService.java
```

---

## 4. Ciclo básico: init, add, commit

```bash
# Criar um repositório novo na pasta atual
git init

# OU clonar um existente
git clone https://github.com/marinsJava/MedLembrete-api.git

# Adicionar mudanças ao staging
git add src/Main.java        # um arquivo específico
git add src/                 # uma pasta inteira
git add .                    # tudo (cuidado: revise antes com git status)
git add -p                   # modo interativo: aprova mudança por mudança 👑

# Commitar o que está no staging
git commit -m "feat: adiciona endpoint de listagem de medicamentos"

# Atalho: add + commit dos arquivos JÁ rastreados (não pega untracked)
git commit -am "fix: corrige validação de dosagem"
```

> 💡 **`git add -p`** (patch) é subestimado: ele mostra cada bloco de mudança e pergunta se entra no commit. É a ferramenta ideal para separar "conserto do bug" de "refatoração que fiz no caminho" em commits distintos.

**Regra de ouro:** um commit = uma mudança lógica. Se a mensagem precisa da palavra "e" ("adiciona X **e** corrige Y"), provavelmente deveriam ser dois commits.

---

## 5. Explorando o histórico

```bash
git log                            # histórico completo, detalhado
git log --oneline                  # uma linha por commit (hash + mensagem)
git log --oneline --graph --all    # desenha o grafo de branches no terminal 👑
git log -p pom.xml                 # histórico COM as mudanças de um arquivo
git log --author="João"            # filtra por autor
git log --since="2 weeks ago"      # filtra por data
git log --grep="JWT"               # busca no texto das mensagens

git show a1b2c3d                   # detalhes completos de um commit
git diff                           # working directory × staging
git diff --staged                  # staging × último commit
git diff main..feature/login       # diferenças entre branches
git blame src/Main.java            # quem alterou cada linha, e em qual commit
```

Sobre **hashes**: cada commit tem um identificador SHA-1 (ex.: `a1b2c3d4...`). Você raramente digita ele inteiro — os 7 primeiros caracteres bastam, e referências relativas ajudam:

| Referência | Significado |
| ---------- | ----------- |
| `HEAD` | O commit onde você está agora |
| `HEAD~1` (ou `HEAD~`) | Um commit antes do atual |
| `HEAD~3` | Três commits antes |
| `main` | O commit para onde a branch main aponta |

---

## 6. Branches: trabalhando em paralelo

Uma branch é apenas um **ponteiro móvel para um commit** — criar uma custa nada (literalmente um arquivo de 41 bytes). É isso que torna o fluxo do Git tão leve: você cria uma branch por tarefa, experimenta à vontade e a `main` fica intocada.

```bash
git branch                         # lista branches locais (* = atual)
git branch -a                      # inclui as remotas

git switch -c feature/jwt-auth     # cria E muda para a nova branch 👑
git switch main                    # volta para a main
git switch -                       # volta para a branch anterior (como cd -)

git branch -d feature/jwt-auth     # deleta branch já mergeada
git branch -D feature/jwt-auth     # força a deleção (descarta commits não mergeados)

git branch -m nome-novo            # renomeia a branch atual
```

> 💡 `git switch` e `git restore` (Git 2.23+) dividiram as responsabilidades do antigo `git checkout`, que fazia coisas demais. O `checkout` ainda funciona, mas os comandos novos são mais claros: `switch` troca de branch, `restore` restaura arquivos.

**Convenção de nomes** (alinhada a times profissionais):

```
feature/cadastro-usuario      → nova funcionalidade
fix/token-expirado            → correção de bug
refactor/service-medicamento  → refatoração sem mudar comportamento
docs/readme-endpoints         → documentação
chore/atualiza-dependencias   → manutenção (build, configs)
```

---

## 7. Merge e resolução de conflitos

`git merge` traz os commits de uma branch para outra. Existem dois cenários:

**Fast-forward** — a branch de destino não andou desde que você ramificou. O Git só move o ponteiro para frente, sem criar commit novo:

```
Antes:  main → A ← B ← C ← feature        Depois:  A ← B ← C ← main
```

**Merge de três vias** — as duas branches evoluíram. O Git cria um **commit de merge** com dois pais:

```
A ← B ← C ← ──── M   (main)
     ↖          ↗
       D ← E ──      (feature)
```

```bash
git switch main
git merge feature/jwt-auth
```

### Conflitos

Conflito acontece quando **as duas branches alteraram as mesmas linhas** do mesmo arquivo. O Git para o merge e marca o arquivo:

```java
<<<<<<< HEAD
    private static final int TOKEN_EXPIRACAO_HORAS = 24;
=======
    private static final int TOKEN_EXPIRACAO_HORAS = 12;
>>>>>>> feature/jwt-auth
```

- Entre `<<<<<<< HEAD` e `=======`: a versão da **sua branch atual**;
- Entre `=======` e `>>>>>>>`: a versão da **branch que está entrando**.

Resolvendo:

```bash
# 1. Abra o arquivo, decida qual versão fica (ou combine as duas)
#    e APAGUE os marcadores <<<<<<< ======= >>>>>>>

# 2. Marque como resolvido
git add src/JwtService.java

# 3. Conclua o merge
git commit

# Ou, se quiser desistir e voltar ao estado anterior:
git merge --abort
```

> ⚠️ Conflito **não é erro** — é o Git sendo honesto: "duas pessoas mudaram a mesma coisa e eu não vou escolher por vocês". IDEs como IntelliJ e VS Code têm ótimas ferramentas visuais de resolução.

---

## 8. Rebase: reescrevendo a linha do tempo

`git rebase` alcança o mesmo objetivo do merge (integrar mudanças), mas por outro caminho: em vez de criar um commit de merge, ele **reaplica seus commits em cima da outra branch**, um por um, como se você tivesse começado o trabalho agora:

```
Antes:                              Depois de git rebase main:
A ← B ← C   (main)                  A ← B ← C   (main)
     ↖                                       ↖
       D ← E   (feature)                       D' ← E'   (feature)
```

```bash
git switch feature/jwt-auth
git rebase main            # reaplica D e E sobre C
```

**Merge × Rebase:**

| | Merge | Rebase |
| - | ----- | ------ |
| Histórico | Preserva como aconteceu (com bifurcações) | Linear, como se fosse sequencial |
| Commits | Cria commit de merge | Reescreve commits (novos hashes: D→D') |
| Rastreabilidade | Total | "Limpa" a história real |
| Uso típico | Integrar feature na main | Atualizar sua feature com a main antes do PR |

### Rebase interativo — a faca suíça

```bash
git rebase -i HEAD~3    # abre os 3 últimos commits para edição
```

No editor, você comanda o destino de cada commit:

```
pick   a1b2c3d feat: adiciona endpoint de login
squash f4e5d6c fix: corrige typo no endpoint       ← funde com o de cima
reword 9g8h7i6 feat: adiciona refresh token        ← reescreve a mensagem
```

Perfeito para transformar "wip", "arruma", "agora vai" em um histórico apresentável antes de abrir o Pull Request.

> ⚠️ **A regra de ouro do rebase:** nunca rebaseie commits que **já foram enviados** para uma branch compartilhada (push). Reescrever história pública quebra o repositório dos colegas. Rebase é para trabalho **local** ou branches que só você usa.

---

## 9. Repositórios remotos e GitHub

O remoto é uma cópia do repositório hospedada em outro lugar (GitHub, GitLab...). Por convenção, o principal se chama `origin`.

```bash
git remote -v                                              # lista remotos
git remote add origin git@github.com:marinsJava/repo.git   # vincula um remoto

# Enviar commits
git push origin main
git push -u origin feature/jwt-auth   # -u: vincula a branch local à remota;
                                      # depois disso, só 'git push' basta

# Receber commits
git fetch origin     # baixa as novidades, mas NÃO altera seus arquivos
git pull origin main # fetch + merge: baixa E integra na branch atual
git pull --rebase    # fetch + rebase: integra sem criar commit de merge
```

> 💡 **`fetch` × `pull`:** o `fetch` é o "só olhar": atualiza sua visão do remoto (`origin/main`) sem tocar no seu trabalho. O `pull` é `fetch` + integração imediata. Em dúvida sobre o que vem do remoto? `git fetch` seguido de `git log HEAD..origin/main --oneline` mostra exatamente o que chegaria.

### HTTPS × SSH

- **HTTPS** — pede autenticação por token (PAT); mais simples de começar;
- **SSH** — usa par de chaves; depois de configurado, nunca mais pede senha:

```bash
ssh-keygen -t ed25519 -C "jmarinsti@gmail.com"
cat ~/.ssh/id_ed25519.pub   # copie e cadastre em GitHub → Settings → SSH Keys
ssh -T git@github.com       # testa a conexão
```

---

## 10. Desfazendo coisas

A seção mais consultada de qualquer guia de Git. O comando certo depende de **onde** a mudança está:

### Mudança só no working directory (não deu `add`)

```bash
git restore src/Main.java    # descarta as edições — CUIDADO: irreversível
git restore .                # descarta tudo
```

### Mudança no staging (deu `add`, não deu `commit`)

```bash
git restore --staged src/Main.java   # tira do staging, mantém a edição no arquivo
```

### Já commitou — três níveis de reset

```bash
git reset --soft HEAD~1    # desfaz o commit; mudanças VOLTAM para o staging
git reset --mixed HEAD~1   # desfaz commit e staging; mudanças ficam nos arquivos (padrão)
git reset --hard HEAD~1    # apaga TUDO: commit, staging e arquivos ⚠️ destrutivo
```

> 💡 Mnemônico: **soft** = mexe só no histórico · **mixed** = histórico + staging · **hard** = histórico + staging + seus arquivos.

### Já commitou E já deu push

**Não use reset** — reescreveria história pública. Use `revert`, que cria um **novo commit desfazendo** o anterior:

```bash
git revert a1b2c3d    # gera um commit "anti-a1b2c3d"; histórico preservado
```

### Errou só a mensagem (ou esqueceu um arquivo) no último commit

```bash
git commit --amend -m "mensagem corrigida"
git add ArquivoEsquecido.java && git commit --amend --no-edit
```

⚠️ `--amend` também reescreve o commit — só use antes do push.

### "Perdi um commit!" — o reflog salva

O Git guarda um diário de **todos** os lugares onde o `HEAD` esteve, mesmo após resets:

```bash
git reflog                  # lista tudo: commits "perdidos" aparecem aqui
git reset --hard a1b2c3d    # volta para o estado desejado
```

> 💡 No Git, é **muito** difícil perder trabalho commitado. Se commitou, o reflog encontra (por ~90 dias).

---

## 11. Stash: guardando trabalho temporariamente

Você está no meio de uma feature e precisa trocar de branch **agora** para corrigir um bug urgente. Commitar pela metade? Não — guarde na "gaveta":

```bash
git stash                          # guarda as mudanças e limpa o working directory
git stash push -m "form de login pela metade"   # com descrição (melhor)

git stash list                     # o que há na gaveta
# stash@{0}: On feature/login: form de login pela metade

git stash pop                      # recupera o mais recente E remove da gaveta
git stash apply stash@{1}          # recupera um específico, mantendo na gaveta
git stash drop stash@{0}           # descarta um stash
git stash -u                       # inclui arquivos untracked (novos)
```

Fluxo típico do bug urgente:

```bash
git stash push -m "feature pela metade"
git switch main
git switch -c fix/bug-urgente
# ... corrige, commita, abre PR ...
git switch feature/login
git stash pop        # de volta exatamente de onde parou
```

---

## 12. Tags e versionamento

Tags marcam pontos importantes do histórico — tipicamente **releases**:

```bash
git tag v1.0.0                                # tag leve (só um ponteiro)
git tag -a v1.0.0 -m "Primeira versão estável" # tag anotada (autor, data, mensagem) 👑
git tag -a v0.9.0 a1b2c3d -m "Beta"           # taguear um commit antigo

git tag                     # listar
git push origin v1.0.0      # tags NÃO sobem com push normal — envie explicitamente
git push origin --tags      # envia todas
```

**Versionamento semântico (SemVer)** — o padrão `MAJOR.MINOR.PATCH`:

- **MAJOR** (`2.0.0`) — mudança incompatível (quebra quem usa sua API);
- **MINOR** (`1.3.0`) — funcionalidade nova, retrocompatível;
- **PATCH** (`1.2.4`) — correção de bug, retrocompatível.

> 💡 No GitHub, tags anotadas alimentam a página **Releases** — você pode anexar changelog e binários a cada versão.

---

## 13. .gitignore

O `.gitignore` lista o que o Git deve **fingir que não existe**: artefatos de build, configs de IDE, segredos, dependências baixáveis.

Exemplo para projeto **Java/Spring Boot + Maven**:

```gitignore
# Build
target/
*.jar
*.war

# IDE
.idea/
*.iml
.vscode/
.settings/
.classpath
.project

# Sistema operacional
.DS_Store
Thumbs.db

# Logs e temporários
*.log
*.tmp

# Segredos — NUNCA versionar
.env
application-local.yml
*.pem
```

Regras de sintaxe:

| Padrão | Efeito |
| ------ | ------ |
| `target/` | Ignora a pasta (a barra final indica diretório) |
| `*.log` | Ignora por extensão |
| `!importante.log` | **Exceção**: não ignora este |
| `**/temp` | `temp` em qualquer nível de pasta |
| `/config.yml` | Só na raiz do repositório |

> ⚠️ **Pegadinha clássica:** o `.gitignore` só vale para arquivos **ainda não rastreados**. Se você já commitou o arquivo, adicioná-lo ao ignore não resolve. Remova do índice primeiro (mantendo no disco):
>
> ```bash
> git rm --cached application-local.yml
> git commit -m "chore: remove config local do versionamento"
> ```

> 🔐 **Se um segredo (senha, token, chave) já subiu para o GitHub, considere-o vazado.** Revogue/troque a credencial imediatamente — apagar o arquivo em um commit novo não apaga o histórico.

---

## 14. Boas práticas de commit (Conventional Commits)

Mensagem de commit é documentação. O padrão **Conventional Commits** estrutura assim:

```
<tipo>(<escopo opcional>): <descrição no imperativo>

[corpo opcional: o PORQUÊ da mudança]

[rodapé opcional: BREAKING CHANGE, refs de issue]
```

Tipos principais:

| Tipo | Quando usar |
| ---- | ----------- |
| `feat` | Nova funcionalidade |
| `fix` | Correção de bug |
| `refactor` | Mudança de código sem alterar comportamento |
| `docs` | Só documentação |
| `test` | Adição/ajuste de testes |
| `chore` | Manutenção: build, dependências, configs |
| `perf` | Melhoria de performance |
| `style` | Formatação (espaços, ponto-e-vírgula), sem lógica |

Exemplos reais:

```bash
git commit -m "feat(auth): adiciona refresh token com expiração de 7 dias"
git commit -m "fix(medicamento): impede dosagem com valor negativo"
git commit -m "refactor(posologia): extrai cálculo de horários para Strategy"
git commit -m "docs: adiciona exemplos de requisição no README"
```

Princípios:

- **Imperativo:** "adiciona", não "adicionado" ou "adicionando" (complete a frase: *"se aplicado, este commit vai... adicionar refresh token"*);
- **Linha de título ≤ 72 caracteres**; detalhes vão no corpo;
- **O código mostra O QUE mudou; a mensagem explica POR QUÊ**;
- Bonus: mensagens convencionais permitem gerar **changelogs automáticos** e automatizar versionamento (semantic-release).

---

## 15. Fluxos de trabalho: GitHub Flow e Git Flow

### GitHub Flow — simples e contínuo (recomendado para a maioria)

Uma branch eterna (`main`, sempre estável e implantável) e branches curtas de trabalho:

```
main ────●──────●─────────●────▶  (sempre estável)
          ↘    ↗ ↘       ↗
   feature/x──●    fix/y●
```

1. Crie uma branch a partir da `main`: `git switch -c feature/nova-funcionalidade`;
2. Commits pequenos e frequentes;
3. Push e abertura de **Pull Request**;
4. Revisão de código + CI (testes automáticos) verdes;
5. Merge na `main` e deleção da branch.

### Git Flow — estruturado, para releases versionadas

Duas branches eternas (`main` = produção, `develop` = integração) e três tipos de branches de apoio:

```
main    ────●───────────────●────▶   (só releases: v1.0, v1.1)
             ↖             ↗ ↖
release        ─────●────●     hotfix/●
                   ↗            ↙
develop ──●───●───●───●───●───●──▶   (integração contínua)
            ↘    ↗  ↘    ↗
      feature/a●     feature/b●
```

- `feature/*` — sai da `develop`, volta para a `develop`;
- `release/*` — sai da `develop`; só recebe ajustes finais; ao concluir, merge em `main` **e** `develop`, com tag de versão;
- `hotfix/*` — sai da `main` (bug em produção); merge em `main` **e** `develop`.

**Qual usar?** GitHub Flow para APIs e apps com deploy contínuo (caso da maioria dos projetos modernos e dos seus projetos de portfólio). Git Flow quando há versões formais mantidas em paralelo (ex.: software distribuído a clientes).

---

## 16. Pull Requests na prática

O Pull Request (PR) é o ritual de integração: "meu trabalho está pronto — revisem antes de entrar na main". Fluxo completo:

```bash
# 1. Branch atualizada a partir da main
git switch main && git pull
git switch -c feature/filtro-por-tags

# 2. Trabalhe com commits pequenos
git add -p && git commit -m "feat(cardapio): adiciona filtro por tags"

# 3. Antes de abrir o PR, atualize com a main (evita conflito na revisão)
git fetch origin
git rebase origin/main

# 4. Envie a branch
git push -u origin feature/filtro-por-tags
```

No GitHub, abra o PR com descrição que responda:

- **O que** foi feito e **por quê**;
- **Como testar** (passos, endpoints, payloads);
- Screenshots/evidências quando visual.

Boas práticas de revisão:

- PRs **pequenos** (idealmente < 400 linhas) são revisados melhor e mais rápido;
- Responda todos os comentários — mesmo que seja para discordar com argumentos;
- CI verde antes de pedir revisão;
- Após o merge, delete a branch (o histórico permanece).

> 💡 Mesmo em projetos solo, vale abrir PRs: cria histórico organizado no GitHub, treina o fluxo profissional e é uma vitrine de disciplina para recrutadores.

---

## 17. Comandos de inspeção e diagnóstico

```bash
git status -sb                  # status curto: menos ruído
git log --oneline --graph --all # o grafo completo do repositório
git shortlog -sn                # ranking de commits por autor
git diff main...feature/x       # o que a feature adiciona desde a bifurcação
git show --stat HEAD            # arquivos alterados no último commit
git bisect start                # busca binária pelo commit que introduziu um bug 👑
git clean -n                    # simula remoção de arquivos untracked (-f executa)
git remote show origin          # detalhes do remoto: branches, tracking, estado
```

Sobre o **`git bisect`**, que merece destaque: você informa um commit bom e um ruim, e o Git vai cortando o intervalo pela metade — você testa cada ponto e responde `git bisect good` ou `git bisect bad` até ele apontar o commit exato que quebrou. Em um intervalo de 100 commits, ~7 testes bastam.

---

## 18. Situações de emergência (troubleshooting)

| Situação | Solução |
| -------- | ------- |
| Commitei na branch errada | `git switch branch-certa` → `git cherry-pick <hash>` → volta e `git reset --hard HEAD~1` na errada |
| Preciso de UM commit de outra branch | `git cherry-pick <hash>` |
| Fiz `reset --hard` e me arrependi | `git reflog` → localize o hash → `git reset --hard <hash>` |
| Push rejeitado (*non-fast-forward*) | `git pull --rebase` → resolva conflitos se houver → `git push` |
| Arquivo enorme commitado por engano | Antes do push: `git reset --soft HEAD~1`, ajuste o `.gitignore`, recommite |
| Segredo vazou no histórico | **Revogue a credencial primeiro.** Depois limpe com `git filter-repo` (histórico compartilhado exige força-tarefa do time) |
| `pull` gerou merge indesejado | `git reset --hard ORIG_HEAD` (o Git guarda o estado pré-merge) |
| Quero ver um arquivo de outra branch sem trocar | `git show feature/x:src/Main.java` |
| Working directory bagunçado, quero recomeçar | `git restore .` + `git clean -fd` ⚠️ (irreversível — confira com `git clean -n` antes) |

---

## 19. Cheat sheet — referência rápida

```bash
# CONFIGURAÇÃO
git config --global user.name "Nome"
git config --global user.email "email@exemplo.com"

# BÁSICO
git init                     # novo repositório
git clone <url>              # clonar existente
git status                   # estado atual
git add <arquivo> | -p | .   # preparar mudanças
git commit -m "msg"          # gravar snapshot

# BRANCHES
git switch -c <branch>       # criar e mudar
git switch <branch>          # mudar
git branch -d <branch>       # deletar
git merge <branch>           # integrar (na branch de destino)
git rebase <branch>          # reaplicar commits sobre outra base

# REMOTO
git push -u origin <branch>  # enviar (primeira vez)
git push                     # enviar
git pull                     # baixar e integrar
git fetch                    # baixar sem integrar

# DESFAZER
git restore <arquivo>            # descarta edição local
git restore --staged <arquivo>   # tira do staging
git commit --amend               # conserta último commit (antes do push)
git reset --soft|--mixed|--hard HEAD~1
git revert <hash>                # desfaz commit público com novo commit
git reflog                       # diário de bordo: recupera "perdidos"

# UTILITÁRIOS
git stash / git stash pop    # gaveta temporária
git tag -a v1.0.0 -m "msg"   # marcar release
git log --oneline --graph    # visualizar histórico
git cherry-pick <hash>       # trazer um commit específico
```

---

## 📚 Referências

- [Pro Git Book (gratuito, com tradução PT-BR)](https://git-scm.com/book/pt-br/v2) — a referência definitiva
- [Documentação oficial do Git](https://git-scm.com/docs)
- [Conventional Commits](https://www.conventionalcommits.org/pt-br/)
- [GitHub Docs — Pull Requests](https://docs.github.com/pt/pull-requests)
- [Learn Git Branching (visualizador interativo)](https://learngitbranching.js.org/?locale=pt_BR)
- [Oh Shit, Git!?! (troubleshooting bem-humorado)](https://ohshitgit.com/)
