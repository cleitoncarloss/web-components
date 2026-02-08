# Capitulo 2: Shadow DOM em Profundidade

## O que e Shadow DOM?

Shadow DOM e um mecanismo que permite anexar uma arvore DOM oculta a um elemento. Essa arvore e completamente isolada do DOM principal -- estilos nao vazam para fora, e estilos externos nao penetram.

### Terminologia

- **Shadow Host**: O elemento regular do DOM ao qual o Shadow DOM e anexado
- **Shadow Root**: O no raiz da arvore shadow
- **Shadow Tree**: A arvore DOM dentro do Shadow Root
- **Shadow Boundary**: A fronteira entre o DOM regular e o Shadow DOM

## Criando Shadow DOM

### Modo Imperativo (JavaScript)

```javascript
class MeuComponente extends HTMLElement {
  constructor() {
    super();
    // attachShadow retorna o shadow root
    const shadow = this.attachShadow({ mode: 'open' });

    // A partir daqui, manipulamos o shadow root
    shadow.innerHTML = `
      <style>
        /* Esses estilos SO afetam elementos dentro deste shadow DOM */
        p { color: red; font-weight: bold; }
        .container { padding: 16px; border: 2px solid blue; }
      </style>
      <div class="container">
        <p>Este texto vermelho esta encapsulado!</p>
      </div>
    `;
  }
}

customElements.define('meu-componente', MeuComponente);
```

```html
<!-- O <p> da pagina NAO fica vermelho -->
<p>Este paragrafo nao e afetado pelo estilo do componente</p>
<meu-componente></meu-componente>
```

### Modo Declarativo (HTML -- para SSR)

```html
<meu-widget>
  <template shadowrootmode="open">
    <style>
      p { color: green; }
    </style>
    <p>Renderiza sem JavaScript!</p>
    <slot></slot>
  </template>
  <span>Conteudo projetado</span>
</meu-widget>
```

O Declarative Shadow DOM e parseado pelo navegador durante o HTML parsing, permitindo que o componente renderize antes do JavaScript carregar. Essencial para SSR e SEO.

Atributos adicionais do template:
- `shadowrootclonable` -- permite clonar o shadow root
- `shadowrootdelegatesfocus` -- delega foco para o primeiro elemento focavel interno

## Open vs Closed

| Caracteristica | `mode: "open"` | `mode: "closed"` |
|---|---|---|
| `element.shadowRoot` externo | Retorna o shadow root | Retorna `null` |
| Acesso interno | Total | Total |
| Caso de uso | Maioria dos componentes | Isolamento estrito (widgets embed) |
| Seguranca | Acessivel via JS | NAO e barreira de seguranca real |

```javascript
// Open -- acesso externo
class OpenComponent extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = '<p>Acessivel externamente</p>';
  }
}

// Closed -- sem acesso externo direto
class ClosedComponent extends HTMLElement {
  #shadow; // Referencia privada

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: 'closed' });
    this.#shadow.innerHTML = '<p>Nao acessivel externamente</p>';
  }
}

customElements.define('open-comp', OpenComponent);
customElements.define('closed-comp', ClosedComponent);

// Testando acesso
const open = document.querySelector('open-comp');
console.log(open.shadowRoot); // ShadowRoot {...}

const closed = document.querySelector('closed-comp');
console.log(closed.shadowRoot); // null
```

> **Recomendacao**: Use `mode: "open"` na maioria dos casos. `mode: "closed"` e util
> apenas para widgets embedaveis de terceiros (chat widgets, analytics, etc).

## Encapsulamento de CSS

### Estilos NAO vazam

```javascript
class EstiloIsolado extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `
      <style>
        /* Estas classes so existem dentro deste shadow DOM */
        h1 { color: red; font-size: 3em; }
        .destaque { background: yellow; }
        #unico { border: 5px solid green; }
      </style>
      <h1>Titulo vermelho interno</h1>
      <p class="destaque">Destaque amarelo interno</p>
      <div id="unico">Borda verde interna</div>
    `;
  }
}

customElements.define('estilo-isolado', EstiloIsolado);
```

