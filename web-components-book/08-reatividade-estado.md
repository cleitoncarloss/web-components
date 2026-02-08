# Capitulo 8: Reatividade e Estado

## O que e Reatividade?

Reatividade significa que a UI atualiza automaticamente quando o estado muda. Em vez de chamar `render()` manualmente, mudancas no estado disparam atualizacoes automaticas.

## Proxy para Reatividade

A API `Proxy` do JavaScript permite interceptar operacoes em objetos, tornando-os reativos:

### Reatividade Basica

```javascript
class ReactiveElement extends HTMLElement {
  #state;
  #shadow;

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: 'open' });

    // Criar estado reativo
    this.#state = this.#createReactiveState({});
  }

  #createReactiveState(initialState) {
    return new Proxy(initialState, {
      set: (target, property, value) => {
        const oldValue = target[property];
        target[property] = value;

        // Automaticamente re-renderizar quando estado muda
        if (oldValue !== value) {
          this.render();
          this.#notifyChange(property, oldValue, value);
        }

        return true;
      },

      deleteProperty: (target, property) => {
        const oldValue = target[property];
        delete target[property];
        this.render();
        this.#notifyChange(property, oldValue, undefined);
        return true;
      }
    });
  }

  #notifyChange(property, oldValue, newValue) {
    this.dispatchEvent(new CustomEvent('state-change', {
      detail: { property, oldValue, newValue },
      bubbles: true,
      composed: true
    }));
  }

  get state() {
    return this.#state;
  }

  render() {
    // Sobrescrever em subclasses
  }
}
```

### Exemplo: Contador Reativo

```javascript
class ReactiveCounter extends ReactiveElement {
  connectedCallback() {
    // Inicializar estado
    this.state.count = 0;
    this.state.step = 1;

    this.render();

    // Event delegation
    this.shadowRoot.addEventListener('click', (e) => {
      if (e.target.matches('.increment')) {
        this.state.count += this.state.step;
      }
      if (e.target.matches('.decrement')) {
        this.state.count -= this.state.step;
      }
      if (e.target.matches('.reset')) {
        this.state.count = 0;
      }
    });
  }

  render() {
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: flex;
          align-items: center;
          gap: 12px;
          padding: 20px;
          font-family: system-ui;
        }
        button {
          padding: 8px 16px;
          font-size: 16px;
          border: none;
          border-radius: 6px;
          cursor: pointer;
          transition: all 0.2s;
        }
        .increment {
          background: #22c55e;
          color: white;
        }
        .decrement {
          background: #ef4444;
          color: white;
        }
        .reset {
          background: #6b7280;
          color: white;
        }
        button:hover {
          transform: scale(1.05);
        }
        .count {
          font-size: 32px;
          font-weight: bold;
          min-width: 60px;
          text-align: center;
          color: #1f2937;
        }
      </style>
      <button class="decrement">-</button>
      <div class="count">${this.state.count || 0}</div>
      <button class="increment">+</button>
      <button class="reset">Reset</button>
    `;
  }
}

customElements.define('reactive-counter', ReactiveCounter);
```

## Reatividade Profunda (Deep Reactivity)

Para objetos aninhados, precisamos criar Proxies recursivamente:

```javascript
function createDeepReactive(obj, onChange) {
  if (typeof obj !== 'object' || obj === null) {
    return obj;
  }

  // Ja e um Proxy
  if (obj.__isProxy) {
    return obj;
  }

  // Tornar propriedades existentes reativas
  for (const key in obj) {
    if (typeof obj[key] === 'object' && obj[key] !== null) {
      obj[key] = createDeepReactive(obj[key], onChange);
    }
  }

  return new Proxy(obj, {
    get(target, property) {
      if (property === '__isProxy') return true;
      return target[property];
    },

    set(target, property, value) {
      const oldValue = target[property];

      // Tornar novo valor reativo se for objeto
      if (typeof value === 'object' && value !== null) {
        value = createDeepReactive(value, onChange);
      }

      target[property] = value;

      if (oldValue !== value) {
        onChange(property, oldValue, value);
      }

      return true;
    },

    deleteProperty(target, property) {
      const oldValue = target[property];
      delete target[property];
      onChange(property, oldValue, undefined);
      return true;
    }
  });
}

