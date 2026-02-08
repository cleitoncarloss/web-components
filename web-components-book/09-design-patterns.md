# Capitulo 9: Design Patterns

## Observer Pattern

O Observer Pattern permite que objetos se inscrevam para receber notificacoes quando algo muda.

### Implementacao Basica

```javascript
class Observable {
  #observers = new Set();

  subscribe(observer) {
    this.#observers.add(observer);

    // Retornar unsubscribe function
    return () => {
      this.#observers.delete(observer);
    };
  }

  notify(data) {
    this.#observers.forEach(observer => {
      observer(data);
    });
  }
}

// Uso
class DataSource extends Observable {
  #data = null;

  setData(newData) {
    this.#data = newData;
    this.notify(newData);
  }

  getData() {
    return this.#data;
  }
}

// Componente observer
class DataDisplay extends HTMLElement {
  #unsubscribe = null;

  connectedCallback() {
    this.attachShadow({ mode: 'open' });

    // Inscrever no data source
    this.#unsubscribe = window.dataSource.subscribe((data) => {
      this.render(data);
    });

    // Render inicial
    this.render(window.dataSource.getData());
  }

  disconnectedCallback() {
    this.#unsubscribe?.();
  }

  render(data) {
    this.shadowRoot.innerHTML = `
      <div>Data: ${JSON.stringify(data)}</div>
    `;
  }
}

// Setup global
window.dataSource = new DataSource();
customElements.define('data-display', DataDisplay);
```

## Factory Pattern

O Factory Pattern centraliza a criacao de objetos/componentes:

```javascript
class ComponentFactory {
  static #registry = new Map();

  // Registrar componente
  static register(type, componentClass, options = {}) {
    const tagName = options.tagName || `app-${type}`;

    this.#registry.set(type, {
      class: componentClass,
      tagName,
      options
    });

    // Auto-definir custom element
    if (!customElements.get(tagName)) {
      customElements.define(tagName, componentClass);
    }
  }

  // Criar instancia
  static create(type, props = {}, children = []) {
    const config = this.#registry.get(type);

    if (!config) {
      throw new Error(`Component type "${type}" not registered`);
    }

    const element = document.createElement(config.tagName);

    // Aplicar propriedades
    Object.entries(props).forEach(([key, value]) => {
      if (typeof value === 'string' || typeof value === 'number' || typeof value === 'boolean') {
        element.setAttribute(key, String(value));
      } else {
        element[key] = value;
      }
    });

    // Adicionar children
    children.forEach(child => {
      if (typeof child === 'string') {
        element.appendChild(document.createTextNode(child));
      } else {
        element.appendChild(child);
      }
    });

    return element;
  }

  // Criar em lote
  static createBatch(configs) {
    return configs.map(config =>
      this.create(config.type, config.props, config.children)
    );
  }

  // Listar tipos registrados
  static getTypes() {
    return Array.from(this.#registry.keys());
  }
}

// Registrar componentes
class AppButton extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        button {
          padding: 10px 20px;
          background: ${this.getAttribute('variant') === 'primary' ? '#3b82f6' : '#6b7280'};
          color: white;
          border: none;
          border-radius: 6px;
          cursor: pointer;
        }
      </style>
      <button><slot></slot></button>
    `;
  }
}

class AppCard extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        .card {
          border: 1px solid #e5e7eb;
          border-radius: 8px;
          padding: 16px;
          box-shadow: 0 1px 3px rgba(0,0,0,0.1);
        }
      </style>
      <div class="card"><slot></slot></div>
    `;
  }
}

// Registrar na factory
ComponentFactory.register('button', AppButton);
ComponentFactory.register('card', AppCard);

// Usar a factory
const button = ComponentFactory.create('button', { variant: 'primary' }, ['Clique Aqui']);
const card = ComponentFactory.create('card', {}, [
  'Conteudo do card',
  button
]);

document.body.appendChild(card);
```

## Mixin Pattern

Mixins permitem compartilhar funcionalidades entre componentes sem heranca multipla:

### Mixin de Logging

