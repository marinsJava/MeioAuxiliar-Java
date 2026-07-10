# Guia Definitivo de Docker

> Containers do zero ao deploy, com exemplos voltados a aplicações Java/Spring Boot + PostgreSQL.

A ideia que resume o Docker: **"funciona na minha máquina" deixa de ser desculpa.** Um container empacota sua aplicação com tudo que ela precisa para rodar — runtime, bibliotecas, configs — de forma que ela executa igual no seu notebook, no CI e em produção.

---

## Índice

- [Container não é máquina virtual](#container-não-é-máquina-virtual)
- [Os conceitos que você precisa dominar](#os-conceitos-que-você-precisa-dominar)
- [Comandos essenciais](#comandos-essenciais)
- [Dockerfile: empacotando sua aplicação](#dockerfile-empacotando-sua-aplicação)
- [Multi-stage build para Spring Boot](#multi-stage-build-para-spring-boot)
- [Boas práticas de Dockerfile](#boas-práticas-de-dockerfile)
- [Volumes: dados que sobrevivem](#volumes-dados-que-sobrevivem)
- [Redes](#redes)
- [Docker Compose: orquestrando vários containers](#docker-compose)
- [Limpeza: recuperando espaço em disco](#limpeza)
- [Referências](#referências)

---

## Container não é máquina virtual

A confusão mais comum. A diferença é fundamental:

- Uma **VM** virtualiza o *hardware*: cada VM roda um sistema operacional completo por cima de um hypervisor. Pesada (gigabytes), lenta para subir (minutos).
- Um **container** virtualiza o *sistema operacional*: todos os containers compartilham o kernel do host, isolados entre si. Leve (megabytes), sobe em segundos.

```
   MÁQUINAS VIRTUAIS                      CONTAINERS
┌──────┐ ┌──────┐ ┌──────┐         ┌──────┐ ┌──────┐ ┌──────┐
│ App  │ │ App  │ │ App  │         │ App  │ │ App  │ │ App  │
│ Libs │ │ Libs │ │ Libs │         │ Libs │ │ Libs │ │ Libs │
│ S.O. │ │ S.O. │ │ S.O. │         └──────┘ └──────┘ └──────┘
└──────┘ └──────┘ └──────┘         ┌──────────────────────────┐
┌──────────────────────────┐       │      Docker Engine       │
│        Hypervisor        │       ├──────────────────────────┤
├──────────────────────────┤       │   Kernel do S.O. (host)  │
│   Kernel do S.O. (host)  │       ├──────────────────────────┤
├──────────────────────────┤       │        Hardware          │
│        Hardware          │       └──────────────────────────┘
└──────────────────────────┘
```

A consequência prática: você roda dezenas de containers onde caberiam poucas VMs.

---

## Os conceitos que você precisa dominar

| Conceito | O que é | Analogia |
|---|---|---|
| **Imagem** | Modelo imutável e versionado da aplicação (read-only) | A classe |
| **Container** | Instância em execução de uma imagem | O objeto |
| **Dockerfile** | Receita que descreve como construir a imagem | O código-fonte da imagem |
| **Registry** | Repositório de imagens (Docker Hub, GHCR) | O Maven Central, mas de imagens |
| **Volume** | Armazenamento persistente fora do ciclo de vida do container | Um HD externo |
| **Network** | Rede virtual que conecta containers | A LAN dos containers |

A relação imagem ↔ container é exatamente a de classe ↔ objeto: uma imagem gera muitos containers, do mesmo jeito que uma classe gera muitos objetos.

---

## Comandos essenciais

```bash
# IMAGENS
docker pull postgres:18              # baixa imagem do registry
docker images                       # lista imagens locais
docker build -t meu-app:1.0 .       # constrói imagem a partir do Dockerfile
docker rmi meu-app:1.0              # remove imagem

# CONTAINERS
docker run -d --name api -p 8080:8080 meu-app:1.0   # cria e roda (detached)
docker ps                          # containers em execução
docker ps -a                       # todos, inclusive parados
docker logs -f api                 # acompanha logs (follow)
docker exec -it api sh             # abre shell dentro do container
docker stop api                    # para
docker start api                   # inicia novamente
docker rm api                      # remove (precisa estar parado)

# INSPEÇÃO
docker inspect api                 # metadados completos (JSON)
docker stats                       # uso de CPU/memória em tempo real
```

O flag `-p 8080:8080` mapeia **porta do host : porta do container**. Sem isso, a porta do container fica inacessível de fora. O `-d` (detached) roda em segundo plano; sem ele, o terminal fica preso aos logs.

---

## Dockerfile: empacotando sua aplicação

O Dockerfile é uma sequência de instruções. Cada instrução cria uma **camada (layer)** — e isso tem implicação direta em performance de build, como veremos.

```dockerfile
FROM eclipse-temurin:21-jre        # imagem base (JRE Java 21)
WORKDIR /app                       # diretório de trabalho dentro do container
COPY target/app.jar app.jar        # copia o jar para a imagem
EXPOSE 8080                        # documenta a porta (não publica sozinho)
ENTRYPOINT ["java", "-jar", "app.jar"]   # comando ao iniciar o container
```

Instruções principais: `FROM` (base), `WORKDIR` (diretório), `COPY`/`ADD` (arquivos), `RUN` (executa no build), `ENV` (variáveis), `EXPOSE` (documenta porta), `ENTRYPOINT`/`CMD` (comando de inicialização).

> `ENTRYPOINT` vs `CMD`: o `ENTRYPOINT` define o executável fixo; o `CMD` define argumentos padrão substituíveis. Para apps, `ENTRYPOINT` costuma ser a escolha certa.

---

## Multi-stage build para Spring Boot

O Dockerfile acima tem um problema: ele assume que você já buildou o `.jar` antes. O ideal é **buildar dentro do Docker**, mas sem carregar o Maven e o JDK inteiros na imagem final. A solução é o *multi-stage build*: um estágio compila, outro só roda.

```dockerfile
# ===== Estágio 1: BUILD (pesado, descartado depois) =====
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
# Copia só o pom primeiro para cachear as dependências
COPY pom.xml .
RUN mvn dependency:go-offline -B
# Agora o código-fonte
COPY src ./src
RUN mvn clean package -DskipTests

# ===== Estágio 2: RUNTIME (leve, é o que vai pra produção) =====
FROM eclipse-temurin:21-jre AS runtime
WORKDIR /app
# Usuário não-root por segurança
RUN groupadd -r spring && useradd -r -g spring spring
USER spring
# Copia APENAS o jar do estágio de build
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

O ganho é enorme: a imagem final contém só o JRE + seu jar (algumas centenas de MB), não o Maven + JDK + cache de dependências (que poderiam somar mais de 1 GB). O estágio `build` é jogado fora.

> 💡 Truque do cache: copiar o `pom.xml` e rodar `dependency:go-offline` **antes** de copiar o `src` faz o Docker reaproveitar a camada de dependências enquanto só o código muda. Builds ficam muito mais rápidos no dia a dia.

---

## Boas práticas de Dockerfile

1. **Aproveite o cache de camadas.** O Docker reusa camadas que não mudaram. Coloque o que muda pouco (dependências) **antes** do que muda muito (código).

2. **Use `.dockerignore`.** Evita copiar lixo para o contexto de build:
   ```
   target/
   .git/
   .idea/
   *.md
   .env
   ```

3. **Rode como usuário não-root.** Container rodando como root é risco de segurança. Crie um usuário dedicado (como no exemplo acima).

4. **Fixe versões (tags específicas).** Use `postgres:18` ou `eclipse-temurin:21-jre`, nunca `latest` — `latest` quebra builds de forma imprevisível.

5. **Prefira imagens base enxutas.** `-jre` em vez de `-jdk` no runtime; considere imagens *distroless* ou *alpine* para reduzir superfície de ataque. (Cuidado: Alpine usa `musl libc`, que ocasionalmente causa surpresas com Java.)

6. **Uma responsabilidade por container.** Não rode app + banco no mesmo container. Separe e conecte via Compose/rede.

---

## Volumes: dados que sobrevivem

Containers são **efêmeros**: quando você remove um, os dados internos somem. Para um banco de dados, isso seria catástrofe. Volumes resolvem isso, guardando dados fora do container.

```bash
# Volume nomeado (gerenciado pelo Docker — recomendado para bancos)
docker volume create pgdata
docker run -d --name pg \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:18

# Bind mount (mapeia uma pasta do host — útil em desenvolvimento)
docker run -d -v $(pwd)/config:/app/config meu-app:1.0
```

Diferença prática: **volume nomeado** é gerenciado pelo Docker e portável (ideal para dados de banco); **bind mount** aponta para uma pasta específica do host (ideal para injetar configs ou ver código em dev).

---

## Redes

Containers na mesma rede se enxergam **pelo nome**, sem precisar de IP. Isso é o que faz sua API achar o banco.

```bash
docker network create minha-rede
docker run -d --name pg --network minha-rede postgres:18
docker run -d --name api --network minha-rede meu-app:1.0
# Dentro do container "api", o banco é acessível pelo host "pg:5432"
```

Na prática você raramente cria redes à mão — o Docker Compose faz isso automaticamente, como veremos.

---

## Docker Compose

Subir vários containers com `docker run` à mão é tedioso e propenso a erro. O **Compose** descreve tudo num único arquivo YAML declarativo. É aqui que sua stack ganha vida.

> Use o Compose V2, invocado como `docker compose` (com espaço). O antigo `docker-compose` (com hífen) é a V1, descontinuada.

`docker-compose.yml` para Spring Boot + PostgreSQL:

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_DB: meuapp
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}      # vem do arquivo .env
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:                              # garante que o banco esteja PRONTO
      test: ["CMD-SHELL", "pg_isready -U app -d meuapp"]
      interval: 5s
      timeout: 3s
      retries: 5

  api:
    build: .                                  # constrói a partir do Dockerfile local
    depends_on:
      db:
        condition: service_healthy            # só sobe a api quando o banco passar no healthcheck
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/meuapp
      SPRING_DATASOURCE_USERNAME: app
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
    ports:
      - "8080:8080"

volumes:
  pgdata:
```

Note a URL do datasource: `jdbc:postgresql://db:5432/meuapp` — o host é `db`, o **nome do serviço**, não `localhost`. O Compose cria a rede e a resolução de nomes para você.

Comandos do Compose:

```bash
docker compose up -d          # sobe tudo em background
docker compose ps             # status dos serviços
docker compose logs -f api    # logs de um serviço
docker compose down           # derruba tudo (mantém volumes)
docker compose down -v        # derruba e APAGA volumes (cuidado!)
docker compose up -d --build  # reconstrói as imagens antes de subir
```

> 🔑 O `healthcheck` + `depends_on: condition: service_healthy` resolve a clássica corrida onde a aplicação Spring tenta conectar antes do Postgres estar pronto e quebra na inicialização. É um detalhe que economiza muita frustração.

---

## Limpeza

Docker acumula imagens, containers parados e volumes órfãos que comem disco. Limpeza periódica:

```bash
docker system df              # mostra quanto espaço o Docker está usando
docker container prune        # remove containers parados
docker image prune            # remove imagens "penduradas" (dangling)
docker image prune -a         # remove TODAS as imagens não usadas
docker volume prune           # remove volumes não usados (cuidado com dados!)
docker system prune -a        # limpeza geral (agressiva)
```

> ⚠️ `docker volume prune` pode apagar dados de banco se o volume não estiver em uso no momento. Tenha certeza do que está removendo.

---

## Referências

1. **Documentação oficial** — https://docs.docker.com/
2. **Dockerfile reference** — https://docs.docker.com/reference/dockerfile/
3. **Compose specification** — https://docs.docker.com/compose/
4. **Spring Boot — Container Images** — https://docs.spring.io/spring-boot/reference/packaging/container-images/index.html (o Spring Boot tem suporte nativo a *buildpacks*, alternativa ao Dockerfile via `mvn spring-boot:build-image`)

> Próximo passo natural: quando vários containers viram muitos containers em produção, com escala, auto-recuperação e balanceamento, entra o **Kubernetes**. Veja o [Guia Definitivo de Kubernetes](../kubernetes/guia-definitivo-kubernetes.md).
