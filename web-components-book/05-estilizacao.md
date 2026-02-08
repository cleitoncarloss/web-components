# Capitulo 5: Estilizacao de Web Components

## Encapsulamento de Estilos

O Shadow DOM fornece encapsulamento completo de CSS -- estilos definidos dentro nao vazam para fora, e estilos externos nao penetram.

### O seletor `:host`

Estiliza o elemento host (o proprio custom element):

```javascript
class StyledCard extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        /* Estiliza o proprio <styled-card> */
        :host {
          display: block;
          padding: 16px;
          border: 1px solid #e0e0e0;
          border-radius: 8px;
          background: white;
        }

        /* Estiliza o host quando tem determinado atributo/classe */
        :host([variant="primary"]) {
          border-color: #3b82f6;
          background: #eff6ff;
        }

        :host([variant="danger"]) {
          border-color: #ef4444;
          background: #fef2f2;
        }

        /* Quando o host tem a classe .featured */
        :host(.featured) {
          box-shadow: 0 4px 12px rgba(0,0,0,0.15);
        }

        /* Quando disabled */
        :host([disabled]) {
          opacity: 0.5;
          pointer-events: none;
        }
      </style>
      <slot></slot>
    `;
  }
}

customElements.define('styled-card', StyledCard);
```

```html
<styled-card variant="primary">Card primario</styled-card>
<styled-card variant="danger">Card de erro</styled-card>
<styled-card class="featured">Card destacado</styled-card>
<styled-card disabled>Card desabilitado</styled-card>
```

### O seletor `:host-context()`

Estiliza o host baseado em um ancestral no DOM:

```css
/* Quando o host esta dentro de um elemento com classe .dark-theme */
:host-context(.dark-theme) {
  background: #1f2937;
  color: white;
  border-color: #374151;
}

/* Quando esta dentro de um <aside> */
:host-context(aside) {
  max-width: 300px;
}

/* Quando esta em um contexto de impressao */
:host-context(.print-mode) {
  box-shadow: none;
  border: 2px solid black;
}
```

> **Nota**: `:host-context()` tem suporte limitado em alguns navegadores. Verifique compatibilidade.

## Estilizando Conteudo Projetado com `::slotted()`

O pseudo-elemento `::slotted()` permite estilizar elementos que foram projetados em um slot (vindos do Light DOM):

```javascript
class ContentBox extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          padding: 20px;
          background: #f9fafb;
        }

        /* Estiliza paragrafos projetados no slot */
        ::slotted(p) {
          margin: 0 0 12px;
          line-height: 1.6;
        }

        /* Estiliza primeiro elemento projetado */
        ::slotted(:first-child) {
          margin-top: 0;
        }

        /* Estiliza elementos com classe .highlight */
        ::slotted(.highlight) {
          background: yellow;
          padding: 2px 4px;
        }

        /* Estiliza headings projetados */
        ::slotted(h1),
        ::slotted(h2),
        ::slotted(h3) {
          color: #1f2937;
          margin: 16px 0 8px;
        }

        /* IMPORTANTE: ::slotted() so funciona com selecao direta */
        /* ::slotted(div p) NAO FUNCIONA */
        /* Use ::slotted(div) para o elemento direto */
      </style>
      <div class="wrapper">
        <slot></slot>
      </div>
    `;
  }
}

customElements.define('content-box', ContentBox);
```

```html
<content-box>
  <h2>Titulo</h2>
  <p>Este paragrafo tera margin e line-height do ::slotted()</p>
  <p class="highlight">Este fica com background amarelo</p>
</content-box>
```

**Limitacoes do `::slotted()`**:
- Apenas seleciona elementos **diretos** projetados no slot
- `::slotted(div p)` NAO funciona -- nao pode selecionar descendentes
- Especificidade mais baixa que estilos globais da pagina

## Exponibilizando Partes com `::part()`

O pseudo-elemento `::part()` permite expor elementos internos do Shadow DOM para estilizacao externa:

```javascript
class CustomButton extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        .button {
          display: inline-flex;
          align-items: center;
          gap: 8px;
          padding: 10px 20px;
          border: none;
          border-radius: 4px;
          background: #3b82f6;
          color: white;
          font-size: 14px;
          cursor: pointer;
          transition: background 0.2s;
        }
        .button:hover {
          background: #2563eb;
        }
        .icon {
          width: 16px;
          height: 16px;
        }
      </style>
      <button class="button" part="button">
        <span class="icon" part="icon">
          <slot name="icon">🔘</slot>
        </span>
        <span part="label">
          <slot></slot>
        </span>
      </button>
    `;
  }
}

