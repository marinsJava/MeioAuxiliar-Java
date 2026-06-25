# Estrutura de Arquivos em Projetos Java

> Como organizar pastas, pacotes e arquivos num projeto Java/Spring Boot — das convenções obrigatórias do Maven/Gradle às decisões de arquitetura que separam um projeto que escala de um que vira espaguete.

A organização de arquivos parece detalhe, mas é uma das primeiras coisas que um avaliador (ou um colega futuro) percebe. Boa estrutura comunica intenção: olhando a árvore de pacotes, dá para entender o que o sistema faz antes de ler uma linha de código.

---

## Índice

- [A estrutura padrão Maven/Gradle (não negocie isto)](#a-estrutura-padrão-mavengradle)
- [Pacotes: o grande debate — por camada vs. por funcionalidade](#pacotes-o-grande-debate)
- [Package by Layer](#package-by-layer)
- [Package by Feature](#package-by-feature)
- [Arquiteturas em camadas (Hexagonal / Clean)](#arquiteturas-em-camadas)
- [Onde ficam os arquivos que não são código](#onde-ficam-os-arquivos-que-não-são-código)
- [Convenções de nomenclatura](#convenções-de-nomenclatura)
- [Recomendação prática](#recomendação-prática)

---

## A estrutura padrão Maven/Gradle

Isto é **convenção, não configuração**. Tanto Maven quanto Gradle assumem essa estrutura por padrão; fugir dela só traz dor. Memorize:

```
meu-projeto/
├── pom.xml                  (ou build.gradle)
├── src/
│   ├── main/
│   │   ├── java/            ← código-fonte da aplicação
│   │   │   └── com/empresa/projeto/...
│   │   └── resources/       ← arquivos não-Java (config, templates, SQL)
│   │       ├── application.yml
│   │       ├── static/      ← CSS, JS, imagens (apps web)
│   │       └── templates/   ← Thymeleaf, etc.
│   └── test/
│       ├── java/            ← testes (espelham a estrutura de main/java)
│       │   └── com/empresa/projeto/...
│       └── resources/       ← fixtures, configs de teste
└── target/                  (ou build/) ← gerado; vai pro .gitignore
```

Dois princípios que sempre valem:

1. **`src/test/java` espelha `src/main/java`.** O teste de `com.empresa.projeto.service.PedidoService` fica em `com.empresa.projeto.service.PedidoServiceTest`. Isso permite testar membros *package-private* e mantém teste perto do que testa.
2. **`target/` (Maven) ou `build/` (Gradle) nunca vão para o Git.** São gerados pela build.

---

## Pacotes: o grande debate

Definida a estrutura física, vem a decisão que mais impacta a manutenção: **como organizar os pacotes dentro de `com/empresa/projeto`?** Há duas filosofias, e a escolha entre elas diz muito sobre como o projeto vai envelhecer.

---

## Package by Layer

Organiza por **tipo técnico**: todos os controllers juntos, todos os services juntos, todos os repositories juntos.

```
com/empresa/projeto/
├── controller/
│   ├── MedicamentoController.java
│   ├── PacienteController.java
│   └── ReceitaController.java
├── service/
│   ├── MedicamentoService.java
│   ├── PacienteService.java
│   └── ReceitaService.java
├── repository/
│   ├── MedicamentoRepository.java
│   ├── PacienteRepository.java
│   └── ReceitaRepository.java
├── model/
│   ├── Medicamento.java
│   ├── Paciente.java
│   └── Receita.java
└── dto/
    └── ...
```

**Prós:** familiar, simples para projetos pequenos, é o que a maioria dos tutoriais ensina.

**Contras:** uma única funcionalidade fica **espalhada** por vários pacotes. Mexer em "medicamento" obriga você a abrir 4 ou 5 pastas. Conforme o projeto cresce, cada pasta vira um depósito gigante e o acoplamento entre features fica invisível.

> 👍 Boa escolha para: projetos pequenos, estudos, provas de conceito, e o típico projeto acadêmico de cadeira.

---

## Package by Feature

Organiza por **domínio/funcionalidade**: tudo de "medicamento" fica junto, independente do tipo técnico.

```
com/empresa/projeto/
├── medicamento/
│   ├── MedicamentoController.java
│   ├── MedicamentoService.java
│   ├── MedicamentoRepository.java
│   ├── Medicamento.java
│   └── MedicamentoDTO.java
├── paciente/
│   ├── PacienteController.java
│   ├── PacienteService.java
│   ├── PacienteRepository.java
│   └── Paciente.java
├── receita/
│   └── ...
└── shared/              ← código realmente comum (config, exceções, utils)
    ├── config/
    └── exception/
```

**Prós:** alta coesão (o que muda junto, fica junto), baixo acoplamento entre features, fácil de navegar, fácil de extrair um módulo no futuro (rumo a microsserviços ou Spring Modulith). Você pode tornar classes *package-private* e impedir que outra feature dependa de detalhes internos.

**Contras:** exige um pouco mais de disciplina para decidir o que é "compartilhado" de verdade (cuidado com o pacote `shared` virando lixeira).

> 👍 Boa escolha para: praticamente todo projeto que pretende crescer. É a recomendação moderna dominante.

---

## Arquiteturas em camadas

Para sistemas maiores, vale conhecer abordagens que impõem direção de dependência. Não use isso num CRUD simples — é peso morto. Mas reconheça os nomes:

### Arquitetura Hexagonal (Ports & Adapters)

A ideia central: o **domínio** (regras de negócio) fica no centro e **não depende de nada externo** — nem de framework, nem de banco, nem de web. As tecnologias externas se conectam ao domínio através de *ports* (interfaces) e *adapters* (implementações).

```
com/empresa/projeto/medicamento/
├── domain/                  ← regras puras, sem Spring, sem JPA
│   ├── Medicamento.java
│   └── MedicamentoService.java
├── application/             ← casos de uso, orquestração
│   └── port/
│       ├── in/              ← o que o mundo pode pedir ao domínio
│       └── out/             ← o que o domínio precisa do mundo (ex.: salvar)
└── infrastructure/          ← adapters concretos
    ├── web/                 ← controllers REST (adapter de entrada)
    └── persistence/         ← JPA repositories (adapter de saída)
```

A regra de ouro: **as setas de dependência apontam para dentro.** A infraestrutura conhece o domínio, nunca o contrário. Isso permite trocar banco, framework ou camada web sem tocar nas regras de negócio.

### Clean Architecture

Mesma filosofia da Hexagonal, com nomenclatura de camadas concêntricas (Entities → Use Cases → Interface Adapters → Frameworks). Na prática, no mundo Java, as duas convergem muito.

> ⚠️ Nota honesta: Hexagonal/Clean adicionam muitas interfaces e indireção. Para um sistema pequeno ou médio, **Package by Feature simples costuma ser a escolha mais sábia.** Aplique arquiteturas elaboradas quando a complexidade do domínio justificar, não por moda.

---

## Onde ficam os arquivos que não são código

```
src/main/resources/
├── application.yml              ← config principal
├── application-dev.yml          ← perfil de desenvolvimento
├── application-prod.yml         ← perfil de produção
├── db/
│   └── migration/               ← scripts Flyway (V1__init.sql, V2__...)
├── static/                      ← assets servidos direto (CSS, JS, imgs)
└── templates/                   ← Thymeleaf, se for app server-side
```

Na raiz do projeto, fora do `src`:

```
meu-projeto/
├── README.md                    ← sempre
├── .gitignore
├── docker-compose.yml           ← banco local, etc.
├── Dockerfile
├── .env.example                 ← modelo de variáveis (o .env real fica no .gitignore!)
└── docs/                        ← documentação, diagramas de arquitetura
```

---

## Convenções de nomenclatura

- **Pacotes:** tudo minúsculo, sem underscore — `com.empresa.projeto.medicamento`.
- **Classes:** `PascalCase` — `MedicamentoService`.
- **Sufixos por papel (consistência ajuda muito):** `...Controller`, `...Service`, `...Repository`, `...DTO`, `...Config`, `...Exception`, `...Mapper`.
- **Testes:** `...Test` (unitário) e `...IT` (integração, convenção do Failsafe).
- **Arquivos de migração Flyway:** `V<versão>__<descrição>.sql` — ex.: `V1__cria_tabela_medicamento.sql`.

---

## Recomendação prática

Para a maioria dos projetos — e quase certamente para os seus de estudo e portfólio:

1. **Siga a estrutura Maven/Gradle padrão** sem exceções.
2. **Use Package by Feature.** É o melhor custo-benefício entre simplicidade e escalabilidade. Comece com `feature/` + um `shared/` enxuto.
3. **Adote Hexagonal/Clean só quando o domínio for complexo** o suficiente para pagar o custo da indireção. Num CRUD, é exagero.
4. **Mantenha consistência de sufixos.** Previsibilidade vale mais que qualquer regra "perfeita".
5. **`README.md`, `.gitignore` e perfis de configuração** desde o primeiro commit.

A melhor estrutura é a que faz um recém-chegado encontrar o que procura sem perguntar. Se você precisa de um mapa para achar onde mora a lógica de "cadastro", a estrutura falhou.