```javascript
const LoggingMixin = (Base) => class extends Base {
  connectedCallback() {
    console.log(`[${this.tagName}] connected`);
    console.time(`[${this.tagName}] connectedCallback`);
    super.connectedCallback?.();
    console.timeEnd(`[${this.tagName}] connectedCallback`);
  }

  disconnectedCallback() {
    console.log(`[${this.tagName}] disconnected`);
    super.disconnectedCallback?.();
  }

  attributeChangedCallback(name, oldValue, newValue) {
    console.log(`[${this.tagName}] attribute "${name}": "${oldValue}" → "${newValue}"`);
    super.attributeChangedCallback?.(name, oldValue, newValue);
  }
};
```

### Mixin de Tematizacao

```javascript
const ThemeMixin = (Base) => class extends Base {
  static get observedAttributes() {
    return [...(super.observedAttributes || []), 'theme'];
  }

  connectedCallback() {
    super.connectedCallback?.();
    this.#applyTheme();

    // Escutar mudancas no tema do sistema
    this.#mediaQuery = window.matchMedia('(prefers-color-scheme: dark)');
    this.#themeHandler = () => this.#applyTheme();
    this.#mediaQuery.addEventListener('change', this.#themeHandler);
  }

  disconnectedCallback() {
    super.disconnectedCallback?.();
    this.#mediaQuery?.removeEventListener('change', this.#themeHandler);
  }

  #applyTheme() {
    const theme = this.getAttribute('theme') || 'auto';
    let isDark = false;

    if (theme === 'dark') {
      isDark = true;
    } else if (theme === 'auto') {
      isDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    }

    this.shadowRoot?.host?.classList.toggle('dark', isDark);

    this.dispatchEvent(new CustomEvent('theme-change', {
      detail: { theme: isDark ? 'dark' : 'light' },
      bubbles: true
    }));
  }
};
```

### Mixin de Fetchable

```javascript
const FetchMixin = (Base) => class extends Base {
  #abortController = null;
  #cache = new Map();

  async fetch(url, options = {}) {
    // Cancelar fetch anterior se houver
    this.#abortController?.abort();
    this.#abortController = new AbortController();

    // Verificar cache
    if (options.cache && this.#cache.has(url)) {
      return this.#cache.get(url);
    }

    try {
      const response = await fetch(url, {
        ...options,
        signal: this.#abortController.signal
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const data = await response.json();

      // Salvar em cache
      if (options.cache) {
        this.#cache.set(url, data);
      }

      return data;
    } catch (error) {
      if (error.name === 'AbortError') {
        console.log('Fetch aborted');
        return null;
      }
      throw error;
    }
  }

  disconnectedCallback() {
    super.disconnectedCallback?.();
    this.#abortController?.abort();
    this.#cache.clear();
  }
};
```

### Combinando Mixins

