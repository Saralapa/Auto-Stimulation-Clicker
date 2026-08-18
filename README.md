# Auto Stimulation Clicker

Automação em **UiPath** que joga sozinha o [Stimulation Clicker](https://neal.fun/stimulation-clicker/),
o clicker game do neal.fun. O robô abre o jogo no Chrome e fica em loop infinito
clicando, comprando upgrades, operando a bolsa, alimentando o pet e limpando as
notificações — sem intervenção humana.

## Como funciona

O processo inteiro vive em `Main.xaml`, dentro de um **Application Card** que
abre `https://neal.fun/stimulation-clicker/` no Chrome. Dentro dele há um
`While (True)` com um `Try/Catch` no corpo: se qualquer seletor falhar em uma
volta (elemento ainda não existe, popup na frente, etc.), a exceção é engolida e
a próxima iteração recomeça do zero. Por isso o robô nunca trava — ele
simplesmente tenta de novo.

Cada iteração passa pelos blocos abaixo, sempre intercalando cliques no botão
principal (`Click me`) entre uma etapa e outra:

### 1. Clique principal e upgrades
- Clica no botão `.press-btn` ("Click me").
- Faz `Find Children` na `.main-upgrades` e clica em cada upgrade disponível,
  **pulando o upgrade do oceano** (`Not currentUiElement.Selector.Contains("ocean")`)
  — é o único que atrapalha o resto da automação.
- Coleta as recompensas de `Collect +N stimulation` e clica em `START`.

### 2. Bolsa de valores
A parte mais elaborada. O robô lê a ação atual (`.stock-select`), o preço
(`.last-price`) e a quantidade de ações (`.stock-shares`), e define as faixas de
compra/venda conforme o papel:

| Ação | Compra abaixo de | Vende acima de |
| --- | --- | --- |
| Apple | 260 | 350 |
| Demais | 10.000 | 50.000 |

Os textos vêm sujos da página (`$`, `,`, `x 2`), então são normalizados com
`Replace` antes do `CInt`. Quando o preço está abaixo da faixa de compra ele
compra 3x; quando está acima da faixa de venda, usa um `Repeat Number of Times`
com `CInt(str_shares)` para vender **todas** as ações de uma vez.

### 3. Pet, loot boxes e caixa de entrada
- Alimenta o tamagotchi (`.tamogotchi-wrapper` → botão `Feed`).
- Abre loot boxes (`.loot-container` / `.loot-box-target`).
- Um `Check App State` observa o badge da inbox (`.main-link-badge`); se houver
  e-mails não lidos, abre a caixa e responde.

### 4. Detecção de resposta certa via JavaScript
Para os e-mails com pergunta, o jogo marca a alternativa correta deixando a
fonte em negrito. Como isso não aparece no seletor, a automação usa um
`Inject Js Script` em cada `.question-choice`:

```js
function(element, input) {
    return window.getComputedStyle(element).fontWeight;
}
```

e clica na opção cujo peso é `bold` ou `>= 700`. É o truque central do projeto —
sem ele não dá para saber qual resposta escolher.

### 5. Manutenção
Quando a barra de armazenamento (`.progress-wrapper`) enche, clica em
`Clear 20% of storage` e fecha o modal pelo `svg.close-icon`.

## Pré-requisitos

- **UiPath Studio 26.0** ou superior (projeto criado na `26.0.180.0`)
- Dependências (restauradas automaticamente ao abrir o projeto):
  - `UiPath.System.Activities` 25.10.3
  - `UiPath.UIAutomation.Activities` 25.10.20
- **Google Chrome** com a extensão do UiPath instalada
- Target framework: **Windows**

## Como executar

1. Abra a pasta do projeto no UiPath Studio e deixe restaurar as dependências.
2. Rode `Main.xaml`. O Chrome abre sozinho no jogo.
3. Deixe rodando. Não mexa no mouse nem troque de janela — os cliques são
   feitos na tela real, não em background.

Para parar, use o Stop do Studio (o `While` é um `Interruptible While`, então ele
para de forma limpa no fim da iteração).

## Limitações conhecidas

- Os seletores são acoplados às classes CSS do neal.fun. Se o site mudar o
  layout, os cliques quebram e o `Try/Catch` vai só mascarar a falha em loop.
- As faixas de preço da bolsa estão fixas no código (`valor_compra` /
  `valor_venda`). Ações fora de Apple caem no genérico 10.000/50.000, que pode
  não ser ótimo dependendo do papel.
- O upgrade do oceano é ignorado de propósito e nunca é comprado.
- Só funciona em primeiro plano, no Chrome, em resolução onde os elementos
  fiquem visíveis.
- A automação não tem condição de parada: ela roda até você interromper.
