# Guia Definitivo de Postman

> Testando APIs de forma profissional: de uma requisição solta a coleções com variáveis, captura automática de token JWT e testes automatizados. Voltado a quem desenvolve APIs REST em Spring Boot.

Postman é o "cliente HTTP" onde você exercita sua API sem precisar de front-end. Mas ele é muito mais que um "curl com botões": bem usado, vira uma suíte de testes viva, documentação executável e ferramenta de automação. Este guia foca no que realmente muda o seu dia a dia.

---

## Índice

- [Anatomia de uma requisição](#anatomia-de-uma-requisição)
- [Coleções: organizando o caos](#coleções-organizando-o-caos)
- [Variáveis e environments](#variáveis-e-environments)
- [Autenticação: o token no lugar certo](#autenticação)
- [O truque que muda tudo: capturar o JWT automaticamente](#o-truque-que-muda-tudo)
- [Escrevendo testes](#escrevendo-testes)
- [Rodando a coleção inteira (Collection Runner)](#rodando-a-coleção-inteira)
- [Importando do Swagger/OpenAPI](#importando-do-swaggeropenapi)
- [Newman: Postman no CI](#newman-postman-no-ci)
- [Boas práticas](#boas-práticas)
- [Referências](#referências)

---

## Anatomia de uma requisição

Toda requisição tem as mesmas partes, e entender cada uma evita 90% da confusão:

- **Método** — `GET` (buscar), `POST` (criar), `PUT`/`PATCH` (atualizar), `DELETE` (remover).
- **URL** — o endereço, incluindo *path variables* (`/medicamentos/1`) e *query params* (`?dosagem=50mg`).
- **Headers** — metadados da requisição: `Content-Type: application/json`, `Authorization: Bearer ...`.
- **Body** — o corpo, normalmente JSON em `POST`/`PUT`. No Postman: aba **Body → raw → JSON**.

A resposta, por sua vez, traz **status** (200, 404...), **headers** e **body**. O Postman mostra tudo, incluindo o tempo e o tamanho — úteis para perceber lentidão.

---

## Coleções: organizando o caos

Uma **coleção** é um conjunto de requisições salvas e organizadas em pastas. Sem isso, você recria as mesmas chamadas toda hora. A estrutura que funciona bem espelha a sua API:

```
📁 API Medicamentos
├── 📁 Auth
│   ├── POST Cadastro
│   └── POST Login
├── 📁 Medicamentos
│   ├── GET Listar
│   ├── GET Buscar por id
│   ├── POST Criar
│   └── DELETE Remover
└── 📁 Relatórios
    └── GET Consumo mensal
```

A coleção também é onde você configura coisas que valem para **todas** as requisições dentro dela — autenticação e scripts — evitando repetição. É a unidade que você exporta, versiona e compartilha com o time.

---

## Variáveis e environments

Aqui mora o pulo do gato para não virar refém de "localhost" espalhado por toda parte. Uma **variável** é escrita como `{{nome}}` e resolvida em tempo de execução.

O caso clássico é a URL base. Em vez de digitar `http://localhost:8080` em 40 requisições, você usa `{{base_url}}/medicamentos`. E define `base_url` num **environment** (ambiente):

| Environment | `base_url` |
|---|---|
| Local | `http://localhost:8080` |
| Homologação | `https://hml.meuapp.com` |
| Produção | `https://api.meuapp.com` |

Trocar de ambiente vira um clique no seletor — a mesma coleção passa a apontar para outro servidor, sem editar nada. Os níveis de variável mais usados: **environment** (por ambiente) e **collection** (fixas da coleção). Há também as globais (evite; viram bagunça).

> 💡 Guarde segredos (senhas, tokens) em variáveis marcadas como *secret*, e **nunca** commite o environment com valores reais. Exporte um environment de exemplo com os campos vazios, como faz com um `.env.example`.

---

## Autenticação

Para uma API protegida por JWT (como a que você constrói no [guia de JWT](../../java/spring-security/guia-jwt-na-pratica.md)), você precisa mandar `Authorization: Bearer <token>` em cada requisição protegida.

No Postman, use a aba **Authorization → Type: Bearer Token** e coloque `{{token}}` no campo. Configure isso **no nível da coleção**, e todas as requisições herdam automaticamente — você não repete em cada uma. Falta só preencher `{{token}}`... e é aí que entra o melhor recurso do Postman.

---

## O truque que muda tudo

Copiar e colar o token manualmente a cada login é tedioso e erra fácil. A solução: um **script pós-resposta** na requisição de login que **extrai o token e salva na variável automaticamente**. Depois disso, você loga uma vez e todas as outras chamadas já saem autenticadas.

Na requisição `POST Login`, aba **Scripts → Post-response** (nas versões novas; era "Tests" antes):

```javascript
// Confere que o login deu certo
pm.test("Login bem-sucedido", () => {
    pm.response.to.have.status(200);
});

// Extrai o token do corpo da resposta e salva no environment
const body = pm.response.json();
pm.environment.set("token", body.token);

console.log("Token salvo:", body.token);
```

A partir daí o fluxo é: rode `Login` uma vez → o `{{token}}` é preenchido sozinho → todas as requisições protegidas funcionam. Esse único script economiza horas ao longo de um projeto e elimina o erro de "esqueci de atualizar o token expirado".

> O objeto `pm` é a API de scripting do Postman. `pm.response` acessa a resposta; `pm.environment.set(chave, valor)` grava uma variável; `pm.test(nome, fn)` registra um teste. Os scripts rodam em JavaScript.

---

## Escrevendo testes

Além de capturar o token, os scripts pós-resposta transformam o Postman numa suíte de testes. Você valida status, corpo e headers com asserções legíveis:

```javascript
pm.test("Status é 200", () => {
    pm.response.to.have.status(200);
});

pm.test("Retorna uma lista", () => {
    const body = pm.response.json();
    pm.expect(body).to.be.an("array");
});

pm.test("Cada medicamento tem id e nome", () => {
    pm.response.json().forEach(m => {
        pm.expect(m).to.have.property("id");
        pm.expect(m).to.have.property("nome");
    });
});

pm.test("Responde em menos de 500ms", () => {
    pm.expect(pm.response.responseTime).to.be.below(500);
});
```

Os resultados aparecem na aba **Test Results** da resposta, em verde/vermelho. Isso já é teste de integração de verdade contra a sua API rodando.

---

## Rodando a coleção inteira

O **Collection Runner** executa todas as requisições de uma coleção em sequência, na ordem das pastas — ótimo para um *smoke test* do fluxo completo: cadastra → loga → cria → busca → remove. Como o login salva o `{{token}}`, os passos seguintes já saem autenticados na mesma rodada.

Você também pode alimentar o Runner com um arquivo **CSV/JSON de dados** para rodar a mesma requisição com várias entradas (*data-driven testing*) — por exemplo, cadastrar 20 medicamentos de uma planilha.

---

## Importando do Swagger/OpenAPI

Você não precisa criar as requisições à mão. Se a sua API expõe a especificação OpenAPI (via springdoc — veja o [guia de Swagger](../swagger/guia-definitivo-swagger.md)), o Postman **importa** esse arquivo e gera a coleção inteira automaticamente, com todos os endpoints, parâmetros e exemplos.

No Postman: **Import → cole a URL** `http://localhost:8080/v3/api-docs` (ou o arquivo JSON). Pronto — a coleção nasce sincronizada com o código. É a ponte natural entre as duas ferramentas: o Swagger documenta, o Postman testa.

---

## Newman: Postman no CI

O **Newman** é o executor de linha de comando do Postman. Ele roda suas coleções fora da interface — o que permite colocá-las num pipeline de CI/CD e falhar o build se um teste de API quebrar.

```bash
npm install -g newman

newman run minha-colecao.json \
  -e ambiente-local.json \
  --reporters cli,junit
```

Exporte a coleção e o environment como JSON, versione-os no repositório, e o Newman roda os mesmos testes que você faz na interface — agora automatizados. É como seus testes de Postman "graduam" para a esteira de integração contínua.

---

## Boas práticas

1. **Use `{{base_url}}` sempre.** Nunca crave `localhost` nas requisições.
2. **Autenticação no nível da coleção**, não repetida em cada requisição.
3. **Capture o token via script** em vez de colar à mão.
4. **Escreva ao menos um `pm.test` por requisição** — status e formato do corpo.
5. **Não commite environments com segredos**; use um de exemplo com campos vazios.
6. **Versione a coleção** (JSON exportado) junto com o código, para o time compartilhar.
7. **Gere a coleção do OpenAPI** quando possível, para manter sincronia com a API.
8. **Nomeie as requisições com clareza** ("POST Criar medicamento", não "nova req").

---

## Referências

1. **Documentação oficial** — https://learning.postman.com/
2. **Scripting com o objeto `pm`** — https://learning.postman.com/docs/tests-and-scripts/write-scripts/postman-sandbox-api-reference/
3. **Newman (CLI)** — https://github.com/postmanlabs/newman
4. **Importar OpenAPI** — https://learning.postman.com/docs/integrations/available-integrations/working-with-openAPI/

> Companheiros deste guia: o [Guia de Swagger](../swagger/guia-definitivo-swagger.md) (que gera a coleção) e o [Guia de JWT](../../java/spring-security/guia-jwt-na-pratica.md) (a autenticação que você testa aqui).