```javascript
class MyComponent extends ThemeMixin(FetchMixin(LoggingMixin(HTMLElement))) {
  async connectedCallback() {
    super.connectedCallback();

    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = '<div>Loading...</div>';

    // Usar funcionalidade do FetchMixin
    const data = await this.fetch('https://api.example.com/data', { cache: true });

    this.shadowRoot.innerHTML = `
      <style>
        :host(.dark) { background: #1f2937; color: white; }
      </style>
      <div>${JSON.stringify(data)}</div>
    `;
  }
}

customElements.define('my-component', MyComponent);
```

## Strategy Pattern

O Strategy Pattern permite trocar algoritmos/comportamentos em runtime:

```javascript
// Estrategias de renderizacao
const renderStrategies = {
  list: (items) => `
    <ul>
      ${items.map(item => `<li>${item.name} - ${item.value}</li>`).join('')}
    </ul>
  `,

  grid: (items) => `
    <div class="grid">
      ${items.map(item => `
        <div class="grid-item">
          <h4>${item.name}</h4>
          <p>${item.value}</p>
        </div>
      `).join('')}
    </div>
  `,

  table: (items) => `
    <table>
      <thead>
        <tr>
          <th>Nome</th>
          <th>Valor</th>
        </tr>
      </thead>
      <tbody>
        ${items.map(item => `
          <tr>
            <td>${item.name}</td>
            <td>${item.value}</td>
          </tr>
        `).join('')}
      </tbody>
    </table>
  `,

  cards: (items) => `
    <div class="cards">
      ${items.map(item => `
        <div class="card">
          <h3>${item.name}</h3>
          <p>${item.value}</p>
        </div>
      `).join('')}
    </div>
  `
};

class DataView extends HTMLElement {
  static observedAttributes = ['view'];

  #items = [];

  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.render();
  }

  set items(value) {
    this.#items = value;
    if (this.isConnected) {
      this.render();
    }
  }

  get items() {
    return this.#items;
  }

  attributeChangedCallback() {
    if (this.isConnected) {
      this.render();
    }
  }

  render() {
    const viewType = this.getAttribute('view') || 'list';
    const strategy = renderStrategies[viewType] || renderStrategies.list;

    this.shadowRoot.innerHTML = `
      <style>
        :host { display: block; padding: 16px; }
        .grid {
          display: grid;
          grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
          gap: 16px;
        }
        .grid-item {
          padding: 16px;
          border: 1px solid #e5e7eb;
          border-radius: 8px;
        }
        table {
          width: 100%;
          border-collapse: collapse;
        }
        th, td {
          padding: 12px;
          text-align: left;
          border-bottom: 1px solid #e5e7eb;
        }
        th {
          background: #f9fafb;
          font-weight: 600;
        }
        .cards {
          display: flex;
          flex-direction: column;
          gap: 12px;
        }
        .card {
          padding: 16px;
          border: 1px solid #e5e7eb;
          border-radius: 8px;
          box-shadow: 0 1px 3px rgba(0,0,0,0.1);
        }
        .card h3 {
          margin: 0 0 8px;
        }
      </style>
      ${strategy(this.#items)}
    `;
  }
}

customElements.define('data-view', DataView);
```

```html
<!-- Trocar visualizacao -->
<select id="viewSelect">
  <option value="list">Lista</option>
  <option value="grid">Grade</option>
  <option value="table">Tabela</option>
  <option value="cards">Cards</option>
</select>

<data-view id="dataView" view="list"></data-view>

<script>
  const dataView = document.getElementById('dataView');
  dataView.items = [
    { name: 'Item 1', value: '100' },
    { name: 'Item 2', value: '200' },
    { name: 'Item 3', value: '300' },
    { name: 'Item 4', value: '400' }
  ];

  document.getElementById('viewSelect').addEventListener('change', (e) => {
    dataView.setAttribute('view', e.target.value);
  });
</script>
```

## Adapter Pattern

O Adapter Pattern permite envolver bibliotecas de terceiros:

```javascript
// Adapter para biblioteca de graficos (ex: Chart.js)
class ChartAdapter extends HTMLElement {
  static observedAttributes = ['type', 'data-url'];

  #chart = null;

  async connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          width: 100%;
          height: 400px;
        }
        canvas {
          width: 100% !important;
          height: 100% !important;
        }
      </style>
      <canvas></canvas>
    `;

    // Lazy load da biblioteca
    if (!window.Chart) {
      await this.#loadChartLibrary();
    }

    await this.#initChart();
  }

  disconnectedCallback() {
    this.#chart?.destroy();
  }

  async #loadChartLibrary() {
    // Simular carregamento de biblioteca externa
    return new Promise((resolve) => {
      const script = document.createElement('script');
      script.src = 'https://cdn.jsdelivr.net/npm/chart.js';
      script.onload = resolve;
      document.head.appendChild(script);
    });
  }

  async #initChart() {
    const canvas = this.shadowRoot.querySelector('canvas');
    const ctx = canvas.getContext('2d');

    const type = this.getAttribute('type') || 'bar';
    let data = { labels: [], datasets: [] };

    // Carregar dados se data-url foi fornecido
    const dataUrl = this.getAttribute('data-url');
    if (dataUrl) {
      data = await this.#fetchData(dataUrl);
    }

    this.#chart = new Chart(ctx, {
      type,
      data,
      options: {
        responsive: true,
        maintainAspectRatio: false
      }
    });
  }

  async #fetchData(url) {
    const response = await fetch(url);
    return await response.json();
  }

  // API publica para atualizar dados
  updateData(newData) {
    if (this.#chart) {
      this.#chart.data = newData;
      this.#chart.update();
    }
  }

  attributeChangedCallback(name, oldValue, newValue) {
    if (oldValue === newValue || !this.#chart) return;

    if (name === 'type') {
      this.#chart.config.type = newValue;
      this.#chart.update();
    } else if (name === 'data-url') {
      this.#fetchData(newValue).then(data => {
        this.updateData(data);
      });
    }
  }
}