class DeepReactiveElement extends HTMLElement {
  #state;

  constructor() {
    super();
    this.attachShadow({ mode: 'open' });

    this.#state = createDeepReactive({}, () => {
      this.render();
    });
  }

  get state() {
    return this.#state;
  }

  render() {
    // Override
  }
}
```

### Exemplo: Todo List Reativo

```javascript
class TodoList extends DeepReactiveElement {
  connectedCallback() {
    this.state.todos = [];
    this.state.filter = 'all'; // 'all', 'active', 'completed'

    this.render();
    this.#attachListeners();
  }

  #attachListeners() {
    this.shadowRoot.addEventListener('submit', (e) => {
      if (e.target.matches('form')) {
        e.preventDefault();
        const input = e.target.querySelector('input');
        if (input.value.trim()) {
          this.#addTodo(input.value.trim());
          input.value = '';
        }
      }
    });

    this.shadowRoot.addEventListener('change', (e) => {
      if (e.target.matches('.todo-checkbox')) {
        const id = parseInt(e.target.dataset.id);
        this.#toggleTodo(id);
      }
    });

    this.shadowRoot.addEventListener('click', (e) => {
      if (e.target.matches('.delete-btn')) {
        const id = parseInt(e.target.dataset.id);
        this.#deleteTodo(id);
      }

      if (e.target.matches('.filter-btn')) {
        this.state.filter = e.target.dataset.filter;
      }
    });
  }

  #addTodo(text) {
    this.state.todos.push({
      id: Date.now(),
      text,
      completed: false
    });
  }

  #toggleTodo(id) {
    const todo = this.state.todos.find(t => t.id === id);
    if (todo) {
      todo.completed = !todo.completed;
    }
  }

  #deleteTodo(id) {
    const index = this.state.todos.findIndex(t => t.id === id);
    if (index !== -1) {
      this.state.todos.splice(index, 1);
    }
  }

  #getFilteredTodos() {
    switch (this.state.filter) {
      case 'active':
        return this.state.todos.filter(t => !t.completed);
      case 'completed':
        return this.state.todos.filter(t => t.completed);
      default:
        return this.state.todos;
    }
  }

  render() {
    const filtered = this.#getFilteredTodos();
    const activeCount = this.state.todos.filter(t => !t.completed).length;

    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          max-width: 500px;
          margin: 0 auto;
          padding: 20px;
          font-family: system-ui;
        }
        h2 {
          text-align: center;
          color: #1f2937;
        }
        form {
          display: flex;
          gap: 8px;
          margin-bottom: 20px;
        }
        input[type="text"] {
          flex: 1;
          padding: 10px;
          border: 1px solid #d1d5db;
          border-radius: 6px;
          font-size: 14px;
        }
        button[type="submit"] {
          padding: 10px 20px;
          background: #3b82f6;
          color: white;
          border: none;
          border-radius: 6px;
          cursor: pointer;
        }
        .filters {
          display: flex;
          gap: 8px;
          margin-bottom: 16px;
          justify-content: center;
        }
        .filter-btn {
          padding: 6px 12px;
          border: 1px solid #d1d5db;
          background: white;
          border-radius: 4px;
          cursor: pointer;
        }
        .filter-btn.active {
          background: #3b82f6;
          color: white;
          border-color: #3b82f6;
        }
        .todo-item {
          display: flex;
          align-items: center;
          gap: 12px;
          padding: 12px;
          border: 1px solid #e5e7eb;
          border-radius: 6px;
          margin-bottom: 8px;
        }
        .todo-item.completed .todo-text {
          text-decoration: line-through;
          color: #9ca3af;
        }
        .todo-text {
          flex: 1;
        }
        .delete-btn {
          padding: 4px 12px;
          background: #ef4444;
          color: white;
          border: none;
          border-radius: 4px;
          cursor: pointer;
          font-size: 12px;
        }
        .stats {
          margin-top: 16px;
          text-align: center;
          color: #6b7280;
          font-size: 14px;
        }
        .empty {
          text-align: center;
          color: #9ca3af;
          padding: 40px;
        }
      </style>

      <h2>📝 Todo List Reativo</h2>

      <form>
        <input type="text" placeholder="Adicionar nova tarefa..." />
        <button type="submit">Adicionar</button>
      </form>

      <div class="filters">
        <button class="filter-btn ${this.state.filter === 'all' ? 'active' : ''}" data-filter="all">
          Todas
        </button>
        <button class="filter-btn ${this.state.filter === 'active' ? 'active' : ''}" data-filter="active">
          Ativas
        </button>
        <button class="filter-btn ${this.state.filter === 'completed' ? 'active' : ''}" data-filter="completed">
          Concluidas
        </button>
      </div>

      <div class="todos">
        ${filtered.length === 0
          ? '<div class="empty">Nenhuma tarefa encontrada</div>'
          : filtered.map(todo => `
              <div class="todo-item ${todo.completed ? 'completed' : ''}">
                <input
                  type="checkbox"
                  class="todo-checkbox"
                  data-id="${todo.id}"
                  ${todo.completed ? 'checked' : ''}
                />
                <span class="todo-text">${todo.text}</span>
                <button class="delete-btn" data-id="${todo.id}">Deletar</button>
              </div>
            `).join('')
        }
      </div>

      <div class="stats">
        ${activeCount} ${activeCount === 1 ? 'tarefa ativa' : 'tarefas ativas'}
      </div>
    `;
  }
}

customElements.define('todo-list', TodoList);
```

