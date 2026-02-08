# Capitulo 13: Micro Frontends e Casos Reais

## O que sao Micro Frontends?

Micro Frontends aplicam o conceito de microservicos ao frontend -- dividir uma aplicacao monolitica em partes menores, independentes e autonomas.

### Beneficios

- **Autonomia**: Times diferentes podem trabalhar em partes diferentes
- **Tecnologia**: Cada parte pode usar stack diferente
- **Deploy Independente**: Atualizar uma parte sem afetar outras
- **Escalabilidade**: Times e codigo escalam separadamente
- **Isolamento**: Falhas isoladas, nao derrubam toda aplicacao

### Web Components para Micro Frontends

Web Components sao ideais porque:
- Framework-agnostic
- Encapsulamento natural (Shadow DOM)
- API padrao do navegador
- Funciona com qualquer stack

## Arquitetura

### Shell Application (Host)

O shell e a aplicacao container que carrega os micro frontends:

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
  <title>E-commerce Platform</title>
  <style>
    body {
      margin: 0;
      font-family: system-ui;
    }
    .layout {
      display: grid;
      grid-template-areas:
        "header header"
        "sidebar content";
      grid-template-columns: 250px 1fr;
      grid-template-rows: auto 1fr;
      min-height: 100vh;
    }
    .header { grid-area: header; }
    .sidebar { grid-area: sidebar; }
    .content { grid-area: content; }
  </style>
</head>
<body>
  <div class="layout">
    <div class="header">
      <!-- Micro Frontend: Header (Team A) -->
      <mfe-header></mfe-header>
    </div>

    <div class="sidebar">
      <!-- Micro Frontend: Navigation (Team A) -->
      <mfe-navigation></mfe-navigation>
    </div>

    <div class="content">
      <!-- Micro Frontend: Product Catalog (Team B) -->
      <mfe-product-catalog></mfe-product-catalog>

      <!-- Micro Frontend: Shopping Cart (Team C) -->
      <mfe-shopping-cart></mfe-shopping-cart>
    </div>
  </div>

  <!-- Carregar micro frontends -->
  <script type="module" src="https://cdn.team-a.com/mfe-header.js"></script>
  <script type="module" src="https://cdn.team-a.com/mfe-navigation.js"></script>
  <script type="module" src="https://cdn.team-b.com/mfe-product-catalog.js"></script>
  <script type="module" src="https://cdn.team-c.com/mfe-shopping-cart.js"></script>
</body>
</html>
```

### Micro Frontend Component

Cada time desenvolve seu componente independentemente:

```javascript
// Team B: Product Catalog
class MfeProductCatalog extends HTMLElement {
  #products = [];

  async connectedCallback() {
    this.attachShadow({ mode: 'open' });

    // Cada MFE tem seu proprio sistema de build, bundling, etc
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          padding: 20px;
        }
        .grid {
          display: grid;
          grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
          gap: 20px;
        }
        .product-card {
          border: 1px solid #e5e7eb;
          border-radius: 8px;
          padding: 16px;
          cursor: pointer;
          transition: box-shadow 0.2s;
        }
        .product-card:hover {
          box-shadow: 0 4px 12px rgba(0,0,0,0.1);
        }
        .product-image {
          width: 100%;
          height: 200px;
          object-fit: cover;
          border-radius: 4px;
          margin-bottom: 12px;
        }
        .product-title {
          font-weight: 600;
          margin-bottom: 8px;
        }
        .product-price {
          color: #22c55e;
          font-size: 1.2em;
          font-weight: bold;
        }
      </style>
      <div class="loading">Carregando produtos...</div>
    `;

    await this.#loadProducts();
    this.#render();
  }

  async #loadProducts() {
    // Cada MFE pode ter sua propria API
    const response = await fetch('https://api.team-b.com/products');
    this.#products = await response.json();
  }

  #render() {
    this.shadowRoot.innerHTML = `
      <style>
        /* ... styles ... */
      </style>
      <h2>Produtos em Destaque</h2>
      <div class="grid">
        ${this.#products.map(product => `
          <div class="product-card" data-id="${product.id}">
            <img class="product-image" src="${product.image}" alt="${product.name}" />
            <div class="product-title">${product.name}</div>
            <div class="product-price">R$ ${product.price.toFixed(2)}</div>
            <button data-product-id="${product.id}">Adicionar ao Carrinho</button>
          </div>
        `).join('')}
      </div>
    `;

    // Comunicacao entre MFEs via eventos
    this.shadowRoot.addEventListener('click', (e) => {
      if (e.target.matches('button')) {
        const productId = e.target.dataset.productId;
        const product = this.#products.find(p => p.id === parseInt(productId));

        // Disparar evento global
        window.dispatchEvent(new CustomEvent('cart:add-item', {
          detail: { product }
        }));
      }
    });
  }
}

customElements.define('mfe-product-catalog', MfeProductCatalog);
```

```javascript
// Team C: Shopping Cart
class MfeShoppingCart extends HTMLElement {
  #items = [];

  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.#render();