customElements.define('chart-adapter', ChartAdapter);
```

## Singleton Pattern

Garantir uma unica instancia de um servico:

```javascript
class ServiceRegistry {
  static #instance = null;
  #services = new Map();

  constructor() {
    if (ServiceRegistry.#instance) {
      return ServiceRegistry.#instance;
    }
    ServiceRegistry.#instance = this;
  }

  static getInstance() {
    if (!ServiceRegistry.#instance) {
      ServiceRegistry.#instance = new ServiceRegistry();
    }
    return ServiceRegistry.#instance;
  }

  register(name, service) {
    if (this.#services.has(name)) {
      console.warn(`Service "${name}" already registered`);
      return;
    }
    this.#services.set(name, service);
  }

  get(name) {
    if (!this.#services.has(name)) {
      throw new Error(`Service "${name}" not found`);
    }
    return this.#services.get(name);
  }

  has(name) {
    return this.#services.has(name);
  }
}

// Uso
const registry = ServiceRegistry.getInstance();

// Registrar servicos
registry.register('api', {
  baseUrl: 'https://api.example.com',
  fetch: async (endpoint) => {
    const response = await fetch(`${this.baseUrl}${endpoint}`);
    return await response.json();
  }
});

registry.register('auth', {
  token: null,
  login: async (credentials) => {
    // ...
  },
  logout: () => {
    // ...
  }
});

// Usar em componentes
class MyComponent extends HTMLElement {
  async connectedCallback() {
    const api = ServiceRegistry.getInstance().get('api');
    const data = await api.fetch('/users');
    // ...
  }
}
```

## Decorator Pattern

Adicionar funcionalidade dinamicamente:

```javascript
function withCache(Component) {
  return class extends Component {
    #cache = new Map();

    async fetchWithCache(url) {
      if (this.#cache.has(url)) {
        console.log('Cache hit:', url);
        return this.#cache.get(url);
      }

      const data = await super.fetch?.(url) || await fetch(url).then(r => r.json());
      this.#cache.set(url, data);
      return data;
    }

    clearCache() {
      this.#cache.clear();
    }
  };
}

function withRetry(Component, maxRetries = 3) {
  return class extends Component {
    async fetchWithRetry(url, options = {}) {
      let lastError;

      for (let i = 0; i < maxRetries; i++) {
        try {
          return await super.fetch?.(url, options) || await fetch(url, options);
        } catch (error) {
          lastError = error;
          console.log(`Retry ${i + 1}/${maxRetries} for ${url}`);
          await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
        }
      }

      throw lastError;
    }
  };
}

// Aplicar decorators
class BaseDataComponent extends HTMLElement {
  async fetch(url) {
    const response = await fetch(url);
    return await response.json();
  }
}

const EnhancedDataComponent = withRetry(withCache(BaseDataComponent));
customElements.define('enhanced-data', EnhancedDataComponent);
```

## Exercicios

1. Implemente um ComponentFactory que suporte templates e validacao
2. Crie 3 mixins (Logging, Fetchable, Themable) e combine-os em um componente
3. Implemente Strategy Pattern para diferentes algoritmos de ordenacao em uma tabela
4. Crie um Adapter para integrar uma biblioteca de mapas (ex: Leaflet)
