# Capitulo 12: Performance e SSR

## Lazy Loading

Carregar componentes apenas quando necessario.

### Com Intersection Observer

```javascript
class LazyLoader {
  static #observer = null;
  static #registry = new Map();

  static init() {
    if (this.#observer) return;

    this.#observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(async (entry) => {
          if (entry.isIntersecting) {
            const element = entry.target;
            const tagName = element.tagName.toLowerCase();

            // Parar de observar
            this.#observer.unobserve(element);

            // Carregar definicao do componente
            if (!customElements.get(tagName)) {
              await this.#loadComponent(tagName);
            }
          }
        });
      },
      {
        rootMargin: '200px' // Comecar carregamento 200px antes
      }
    );
  }

  static register(tagName, importFn) {
    this.#registry.set(tagName, importFn);
  }

  static observe(tagName) {
    const elements = document.querySelectorAll(tagName);
    elements.forEach(el => {
      if (!customElements.get(tagName)) {
        this.#observer.observe(el);
      }
    });
  }

  static async #loadComponent(tagName) {
    const importFn = this.#registry.get(tagName);

    if (!importFn) {
      console.warn(`Component ${tagName} not registered for lazy loading`);
      return;
    }

    try {
      await importFn();
      console.log(`✓ Loaded ${tagName}`);
    } catch (error) {
      console.error(`Error loading ${tagName}:`, error);
    }
  }
}

// Inicializar
LazyLoader.init();

// Registrar componentes lazy
LazyLoader.register('heavy-chart', () => import('./components/heavy-chart.js'));
LazyLoader.register('data-table', () => import('./components/data-table.js'));
LazyLoader.register('video-player', () => import('./components/video-player.js'));

// Observar elementos na pagina
LazyLoader.observe('heavy-chart');
LazyLoader.observe('data-table');
LazyLoader.observe('video-player');
```

### Dynamic Import Manual

```javascript
class DynamicComponent extends HTMLElement {
  async connectedCallback() {
    this.attachShadow({ mode: 'open' });

    // Mostrar loading
    this.shadowRoot.innerHTML = `
      <style>
        .loading {
          padding: 40px;
          text-align: center;
          color: #666;
        }
        .spinner {
          border: 3px solid #f3f3f3;
          border-top: 3px solid #3b82f6;
          border-radius: 50%;
          width: 40px;
          height: 40px;
          animation: spin 1s linear infinite;
          margin: 0 auto 16px;
        }
        @keyframes spin {
          0% { transform: rotate(0deg); }
          100% { transform: rotate(360deg); }
        }
      </style>
      <div class="loading">
        <div class="spinner"></div>
        <p>Carregando componente...</p>
      </div>
    `;

    try {
      // Importar modulo dinamicamente
      const module = await import('./heavy-module.js');

      // Renderizar componente carregado
      module.render(this.shadowRoot);
    } catch (error) {
      this.shadowRoot.innerHTML = `
        <div style="color: red; padding: 20px;">
          Erro ao carregar componente: ${error.message}
        </div>
      `;
    }
  }
}

customElements.define('dynamic-component', DynamicComponent);
```

## Code Splitting

Dividir codigo em chunks menores:

```javascript
// components/chart-loader.js
class ChartLoader extends HTMLElement {
  async connectedCallback() {
    const type = this.getAttribute('type');

    // Carregar apenas o tipo de grafico necessario
    switch (type) {
      case 'line':
        await import('./charts/line-chart.js');
        break;
      case 'bar':
        await import('./charts/bar-chart.js');
        break;
      case 'pie':
        await import('./charts/pie-chart.js');
        break;
      default:
        console.warn('Unknown chart type:', type);
    }
  }
}

customElements.define('chart-loader', ChartLoader);
```

## Otimizacao de Rendering

### Batch Updates com requestAnimationFrame

```javascript
class OptimizedList extends HTMLElement {
  #items = [];
  #renderScheduled = false;

  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        .list { list-style: none; padding: 0; margin: 0; }
        .item { padding: 12px; border-bottom: 1px solid #e5e7eb; }
      </style>
      <ul class="list"></ul>
    `;
  }

  addItem(item) {
    this.#items.push(item);
    this.#scheduleRender();
  }

  addItems(items) {
    this.#items.push(...items);
    this.#scheduleRender();
  }

  #scheduleRender() {
    if (this.#renderScheduled) return;

    this.#renderScheduled = true;

    requestAnimationFrame(() => {
      this.#render();
      this.#renderScheduled = false;
    });
  }

  #render() {
    const list = this.shadowRoot.querySelector('.list');

    // Usar DocumentFragment para batch DOM operations
    const fragment = document.createDocumentFragment();

    this.#items.forEach(item => {
      const li = document.createElement('li');
      li.className = 'item';
      li.textContent = item.text;
      fragment.appendChild(li);
    });

    // Uma unica operacao DOM
    list.replaceChildren(fragment);
  }

  clear() {
    this.#items = [];
    this.#scheduleRender();
  }
}