## Otimizacao: Debouncing e Batching

Evitar re-renderizacoes excessivas:

### Debounced Rendering

```javascript
class OptimizedElement extends HTMLElement {
  #state;
  #renderTimeout = null;

  constructor() {
    super();
    this.attachShadow({ mode: 'open' });

    this.#state = new Proxy({}, {
      set: (target, property, value) => {
        target[property] = value;
        this.#scheduleRender();
        return true;
      }
    });
  }

  #scheduleRender() {
    // Cancelar render anterior pendente
    if (this.#renderTimeout) {
      clearTimeout(this.#renderTimeout);
    }

    // Agendar novo render com debounce
    this.#renderTimeout = setTimeout(() => {
      this.render();
      this.#renderTimeout = null;
    }, 16); // ~60fps
  }

  get state() {
    return this.#state;
  }

  render() {
    // Override
  }
}
```

### Batching com requestAnimationFrame

```javascript
class BatchedElement extends HTMLElement {
  #state;
  #pendingRender = false;

  constructor() {
    super();
    this.attachShadow({ mode: 'open' });

    this.#state = new Proxy({}, {
      set: (target, property, value) => {
        target[property] = value;
        this.#requestRender();
        return true;
      }
    });
  }

  #requestRender() {
    if (this.#pendingRender) return;

    this.#pendingRender = true;

    requestAnimationFrame(() => {
      this.render();
      this.#pendingRender = false;
    });
  }

  get state() {
    return this.#state;
  }

  render() {
    // Override
  }
}
```

## Computed Values (Valores Derivados)

```javascript
class ComputedElement extends HTMLElement {
  #state;
  #computed = new Map();

  constructor() {
    super();
    this.attachShadow({ mode: 'open' });

    this.#state = new Proxy({}, {
      set: (target, property, value) => {
        target[property] = value;
        this.#invalidateComputed();
        this.render();
        return true;
      }
    });
  }

  get state() {
    return this.#state;
  }

  // Definir valor computado
  computed(name, fn) {
    this.#computed.set(name, {
      fn,
      cache: null,
      dirty: true
    });
  }

  // Obter valor computado (com cache)
  getComputed(name) {
    const computed = this.#computed.get(name);
    if (!computed) return undefined;

    if (computed.dirty) {
      computed.cache = computed.fn(this.#state);
      computed.dirty = false;
    }

    return computed.cache;
  }

  #invalidateComputed() {
    this.#computed.forEach(computed => {
      computed.dirty = true;
    });
  }

  render() {
    // Override
  }
}
```