    // Escutar eventos de outros MFEs
    window.addEventListener('cart:add-item', (e) => {
      this.#addItem(e.detail.product);
    });
  }

  #addItem(product) {
    const existing = this.#items.find(item => item.id === product.id);

    if (existing) {
      existing.quantity++;
    } else {
      this.#items.push({ ...product, quantity: 1 });
    }

    this.#render();

    // Notificar outros MFEs
    window.dispatchEvent(new CustomEvent('cart:updated', {
      detail: {
        items: this.#items,
        total: this.#calculateTotal()
      }
    }));
  }

  #calculateTotal() {
    return this.#items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
  }

  #render() {
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          padding: 20px;
          background: #f9fafb;
          border-radius: 8px;
        }
        .cart-item {
          display: flex;
          justify-content: space-between;
          padding: 12px 0;
          border-bottom: 1px solid #e5e7eb;
        }
        .total {
          font-size: 1.4em;
          font-weight: bold;
          margin-top: 16px;
          text-align: right;
        }
      </style>
      <h3>Carrinho (${this.#items.length} itens)</h3>
      <div class="items">
        ${this.#items.map(item => `
          <div class="cart-item">
            <span>${item.name} (x${item.quantity})</span>
            <span>R$ ${(item.price * item.quantity).toFixed(2)}</span>
          </div>
        `).join('')}
      </div>
      <div class="total">
        Total: R$ ${this.#calculateTotal().toFixed(2)}
      </div>
      <button style="width: 100%; padding: 12px; margin-top: 16px; background: #3b82f6; color: white; border: none; border-radius: 6px; cursor: pointer;">
        Finalizar Compra
      </button>
    `;
  }
}

customElements.define('mfe-shopping-cart', MfeShoppingCart);
```

## Comunicacao entre Micro Frontends

### 1. Custom Events (Window)

```javascript
// MFE A: Disparar evento
window.dispatchEvent(new CustomEvent('user:login', {
  detail: {
    userId: '123',
    username: 'joao'
  }
}));

// MFE B: Escutar evento
window.addEventListener('user:login', (e) => {
  console.log('Usuario logou:', e.detail.username);
});
```

### 2. Event Bus Global

```javascript
// shared/event-bus.js (compartilhado entre MFEs)
class GlobalEventBus {
  static #instance;
  #events = new Map();

  static getInstance() {
    if (!GlobalEventBus.#instance) {
      GlobalEventBus.#instance = new GlobalEventBus();
    }
    return GlobalEventBus.#instance;
  }

  on(event, callback) {
    if (!this.#events.has(event)) {
      this.#events.set(event, new Set());
    }
    this.#events.get(event).add(callback);

    return () => this.#events.get(event).delete(callback);
  }

  emit(event, data) {
    this.#events.get(event)?.forEach(cb => cb(data));
  }
}

window.MFE_EventBus = GlobalEventBus.getInstance();
```

### 3. Shared State

```javascript
// shared/store.js
class GlobalStore {
  static #instance;
  #state = {};
  #listeners = new Set();

  static getInstance() {
    if (!GlobalStore.#instance) {
      GlobalStore.#instance = new GlobalStore();
    }
    return GlobalStore.#instance;
  }

  setState(key, value) {
    this.#state[key] = value;
    this.#notify(key, value);
  }

  getState(key) {
    return this.#state[key];
  }

  subscribe(callback) {
    this.#listeners.add(callback);
    return () => this.#listeners.delete(callback);
  }

  #notify(key, value) {
    this.#listeners.forEach(cb => cb(key, value));
  }
}

window.MFE_Store = GlobalStore.getInstance();
```

## Casos Reais de Uso

### 1. GitHub

GitHub usa Web Components extensivamente:

```html
<!-- Exemplos reais de componentes do GitHub -->
<relative-time datetime="2026-02-08T10:00:00Z">
  Feb 8, 2026
</relative-time>

<details-dialog class="Box Box--overlay">
  <summary>Open Dialog</summary>
  <div>Dialog content</div>
</details-dialog>

<auto-complete src="/users/search">
  <input type="text" />
</auto-complete>

<clipboard-copy value="npm install">
  Copy
</clipboard-copy>
```

**Por que funcionou**:
- Encapsulamento perfeito
- Sem conflitos CSS
- Facil manutencao
- Reutilizavel em todo o site

### 2. Salesforce (Lightning Web Components)

Salesforce construiu toda sua plataforma em Web Components:

```javascript
// Exemplo simplificado de Lightning Web Component
import { LightningElement, api } from 'lwc';

export default class ContactCard extends LightningElement {
  @api contact;

  handleCall() {
    this.dispatchEvent(new CustomEvent('call', {
      detail: { phoneNumber: this.contact.phone }
    }));
  }
}
```

**Escala**:
- Milhares de componentes
- Milhoes de usuarios
- Organizacao isolada
- Performance otimizada

### 3. Adobe (Spectrum Web Components)

Design system completo em Web Components:

```html
<sp-button variant="cta">Call to Action</sp-button>
<sp-textfield label="Name" placeholder="Enter your name"></sp-textfield>
<sp-slider value="50" min="0" max="100"></sp-slider>
<sp-dropdown>
  <sp-menu-item>Option 1</sp-menu-item>
  <sp-menu-item>Option 2</sp-menu-item>
