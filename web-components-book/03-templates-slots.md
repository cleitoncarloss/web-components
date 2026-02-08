# Capitulo 3: Templates e Slots

## HTML Templates

O elemento `<template>` contem markup que e parseado pelo navegador mas NAO renderizado ate ser clonado e inserido no DOM via JavaScript.

### Basico

```html
<template id="card-template">
  <style>
    .card {
      border: 1px solid #ddd;
      border-radius: 8px;
      padding: 16px;
      margin: 8px;
    }
    .title { font-weight: bold; font-size: 1.2em; }
    .body { color: #555; margin-top: 8px; }
  </style>
  <div class="card">
    <div class="title"></div>
    <div class="body"></div>
  </div>
</template>
```

```javascript
class SimpleCard extends HTMLElement {
  connectedCallback() {
    const template = document.getElementById('card-template');
    const shadow = this.attachShadow({ mode: 'open' });

    // cloneNode(true) = deep clone (inclui filhos)
    const clone = template.content.cloneNode(true);

    // Preencher dados antes de inserir
    clone.querySelector('.title').textContent = this.getAttribute('title') || '';
    clone.querySelector('.body').textContent = this.getAttribute('body') || '';

    shadow.appendChild(clone);
  }
}

customElements.define('simple-card', SimpleCard);
```

### Template embutido no componente

Na pratica, e mais comum definir o template dentro do proprio componente:

```javascript
class InlineCard extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });

    // Template como string -- mais pratico para componentes autocontidos
    const template = document.createElement('template');
    template.innerHTML = `
      <style>
        :host { display: block; }
        .card { border: 1px solid #e0e0e0; border-radius: 8px; padding: 16px; }
      </style>
      <div class="card">
        <slot></slot>
      </div>
    `;

    shadow.appendChild(template.content.cloneNode(true));
  }
}

customElements.define('inline-card', InlineCard);
```

### Template reutilizavel (performance)

Para componentes instanciados muitas vezes, criar o template uma unica vez:

```javascript
// Template criado UMA vez, fora da classe
const accordionTemplate = document.createElement('template');
accordionTemplate.innerHTML = `
  <style>
    :host { display: block; }
    .header {
      padding: 12px 16px;
      background: #f5f5f5;
      cursor: pointer;
      user-select: none;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    .header:hover { background: #eee; }
    .body {
      padding: 0 16px;
      max-height: 0;
      overflow: hidden;
      transition: max-height 0.3s ease, padding 0.3s ease;
    }
    :host([open]) .body {
      max-height: 500px;
      padding: 16px;
    }
    .arrow {
      transition: transform 0.3s;
    }
    :host([open]) .arrow {
      transform: rotate(90deg);
    }
  </style>
  <div class="header">
    <slot name="header">Clique para expandir</slot>
    <span class="arrow">&#9654;</span>
  </div>
  <div class="body">
    <slot></slot>
  </div>
`;

class AccordionItem extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });
    // Reutiliza o mesmo template -- navegador otimiza o parse
    shadow.appendChild(accordionTemplate.content.cloneNode(true));

    shadow.querySelector('.header').addEventListener('click', () => {
      this.toggleAttribute('open');
    });
  }
}

customElements.define('accordion-item', AccordionItem);
```

```html
<accordion-item>
  <span slot="header">Pergunta frequente #1</span>
  <p>Esta e a resposta detalhada da pergunta.</p>
</accordion-item>

<accordion-item open>
  <span slot="header">Pergunta frequente #2</span>
  <p>Esta ja comeca aberta por causa do atributo "open".</p>
</accordion-item>
```

## Slots

Slots sao pontos de insercao no Shadow DOM onde o conteudo do Light DOM (fornecido pelo consumidor) e projetado.

### Slot padrao (unnamed)

```javascript
class SimpleContainer extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `
      <style>
        .wrapper {
          border: 2px dashed #3b82f6;
          padding: 16px;
          border-radius: 8px;
        }
      </style>
      <div class="wrapper">
        <!-- Todo conteudo sem atributo slot="..." vai aqui -->
        <slot></slot>
      </div>
    `;
  }
}

customElements.define('simple-container', SimpleContainer);
```

```html
<simple-container>
  <p>Este conteudo e projetado para dentro do slot padrao</p>
  <p>Este tambem</p>
</simple-container>
```

### Named Slots

```javascript
class ArticleLayout extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `
      <style>
        :host { display: block; max-width: 800px; }
        header { border-bottom: 2px solid #333; padding-bottom: 12px; }
        .meta { color: #888; font-size: 0.85em; }
        main { padding: 20px 0; line-height: 1.6; }
        footer { border-top: 1px solid #ddd; padding-top: 12px; color: #666; }
      </style>
      <article>
        <header>
          <slot name="titulo">Sem titulo</slot>
          <div class="meta">
            <slot name="meta"></slot>
          </div>
        </header>
        <main>
          <slot></slot>
        </main>
        <footer>
          <slot name="rodape"></slot>
        </footer>
      </article>
    `;
  }
}

customElements.define('article-layout', ArticleLayout);
```

```html
<article-layout>
  <h1 slot="titulo">Guia Completo de Web Components</h1>
  <span slot="meta">Por Cleiton Carlos | Fevereiro 2026</span>

  <!-- Conteudo sem slot vai para o slot padrao (main) -->
  <p>Web Components sao um conjunto de APIs nativas...</p>
  <p>Neste artigo, vamos explorar...</p>

  <small slot="rodape">Publicado sob licenca MIT</small>
</article-layout>
```

### Fallback Content

Conteudo dentro das tags `<slot>` e exibido quando nenhum conteudo e projetado:

```javascript
class ProfileCard extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `
      <style>
        :host { display: inline-block; }
        .card { border: 1px solid #ddd; border-radius: 8px; padding: 20px; width: 250px; }
        .avatar-fallback {
          width: 60px; height: 60px; border-radius: 50%;
          background: #ccc; display: flex; align-items: center;
          justify-content: center; color: white; font-size: 1.5em;
        }
      </style>
      <div class="card">
        <slot name="avatar">
          <!-- Fallback: exibido se ninguem fornecer conteudo para slot="avatar" -->
          <div class="avatar-fallback">?</div>
        </slot>
        <h3><slot name="nome">Usuario Anonimo</slot></h3>
        <p><slot name="bio">Nenhuma bio disponivel</slot></p>
        <slot>
          <!-- Fallback do slot padrao -->
          <button>Ver perfil</button>
        </slot>
      </div>
    `;
  }
}

customElements.define('profile-card', ProfileCard);
```

```html
<!-- Com conteudo customizado -->
<profile-card>
  <img slot="avatar" src="foto.jpg" alt="Foto" />
  <span slot="nome">Cleiton</span>
  <span slot="bio">Desenvolvedor Front-end</span>
</profile-card>

<!-- Sem conteudo -- mostra todos os fallbacks -->
<profile-card></profile-card>
```

### Multiplos elementos no mesmo slot

Varios elementos podem apontar para o mesmo named slot -- todos aparecem em ordem:

```html
<article-layout>
  <span slot="meta">Autor: Cleiton</span>
  <span slot="meta"> | </span>
  <span slot="meta">Data: 2026-02-08</span>

  <p>Primeiro paragrafo do conteudo...</p>
  <p>Segundo paragrafo do conteudo...</p>
</article-layout>
```

### Evento `slotchange`

Disparado quando o conteudo projetado em um slot muda:

```javascript
class SlotMonitor extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `
      <div class="count">Itens: <span>0</span></div>
      <slot name="items"></slot>
    `;

    const slot = shadow.querySelector('slot[name="items"]');
    const countEl = shadow.querySelector('.count span');

    slot.addEventListener('slotchange', () => {
      const nodes = slot.assignedNodes({ flatten: true });
      // Filtrar apenas Element nodes (ignorar text nodes vazios)
      const elements = nodes.filter(n => n.nodeType === Node.ELEMENT_NODE);
      countEl.textContent = elements.length;
      console.log('Conteudo do slot mudou:', elements);
    });
  }
}

customElements.define('slot-monitor', SlotMonitor);
```

### APIs de Slot

```javascript
// Obter nos atribuidos a um slot
const slot = shadow.querySelector('slot[name="titulo"]');
slot.assignedNodes();                    // Nos atribuidos (pode incluir text nodes)
slot.assignedNodes({ flatten: true });   // Inclui fallback se nenhum no atribuido
slot.assignedElements();                 // Apenas Element nodes

// Obter o slot ao qual um elemento foi atribuido
const element = document.querySelector('[slot="titulo"]');
element.assignedSlot; // Retorna o <slot> correspondente
```

## Light DOM vs Shadow DOM

```
<my-component>           <-- Shadow Host
  |
  |-- #shadow-root       <-- Shadow Root (Shadow DOM)
  |   |-- <style>
  |   |-- <div class="wrapper">
  |   |   |-- <slot name="header">   <-- Ponto de projecao
  |   |   |-- <slot>                 <-- Slot padrao
  |   |
  |-- <h1 slot="header">Titulo</h1>  <-- Light DOM (projetado no slot)
  |-- <p>Conteudo</p>                 <-- Light DOM (projetado no slot padrao)
```

O conteudo do Light DOM **nao e movido** para dentro do Shadow DOM -- ele permanece no Light DOM mas e **renderizado** na posicao do slot correspondente. Isso e importante para:

- Estilizacao: estilos globais afetam o Light DOM, nao o Shadow DOM
- Queries: `document.querySelector` encontra elementos do Light DOM
- Eventos: eventos do Light DOM mantem seu target original

## Exercicios

1. Crie um componente `tab-container` com slots nomeados para cada aba e conteudo
2. Crie um componente `modal-dialog` que use slots para header, body e footer, com fallbacks padroes
3. Use o evento `slotchange` para criar um componente que conta e exibe quantos itens foram projetados em um slot