### Exemplo com Computed

```javascript
class ShoppingCart extends ComputedElement {
  connectedCallback() {
    this.state.items = [];
    this.state.discountPercent = 0;

    // Definir valores computados
    this.computed('subtotal', (state) => {
      return state.items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
    });

    this.computed('discount', (state) => {
      const subtotal = this.getComputed('subtotal');
      return subtotal * (state.discountPercent / 100);
    });

    this.computed('total', () => {
      const subtotal = this.getComputed('subtotal');
      const discount = this.getComputed('discount');
      return subtotal - discount;
    });

    this.render();
  }

  addItem(name, price, quantity = 1) {
    this.state.items.push({ name, price, quantity });
  }

  render() {
    const subtotal = this.getComputed('subtotal');
    const discount = this.getComputed('discount');
    const total = this.getComputed('total');

    this.shadowRoot.innerHTML = `
      <style>
        :host { display: block; padding: 20px; font-family: system-ui; }
        .summary { background: #f9fafb; padding: 16px; border-radius: 8px; }
        .row { display: flex; justify-content: space-between; margin: 8px 0; }
        .total { font-size: 1.4em; font-weight: bold; color: #1f2937; border-top: 2px solid #d1d5db; padding-top: 8px; margin-top: 8px; }
      </style>
      <div class="summary">
        <h3>Carrinho</h3>
        <div class="items">
          ${this.state.items.map(item => `
            <div class="row">
              <span>${item.name} (x${item.quantity})</span>
              <span>R$ ${(item.price * item.quantity).toFixed(2)}</span>
            </div>
          `).join('')}
        </div>
        <div class="row">
          <span>Subtotal:</span>
          <span>R$ ${subtotal.toFixed(2)}</span>
        </div>
        ${discount > 0 ? `
          <div class="row" style="color: #22c55e;">
            <span>Desconto (${this.state.discountPercent}%):</span>
            <span>- R$ ${discount.toFixed(2)}</span>
          </div>
        ` : ''}
        <div class="row total">
          <span>Total:</span>
          <span>R$ ${total.toFixed(2)}</span>
        </div>
      </div>
    `;
  }
}

customElements.define('shopping-cart', ShoppingCart);
```

```html
<shopping-cart id="cart"></shopping-cart>

<script>
  const cart = document.getElementById('cart');

  cart.addItem('Notebook', 3500, 1);
  cart.addItem('Mouse', 80, 2);
  cart.addItem('Teclado', 350, 1);

  setTimeout(() => {
    cart.state.discountPercent = 10; // Aplicar 10% de desconto
  }, 2000);
</script>
```

## Watchers (Observadores)

```javascript
class WatchableElement extends HTMLElement {
  #state;
  #watchers = new Map();

  constructor() {
    super();
    this.attachShadow({ mode: 'open' });

    this.#state = new Proxy({}, {
      set: (target, property, value) => {
        const oldValue = target[property];
        target[property] = value;

        if (oldValue !== value) {
          this.#notifyWatchers(property, value, oldValue);
          this.render();
        }

        return true;
      }
    });
  }

  get state() {
    return this.#state;
  }

  // Observar mudancas em propriedade especifica
  watch(property, callback) {
    if (!this.#watchers.has(property)) {
      this.#watchers.set(property, new Set());
    }
    this.#watchers.get(property).add(callback);

    // Retornar unwatch function
    return () => {
      this.#watchers.get(property)?.delete(callback);
    };
  }

  #notifyWatchers(property, newValue, oldValue) {
    const callbacks = this.#watchers.get(property);
    if (!callbacks) return;

    callbacks.forEach(callback => {
      callback(newValue, oldValue);
    });
  }

  render() {
    // Override
  }
}
```

## Exercicios

1. Crie um componente `reactive-form` que valide campos em tempo real usando Proxy
2. Implemente um `data-grid` reativo com filtro e ordenacao
3. Crie um `real-time-chart` que atualiza automaticamente com novos dados
4. Implemente um sistema deundo/redo usando estado reativo