</sp-dropdown>
```

### 4. ING Bank (Lion Web Components)

Sistema bancario enterprise:

```html
<lion-input name="account" label="Account Number"></lion-input>
<lion-select name="type" label="Account Type">
  <select slot="input">
    <option value="checking">Checking</option>
    <option value="savings">Savings</option>
  </select>
</lion-select>
<lion-button type="submit">Transfer</lion-button>
```

## Implementacao Pratica

### Module Federation (Webpack 5)

```javascript
// webpack.config.js (Team A)
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'team_a',
      filename: 'remoteEntry.js',
      exposes: {
        './Header': './src/components/Header',
        './Navigation': './src/components/Navigation'
      },
      shared: ['lit', 'shared-utils']
    })
  ]
};

// webpack.config.js (Shell)
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        team_a: 'team_a@https://cdn.team-a.com/remoteEntry.js',
        team_b: 'team_b@https://cdn.team-b.com/remoteEntry.js'
      }
    })
  ]
};
```

### Import Maps

```html
<script type="importmap">
{
  "imports": {
    "mfe-header": "https://cdn.team-a.com/mfe-header.js",
    "mfe-catalog": "https://cdn.team-b.com/mfe-catalog.js",
    "mfe-cart": "https://cdn.team-c.com/mfe-cart.js",
    "shared/utils": "https://cdn.shared.com/utils.js"
  }
}
</script>

<script type="module">
  import 'mfe-header';
  import 'mfe-catalog';
  import 'mfe-cart';
</script>
```

## Desafios e Solucoes

### Desafio 1: Estilo Global vs Isolado

**Problema**: Como compartilhar estilos sem quebrar isolamento?

**Solucao**:
```javascript
// Design tokens como CSS Custom Properties
:root {
  --color-primary: #3b82f6;
  --color-secondary: #8b5cf6;
  --spacing-unit: 8px;
  --font-family: system-ui;
}

// MFEs usam as variaveis
class MyMFE extends HTMLElement {
  connectedCallback() {
    this.shadowRoot.innerHTML = `
      <style>
        button {
          background: var(--color-primary);
          padding: calc(var(--spacing-unit) * 2);
          font-family: var(--font-family);
        }
      </style>
      <button>Click Me</button>
    `;
  }
}
```

### Desafio 2: Versionamento

**Problema**: Como garantir compatibilidade entre versoes?

**Solucao**:
```html
<!-- Versionamento no nome ou URL -->
<script type="module" src="https://cdn.team-a.com/mfe-header@2.5.3.js"></script>

<!-- OU usar import maps com versoes -->
<script type="importmap">
{
  "imports": {
    "mfe-header": "https://cdn.team-a.com/mfe-header@^2.0.0.js"
  }
}
</script>
```

### Desafio 3: Performance

**Problema**: Carregar multiplos MFEs pode ser lento.

**Solucao**:
```javascript
// Lazy load MFEs
const observer = new IntersectionObserver((entries) => {
  entries.forEach(async (entry) => {
    if (entry.isIntersecting) {
      const mfeName = entry.target.tagName.toLowerCase();
      if (!customElements.get(mfeName)) {
        await import(`https://cdn.example.com/${mfeName}.js`);
      }
    }
  });
});

document.querySelectorAll('[data-mfe]').forEach(el => {
  observer.observe(el);
});
```

## Best Practices

### ✅ Organizacao

- [ ] Definir API de comunicacao clara entre MFEs
- [ ] Documentar eventos e contratos
- [ ] Versionar MFEs semanticamente
- [ ] Usar monorepo para shared code

### ✅ Performance

- [ ] Lazy load MFEs nao criticos
- [ ] Compartilhar dependencias comuns
- [ ] Usar CDN para servir MFEs
- [ ] Implementar cache strategies

### ✅ Desenvolvimento

- [ ] Cada MFE deve rodar standalone
- [ ] Testes independentes por MFE
- [ ] CI/CD separado por time
- [ ] Ambiente de desenvolvimento integrado (shell local)

### ✅ Producao

- [ ] Monitoring por MFE
- [ ] Feature flags para rollback
- [ ] Estrategia de deploy (blue/green, canary)
- [ ] Fallbacks para falhas de MFE

## Exercicios

1. Crie uma aplicacao shell que carrega 3 micro frontends diferentes
2. Implemente comunicacao via Event Bus entre MFEs
3. Configure Module Federation para compartilhar dependencias
4. Crie um design system de tokens CSS compartilhados

## Conclusao

Web Components nativos oferecem a base perfeita para Micro Frontends porque:
- Sao framework-agnostic
- Tem encapsulamento real
- Sao padroes da web (sem lock-in)
- Performance nativa do navegador

Empresas como GitHub, Salesforce, Adobe, Microsoft e ING provam que Web Components escalam para aplicacoes enterprise de milhoes de usuarios.