customElements.define('custom-button', CustomButton);
```

Estilizacao externa usando `::part()`:

```css
/* Estilizar a parte "button" do custom-button */
custom-button::part(button) {
  background: linear-gradient(45deg, #667eea, #764ba2);
  border-radius: 20px;
  font-weight: bold;
  text-transform: uppercase;
}

custom-button::part(button):hover {
  background: linear-gradient(45deg, #764ba2, #667eea);
}

/* Estilizar a parte "icon" */
custom-button::part(icon) {
  width: 20px;
  height: 20px;
}

/* Estilizar a parte "label" */
custom-button::part(label) {
  letter-spacing: 0.5px;
}

/* Multiplas partes ao mesmo tempo */
custom-button::part(icon label) {
  opacity: 0.9;
}
```

**Multiplas parts no mesmo elemento**:

```html
<span class="title" part="title heading">Titulo</span>
```

```css
/* Estiliza via qualquer uma das parts */
my-component::part(title) { font-size: 1.5em; }
my-component::part(heading) { font-weight: bold; }
```

## CSS Custom Properties (Variaveis)

CSS Custom Properties **atravessam** a shadow boundary, permitindo tematizacao:

```javascript
class ThemedComponent extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          /* Valores padrao caso variaveis nao sejam definidas */
          --component-bg: var(--theme-background, white);
          --component-text: var(--theme-text, black);
          --component-primary: var(--theme-primary, #3b82f6);
          --component-radius: var(--theme-radius, 8px);
          --component-spacing: var(--theme-spacing, 16px);
        }

        .container {
          background: var(--component-bg);
          color: var(--component-text);
          border-radius: var(--component-radius);
          padding: var(--component-spacing);
        }

        .button {
          background: var(--component-primary);
          color: white;
          padding: calc(var(--component-spacing) / 2) var(--component-spacing);
          border: none;
          border-radius: var(--component-radius);
          cursor: pointer;
        }
      </style>
      <div class="container">
        <h3><slot name="title">Titulo</slot></h3>
        <p><slot></slot></p>
        <button class="button">Acao</button>
      </div>
    `;
  }
}

customElements.define('themed-component', ThemedComponent);
```

**Definindo tema globalmente**:

```css
:root {
  --theme-background: #f9fafb;
  --theme-text: #1f2937;
  --theme-primary: #3b82f6;
  --theme-radius: 12px;
  --theme-spacing: 20px;
}

/* Tema escuro */
.dark-mode {
  --theme-background: #1f2937;
  --theme-text: #f9fafb;
  --theme-primary: #60a5fa;
}
```

**Customizacao por instancia**:

```css
/* Customizar uma instancia especifica */
themed-component.special {
  --component-primary: #ef4444;
  --component-radius: 20px;
}
```

### Design Tokens com CSS Custom Properties

```javascript
class DesignSystemButton extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          /* API de estilizacao documentada */
          --btn-bg: var(--ds-color-primary, #3b82f6);
          --btn-bg-hover: var(--ds-color-primary-dark, #2563eb);
          --btn-text: var(--ds-color-white, white);
          --btn-padding-y: var(--ds-spacing-2, 10px);
          --btn-padding-x: var(--ds-spacing-4, 20px);
          --btn-radius: var(--ds-radius-md, 6px);
          --btn-font-size: var(--ds-font-size-base, 14px);
          --btn-font-weight: var(--ds-font-weight-medium, 500);
          --btn-transition: var(--ds-transition-fast, 0.15s ease);

          display: inline-block;
        }

        button {
          background: var(--btn-bg);
          color: var(--btn-text);
          padding: var(--btn-padding-y) var(--btn-padding-x);
          border: none;
          border-radius: var(--btn-radius);
          font-size: var(--btn-font-size);
          font-weight: var(--btn-font-weight);
          cursor: pointer;
          transition: var(--btn-transition);
        }

        button:hover {
          background: var(--btn-bg-hover);
        }

        :host([size="small"]) button {
          --btn-padding-y: var(--ds-spacing-1, 6px);
          --btn-padding-x: var(--ds-spacing-3, 12px);
          --btn-font-size: var(--ds-font-size-sm, 12px);
        }

        :host([size="large"]) button {
          --btn-padding-y: var(--ds-spacing-3, 14px);
          --btn-padding-x: var(--ds-spacing-5, 28px);
          --btn-font-size: var(--ds-font-size-lg, 16px);
        }

        :host([variant="secondary"]) button {
          --btn-bg: transparent;
          --btn-text: var(--ds-color-primary, #3b82f6);
          border: 1px solid var(--ds-color-primary, #3b82f6);
        }
      </style>
      <button><slot></slot></button>
    `;
  }
}

