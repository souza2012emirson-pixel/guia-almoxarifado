# Guia de Processos do Almoxarifado — Fiasul

Assistente interativo de procedimentos operacionais do almoxarifado (TOTVS Protheus).
O funcionário escolhe um processo, responde às perguntas e recebe a orientação do próximo passo,
inclusive os caminhos de exceção.

- **Uso interno.** Não há login, banco de dados, integração com o TOTVS nem gravação de dados pessoais.
- **Funciona offline.** O build gera um único arquivo `dist/index.html` (HTML, CSS e JS embutidos).
- **Não substitui** a conferência no TOTVS nem as regras internas autorizadas.

## Executar

```bash
npm install
npm run dev     # desenvolvimento
npm run build   # gera dist/index.html (arquivo único, pronto para uso)
```

## Estrutura

```
src/
├── types.ts                      Formato dos dados (etapas, opções, exceções, notas)
├── engine/
│   ├── flowEngine.ts             Motor: histórico, voltar, trilha, reiniciar e validação do fluxo
│   └── useFlow.ts                Hook React que executa um fluxo
├── data/processes/
│   ├── index.ts                  REGISTRO: lista de processos exibidos na página inicial
│   ├── enderecamento-guarda.ts   Conteúdo do processo "Endereçar e guardar material"
│   └── roadmap.ts                Categorias "Em desenvolvimento" (ainda sem procedimento aprovado)
└── components/                   Interface (não precisa ser alterada para novos processos)
```

**Regra de separação:** o conteúdo (textos, perguntas, ramificações) fica em `src/data/processes/`.
A lógica de navegação fica em `src/engine/`. A interface apenas renderiza o que os dados descrevem.

## Como editar o processo existente

Abra `src/data/processes/enderecamento-guarda.ts`. Cada item de `steps` é um passo com um `id` único.

| Para alterar | Campo |
|---|---|
| Título e texto da etapa | `title`, `body` (instrução), `question` (pergunta) |
| Caminho de menus | `menuPath` |
| Opções de resposta e destino | `options` → `label` e `target` (id do passo de destino) |
| Verificações obrigatórias | `questions` → `text` e `onNo` (destino se a resposta for "Não") |
| Avanço após "Sim, concluído" | `next` |
| Exceção: o que foi identificado / verificar / ação permitida / quando interromper | `identified`, `verify`, `allowed`, `stopWhen` |
| Alertas em destaque | `notes` (`tone`: `info`, `warning`, `danger`, `success`, `validate`) |
| Itens do checklist final | `items` |

Um passo sem `options` do tipo `exception` é uma **parada**: o guia mostra
"Interrompa esta etapa e consulte o responsável pelo almoxarifado" e não avança.

### Marcar o que precisa de validação

Use a nota `tone: "validate"` para qualquer orientação que não conste do procedimento.
Ela aparece com o rótulo **"A validar"** e a mensagem
*"Procedimento a validar com o responsável pelo almoxarifado."*

## Como adicionar um novo processo

1. **Crie o arquivo** `src/data/processes/nome-do-processo.ts`:

   ```ts
   import type { ProcessDefinition } from "../../types";

   export const meuProcesso: ProcessDefinition = {
     id: "meu-processo",
     title: "Título exibido no cartão",
     category: "Categoria",
     description: "Resumo de uma ou duas frases.",
     keywords: ["palavra", "chave", "para busca"],
     status: "disponivel",
     flow: {
       id: "meu-processo",
       title: "Título do fluxo",
       source: "Origem do procedimento e observações de validação.",
       start: "inicio",
       stages: [
         { id: "A", label: "Primeira etapa" },
         { id: "B", label: "Segunda etapa" },
       ],
       steps: [
         {
           id: "inicio",
           kind: "instruction",
           stage: "A",
           title: "Primeiro passo",
           body: ["Faça isto."],
           next: "pergunta",
         },
         {
           id: "pergunta",
           kind: "question",
           stage: "B",
           title: "Verificação",
           question: "Isto está correto?",
           options: [
             { label: "Sim", target: "fim", tone: "primary" },
             { label: "Não", target: "parada", tone: "danger" },
           ],
         },
         {
           id: "parada",
           kind: "exception",
           stage: "B",
           title: "Problema encontrado",
           identified: "O que aconteceu.",
           verify: ["O que conferir."],
           allowed: ["O que o procedimento permite."],
           stopWhen: "Quando interromper.",
         },
         {
           id: "fim",
           kind: "end",
           stage: "B",
           title: "Encerramento",
           body: ["Processo concluído."],
         },
       ],
     },
   };
   ```

2. **Registre o processo** em `src/data/processes/index.ts`:

   ```ts
   import { meuProcesso } from "./meu-processo";
   export const processes: ProcessDefinition[] = [enderecamentoGuarda, meuProcesso, ...roadmapProcesses];
   ```

3. Para liberar uma categoria da lista "Em desenvolvimento", substitua a entrada em `roadmap.ts`
   por um processo completo com `status: "disponivel"` e `flow`.

### Tipos de passo disponíveis

| `kind` | Uso |
|---|---|
| `instruction` | Ação a executar. Botão "Sim, concluído" (ou `nextLabel`). Aceita `menuPath`. |
| `question` | Pergunta com uma opção por destino. |
| `multi-question` | Verificações obrigatórias. Qualquer "Não" desvia; o avanço exige todos "Sim". |
| `exception` | Exceção com as 4 partes obrigatórias. Sem `options` = parada. |
| `checklist` | Lista de itens. Só libera o avanço com todos marcados. |
| `end` | Encerramento com resumo do percurso. |

### Validação automática

Ao abrir um processo, o navegador (console de desenvolvimento) lista problemas do fluxo:
destinos inexistentes, ids duplicados, etapas inexistentes e passos inalcançáveis.

## Fonte e pendências

O conteúdo do processo de endereçamento foi organizado a partir do procedimento
"PASSO A PASSO PARA GUARDAR PEÇA NO TOTVS". Antes da publicação, confira o texto com o PDF original
e com o responsável pelo almoxarifado. Os itens marcados como **"A validar"** precisam de confirmação,
por não constarem do procedimento:

- Regras de formação do código do box e padrões válidos por armazém.
- Campos adicionais e mensagens das telas de cadastro de endereço.
- Tratamento de avisos específicos do sistema.
- Forma de guarda física do material.
- Tratamento de indicador diferente de verde e vermelho.
