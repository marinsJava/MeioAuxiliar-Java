# 📚 Catálogo de Meios Auxiliares para Programação

Coletânea curada de guias, referências e materiais de apoio para desenvolvimento de software, com foco em **Java**, Spring Boot e boas práticas de engenharia. Cada material é escrito para ser direto ao ponto, com exemplos reais e conexões com o que se usa no dia a dia.

> Use o índice abaixo para navegar. Cada link leva ao guia completo.

---

## ☕ Java

### Padrões de Projeto
| Guia | Descrição |
|---|---|
| [Padrões de Projeto em Java (com ORM)](java/design-patterns/guia-padroes-projeto-java-orm.md) | Os 23 padrões GoF + padrões de persistência (Fowler/PoEAA), cada um conectado a onde aparece no JPA/Hibernate/Spring Data. |

### Spring Security & Autenticação
| Guia | Descrição |
|---|---|
| [Login e Cadastro com OAuth 2.0 / OpenID Connect](java/spring-security/guia-login-cadastro-oauth2-oidc.md) | Autenticação moderna com Spring Boot 4.1 + Spring Security 7: fluxo Authorization Code + PKCE, login social, provisionamento de usuário (JIT) e proteção de API com JWT. |

### Arquitetura & Organização
| Guia | Descrição |
|---|---|
| [Estrutura de Arquivos em Projetos Java](java/arquitetura/guia-estrutura-de-arquivos.md) | Convenções Maven/Gradle, Package by Layer vs. Package by Feature, arquitetura Hexagonal/Clean e nomenclatura. |

### Spring Boot
*(em construção)*

### Clean Code
*(em construção)*

---

## 🐍 Python
*(em construção)*

## 🛠️ Ferramentas
*(Docker, Git, WSL2 — em construção)*

## 💡 Conceitos (agnósticos de linguagem)
*(SOLID, algoritmos, estruturas de dados — em construção)*

---

## Como este catálogo é organizado

- **Por linguagem** no primeiro nível (`java/`, `python/`), porque é como a maioria busca material.
- **Por subtópico** dentro de cada linguagem (`design-patterns/`, `spring-security/`...).
- **Tópicos transversais** (ferramentas, conceitos) têm pastas próprias para não duplicar conteúdo.
- Arquivos em **kebab-case minúsculo, sem acentos** — gera URLs limpas no GitHub.

Cada guia segue um template comum: **título → resumo de uma linha → índice → conteúdo → referências.**

---

## Contribuindo

Sugestões, correções e novos materiais são bem-vindos. Veja [CONTRIBUTING.md](CONTRIBUTING.md).

## Licença

Conteúdo textual sob [Creative Commons Attribution 4.0 (CC BY 4.0)](LICENSE). Os trechos de código nos guias podem ser usados livremente. Veja o arquivo [LICENSE](LICENSE) para detalhes.
