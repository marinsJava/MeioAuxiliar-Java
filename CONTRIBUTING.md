# Como contribuir

Obrigado pelo interesse em contribuir com o catálogo! Este documento descreve as convenções para manter tudo coeso.

## Adicionando um novo guia

1. **Escolha a pasta certa.** Linguagem no primeiro nível (`java/`, `python/`), subtópico dentro dela. Se for transversal, use `ferramentas/` ou `conceitos/`.

2. **Nomeie o arquivo em kebab-case, minúsculo, sem acentos.**
   - ✅ `guia-tratamento-de-excecoes.md`
   - ❌ `Guia Tratamento de Exceções.md`

3. **Siga o template padrão** para manter consistência visual:

   ```markdown
   # Título do Guia

   > Resumo de uma linha do que o guia cobre.

   ## Índice
   - ...

   ## (conteúdo em seções)

   ## Referências
   1. ...
   ```

4. **Adicione o link no `README.md`**, na tabela da seção correspondente, com uma descrição curta.

## Estilo de conteúdo

- Prefira **exemplos reais** a exemplos abstratos.
- Seja honesto sobre limitações e trade-offs — vale mais que parecer completo.
- Código em blocos com a linguagem indicada (` ```java `).
- Onde citar versões de ferramentas, **indique a data/versão de referência** (elas mudam).

## Imagens e diagramas

- Coloque assets em `assets/` e referencie com caminho relativo.
- Prefira diagramas em texto (Mermaid) quando possível — versionam melhor que imagens binárias.

## Commits

- Mensagens claras e no imperativo: `adiciona guia de tratamento de exceções`.
- Um material por commit/PR quando possível, facilita a revisão.

## Correções

Encontrou um erro? Abra uma *issue* ou um *pull request* direto. Correções pequenas (typos, links quebrados) são sempre bem-vindas.