```html
<!-- Nenhum desses e afetado -->
<h1>Titulo normal da pagina</h1>
<p class="destaque">Sem background amarelo</p>
<div id="unico">Sem borda verde</div>
<estilo-isolado></estilo-isolado>
```

### Estilos da pagina NAO penetram

```html
<style>
  /* Este estilo global NAO afeta elementos dentro do Shadow DOM */
  p { color: purple; font-size: 2em; }
  .container { background: pink; }
</style>

<p>Este fica roxo e grande</p>
<meu-componente></meu-componente>
<!-- O <p> dentro do shadow DOM de meu-componente NAO fica roxo -->
```

### Excecao: CSS Custom Properties (variaveis)

CSS Custom Properties (variaveis) **herdam atraves** da shadow boundary:

```html
<style>
  :root {
    --cor-primaria: #3b82f6;
    --espacamento: 16px;
    --fonte: 'Inter', sans-serif;
  }
</style>
```

```javascript
class ComponenteTematizado extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `
      <style>
        .botao {
          /* Variaveis CSS da pagina funcionam aqui! */
          background: var(--cor-primaria, #333);
          padding: var(--espacamento, 8px);
          font-family: var(--fonte, sans-serif);
          color: white;
          border: none;
          border-radius: 4px;
          cursor: pointer;
        }
      </style>
      <button class="botao"><slot></slot></button>
    `;
  }
}

customElements.define('comp-tema', ComponenteTematizado);
```

## Encapsulamento de JavaScript

O `document.querySelector` NAO encontra elementos dentro do Shadow DOM:

```javascript
// Dado: <meu-comp> com <span class="interno"> dentro do shadow DOM

// NAO encontra
document.querySelector('.interno'); // null
document.querySelectorAll('span'); // Nao inclui o span interno

// Encontra -- precisa acessar via shadowRoot
const comp = document.querySelector('meu-comp');
comp.shadowRoot.querySelector('.interno'); // <span class="interno">
```

### Eventos e Shadow DOM

Eventos nativos (click, input, etc) que se originam dentro do Shadow DOM sao **retargeted** -- o `event.target` e ajustado para o shadow host quando observado de fora:

```javascript
class EventDemo extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `<button id="btn-interno">Clique</button>`;

    shadow.querySelector('#btn-interno').addEventListener('click', (e) => {
      console.log('Dentro do shadow:', e.target); // <button id="btn-interno">
    });
  }
}
customElements.define('event-demo', EventDemo);
```

```javascript
// Fora do shadow DOM
document.querySelector('event-demo').addEventListener('click', (e) => {
  console.log('Fora do shadow:', e.target); // <event-demo> (retargeted!)
  console.log('composedPath:', e.composedPath()); // Caminho completo incluindo internos
});
```

Custom Events com `composed: true` cruzam a shadow boundary:

```javascript
// Dentro do componente
this.dispatchEvent(new CustomEvent('meu-evento', {
  bubbles: true,    // Propaga para cima no DOM
  composed: true,   // Cruza shadow boundaries
  detail: { dados: 'valor' }
}));
```

## Shadow DOM e Foco

```javascript
class FocusDemo extends HTMLElement {
  connectedCallback() {
    // delegatesFocus: true -> quando o host recebe foco,
    // o foco e delegado ao primeiro elemento focavel interno
    const shadow = this.attachShadow({
      mode: 'open',
      delegatesFocus: true
    });

    shadow.innerHTML = `
      <style>
        :host(:focus-within) {
          outline: 2px solid #3b82f6;
          border-radius: 4px;
        }
        input {
          border: 1px solid #ccc;
          padding: 8px;
          border-radius: 4px;
          outline: none;
        }
      </style>
      <label>Nome:</label>
      <input type="text" placeholder="Digite aqui..." />
    `;
  }
}

customElements.define('focus-demo', FocusDemo);
```

## Exercicios

1. Crie um componente `theme-box` que use CSS Custom Properties para permitir customizacao de cores, fontes e espacamentos por quem consome
2. Crie dois componentes: um `open-comp` e um `closed-comp`. Tente acessar o conteudo interno de cada um via JavaScript do console
3. Crie um componente que dispara um Custom Event com `composed: true` e outro com `composed: false`. Observe a diferenca ao escutar o evento fora do componente