customElements.define('ds-button', DesignSystemButton);
```

## Constructable Stylesheets

Constructable Stylesheets permitem criar folhas de estilo via JavaScript e reutiliza-las em multiplos shadow roots:

```javascript
// Criar stylesheet compartilhada UMA vez
const sharedStyles = new CSSStyleSheet();
sharedStyles.replaceSync(`
  :host {
    font-family: system-ui, -apple-system, sans-serif;
    box-sizing: border-box;
  }
  *, *::before, *::after {
    box-sizing: inherit;
  }
  .flex { display: flex; }
  .gap-2 { gap: 8px; }
  .gap-4 { gap: 16px; }
  .p-4 { padding: 16px; }
  .rounded { border-radius: 8px; }
  .shadow { box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
`);

const buttonStyles = new CSSStyleSheet();
buttonStyles.replaceSync(`
  button {
    padding: 8px 16px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
    transition: all 0.2s;
  }
  button.primary {
    background: #3b82f6;
    color: white;
  }
  button.primary:hover {
    background: #2563eb;
  }
  button.secondary {
    background: #e5e7eb;
    color: #1f2937;
  }
  button.secondary:hover {
    background: #d1d5db;
  }
`);

// Reutilizar em multiplos componentes
class ComponentA extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });

    // Adotar estilos compartilhados
    shadow.adoptedStyleSheets = [sharedStyles, buttonStyles];

    shadow.innerHTML = `
      <div class="p-4 rounded shadow">
        <button class="primary">Component A</button>
      </div>
    `;
  }
}

class ComponentB extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });

    // Reutilizar os mesmos estilos -- zero custo de parse!
    shadow.adoptedStyleSheets = [sharedStyles, buttonStyles];

    // Adicionar estilos locais especificos
    const localStyles = new CSSStyleSheet();
    localStyles.replaceSync(`
      .special { background: linear-gradient(45deg, #667eea, #764ba2); }
    `);

    shadow.adoptedStyleSheets = [sharedStyles, buttonStyles, localStyles];

    shadow.innerHTML = `
      <div class="p-4 rounded shadow special">
        <button class="secondary">Component B</button>
      </div>
    `;
  }
}

customElements.define('component-a', ComponentA);
customElements.define('component-b', ComponentB);
```

**Vantagens**:
- Stylesheet parseada **uma vez** e compartilhada -- economia de memoria
- Atualizacoes dinamicas propagam para todos os consumidores
- Performance superior ao `<style>` em linha

### Atualizacao Dinamica

```javascript
const themeStyles = new CSSStyleSheet();
themeStyles.replaceSync(`
  :host { --color: blue; }
`);

class ThemedElement extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.adoptedStyleSheets = [themeStyles];
    shadow.innerHTML = `
      <style>p { color: var(--color); }</style>
      <p>Texto colorido</p>
    `;
  }
}

customElements.define('themed-element', ThemedElement);

// Alterar tema dinamicamente -- todos os elementos atualizam!
function setTheme(color) {
  themeStyles.replaceSync(`:host { --color: ${color}; }`);
}

setTheme('red');   // Todos os themed-element ficam vermelhos
setTheme('green'); // Todos ficam verdes
```

## Estrategias de Estilizacao

### 1. Encapsulamento Total

Estilos completamente internos, sem customizacao externa:

```javascript
class ClosedStyleComponent extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'closed' });
    // Estilos totalmente isolados
  }
}
```

**Quando usar**: Widgets embedaveis de terceiros (chat, analytics)

### 2. API via CSS Custom Properties

Expor variaveis para customizacao:

```css
:host {
  --component-color: blue;
  --component-size: 16px;
}
```

**Quando usar**: Design systems, componentes tematizaveis

### 3. API via `::part()`

Expor elementos internos especificos:

```html
<span part="label">Texto</span>
```

**Quando usar**: Quando usuarios precisam de controle granular sobre elementos especificos

### 4. Hibrido

Combinar todas as estrategias:

```javascript
class FlexibleComponent extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          --bg: white;
          --text: black;
        }
        .header { background: var(--bg); color: var(--text); }
      </style>
      <div class="header" part="header">
        <slot name="title"></slot>
      </div>
    `;
  }
}
```

Usuarios podem:
- Customizar via CSS custom properties
- Estilizar partes via `::part()`
- Projetar conteudo via slots

## Exercicios

1. Crie um componente `theme-card` que aceite um tema completo via CSS custom properties (cores, espacamentos, tipografia)
2. Crie um componente `stylable-button` que exponha 3-5 partes (`::part()`) para customizacao externa
3. Crie um design system com 5 componentes compartilhando Constructable Stylesheets
4. Experimente estilizar conteudo projetado com `::slotted()` e observe as limitacoes de especificidade