customElements.define('optimized-list', OptimizedList);
```

### Virtual Scrolling

Para listas muito longas:

```javascript
class VirtualList extends HTMLElement {
  #items = [];
  #itemHeight = 50;
  #visibleCount = 0;
  #scrollTop = 0;

  connectedCallback() {
    this.attachShadow({ mode: 'open' });

    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          height: 400px;
          overflow-y: auto;
          border: 1px solid #e5e7eb;
        }
        .viewport {
          position: relative;
        }
        .spacer {
          position: absolute;
          top: 0;
          left: 0;
          right: 0;
        }
        .item {
          height: ${this.#itemHeight}px;
          padding: 12px;
          border-bottom: 1px solid #e5e7eb;
        }
      </style>
      <div class="viewport">
        <div class="spacer"></div>
      </div>
    `;

    this.addEventListener('scroll', () => {
      this.#scrollTop = this.scrollTop;
      this.#render();
    });

    this.#visibleCount = Math.ceil(this.clientHeight / this.#itemHeight) + 1;
  }

  set items(value) {
    this.#items = value;
    this.#render();
  }

  #render() {
    const startIndex = Math.floor(this.#scrollTop / this.#itemHeight);
    const endIndex = Math.min(
      startIndex + this.#visibleCount,
      this.#items.length
    );

    const spacer = this.shadowRoot.querySelector('.spacer');

    // Definir altura total para manter scroll correto
    spacer.style.height = `${this.#items.length * this.#itemHeight}px`;

    // Renderizar apenas itens visiveis
    const fragment = document.createDocumentFragment();

    for (let i = startIndex; i < endIndex; i++) {
      const item = document.createElement('div');
      item.className = 'item';
      item.style.transform = `translateY(${i * this.#itemHeight}px)`;
      item.style.position = 'absolute';
      item.style.left = '0';
      item.style.right = '0';
      item.textContent = this.#items[i];
      fragment.appendChild(item);
    }

    // Limpar e inserir
    const oldItems = spacer.querySelectorAll('.item');
    oldItems.forEach(item => item.remove());
    spacer.appendChild(fragment);
  }
}

customElements.define('virtual-list', VirtualList);
```

```javascript
// Uso com 10000 itens
const list = document.querySelector('virtual-list');
list.items = Array.from({ length: 10000 }, (_, i) => `Item ${i + 1}`);
```

## Constructable Stylesheets (Performance)

Compartilhar estilos entre multiplas instancias:

```javascript
// Criar stylesheet UMA vez
const sharedStyles = new CSSStyleSheet();
sharedStyles.replaceSync(`
  :host {
    display: block;
    font-family: system-ui;
  }
  .card {
    border: 1px solid #e5e7eb;
    border-radius: 8px;
    padding: 16px;
    margin-bottom: 12px;
  }
  .title {
    font-size: 1.2em;
    font-weight: bold;
    margin-bottom: 8px;
  }
`);

class Card extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });

    // Reutilizar stylesheet -- zero overhead de parsing
    shadow.adoptedStyleSheets = [sharedStyles];

    shadow.innerHTML = `
      <div class="card">
        <div class="title"><slot name="title"></slot></div>
        <slot></slot>
      </div>
    `;
  }
}

customElements.define('shared-card', Card);
```

**Beneficio**: 1000 instancias = 1 parse de CSS (vs 1000 parses com `<style>`)

## Server-Side Rendering (SSR)

### Declarative Shadow DOM

Permite que o Shadow DOM seja renderizado no HTML sem JavaScript:

```html
<!-- Renderizado pelo servidor -->
<my-card>
  <template shadowrootmode="open">
    <style>
      .card {
        border: 1px solid #ddd;
        padding: 16px;
        border-radius: 8px;
      }
      .title {
        font-weight: bold;
        margin-bottom: 8px;
      }
    </style>
    <div class="card">
      <div class="title"><slot name="title">Titulo</slot></div>
      <div><slot>Conteudo</slot></div>
    </div>
  </template>

  <!-- Light DOM (conteudo do usuario) -->
  <span slot="title">Meu Titulo</span>
  <p>Meu conteudo aqui</p>
</my-card>
```

**Vantagens**:
- Renderiza sem JavaScript
- Melhora FCP (First Contentful Paint)
- Melhora SEO
- Progressive Enhancement

### Gerando DSD no Servidor (Node.js)

```javascript
// server.js (Node.js + Express)
import { JSDOM } from 'jsdom';

function renderCard(title, content) {
  return `
    <my-card>
      <template shadowrootmode="open">
        <style>
          .card {
            border: 1px solid #ddd;
            padding: 16px;
            border-radius: 8px;
          }
        </style>
        <div class="card">
          <div class="title"><slot name="title">Titulo</slot></div>
          <div><slot></slot></div>
        </div>
      </template>
      <span slot="title">${escapeHtml(title)}</span>
      <p>${escapeHtml(content)}</p>
    </my-card>
  `;
}

app.get('/produto/:id', async (req, res) => {
  const produto = await db.getProduto(req.params.id);

  const html = `
    <!DOCTYPE html>
    <html>
      <head>
        <title>${produto.nome}</title>
        <script type="module" src="/components/my-card.js"></script>
      </head>
      <body>
        ${renderCard(produto.nome, produto.descricao)}
      </body>
    </html>
  `;

  res.send(html);
});

function escapeHtml(text) {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;');
}
```

### Hydration

Quando o JavaScript carrega, ele "hidrata" o componente:

```javascript
class MyCard extends HTMLElement {
  connectedCallback() {
    // Se ja tem shadowRoot (DSD), apenas adicionar comportamento
    if (this.shadowRoot) {
      this.#attachBehavior();
      return;
    }

    // Caso contrario, criar Shadow DOM normalmente
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>...</style>
      <div class="card">...</div>
    `;

    this.#attachBehavior();
  }

  #attachBehavior() {
    // Adicionar interatividade
    const card = this.shadowRoot.querySelector('.card');
    card.addEventListener('click', () => {
      console.log('Card clicado');
    });
  }
}

customElements.define('my-card', MyCard);
```

## Medindo Performance

### Performance Observer

```javascript
class PerformanceMonitor {
  static init() {
    // Observar custom element timing
    const observer = new PerformanceObserver((list) => {
      list.getEntries().forEach((entry) => {
        console.log(`${entry.name}: ${entry.duration}ms`);
      });
    });

    observer.observe({ entryTypes: ['measure'] });
  }

  static measure(name, startMark, endMark) {
    try {
      performance.measure(name, startMark, endMark);
    } catch (e) {
      console.warn('Performance measurement failed:', e);
    }
  }
}

PerformanceMonitor.init();

class MonitoredComponent extends HTMLElement {
  connectedCallback() {
    performance.mark('component-start');

    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `<div>Content</div>`;

    performance.mark('component-end');
    PerformanceMonitor.measure(
      'component-render',
      'component-start',
      'component-end'
    );
  }
}

customElements.define('monitored-component', MonitoredComponent);
```

### Web Vitals

```javascript
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

function sendToAnalytics(metric) {
  console.log(metric.name, metric.value);
  // Enviar para Google Analytics, etc
}

getCLS(sendToAnalytics);  // Cumulative Layout Shift
getFID(sendToAnalytics);  // First Input Delay
getFCP(sendToAnalytics);  // First Contentful Paint
getLCP(sendToAnalytics);  // Largest Contentful Paint
getTTFB(sendToAnalytics); // Time to First Byte
```

## Checklist de Performance

### ✅ Carregamento

- [ ] Lazy load componentes pesados
- [ ] Code splitting para reducir bundle inicial
- [ ] Usar dynamic imports
- [ ] Preload componentes criticos
- [ ] Defer componentes nao-criticos

### ✅ Rendering

- [ ] Batch DOM updates (requestAnimationFrame)
- [ ] Usar DocumentFragment para insercoes em massa
- [ ] Implementar virtual scrolling para listas longas
- [ ] Evitar layout thrashing (ler/escrever DOM alternadamente)
- [ ] Usar Constructable Stylesheets

### ✅ Runtime

- [ ] Limpar timers em disconnectedCallback
- [ ] Abortar fetches pendentes
- [ ] Desconectar observers
- [ ] Evitar re-renders desnecessarios (debounce/throttle)
- [ ] Usar Web Workers para processamento pesado

### ✅ Bundle Size

- [ ] Tree-shaking habilitado
- [ ] Minification
- [ ] Compression (gzip/brotli)
- [ ] Remover dependencias nao usadas
- [ ] Code splitting por rota/feature

### ✅ SSR/SEO

- [ ] Usar Declarative Shadow DOM quando possivel
- [ ] Garantir conteudo acessivel sem JavaScript
- [ ] Meta tags apropriadas
- [ ] Structured data (JSON-LD)
- [ ] Sitemap.xml atualizado

## Exercicios

1. Implemente lazy loading com Intersection Observer para uma galeria de imagens
2. Crie uma lista virtual scrolling que suporte 100.000 itens
3. Adicione Declarative Shadow DOM a um componente existente
4. Meça e otimize o tempo de render de um componente complexo
