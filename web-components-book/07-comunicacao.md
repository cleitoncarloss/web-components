# Capitulo 7: Comunicacao entre Componentes

## Estrategias de Comunicacao

Web Components podem se comunicar de varias formas:

1. **Atributos e Propriedades** -- Pai → Filho (one-way down)
2. **Custom Events** -- Filho → Pai (one-way up)
3. **Event Bus / PubSub** -- Qualquer componente ↔ Qualquer componente
4. **Shared State** -- Estado global compartilhado
5. **Context API** -- Provedor → Consumidores

## 1. Comunicacao Pai → Filho (Atributos e Propriedades)

### Via Atributos HTML

```javascript
class ChildComponent extends HTMLElement {
  static observedAttributes = ['message', 'count'];

  attributeChangedCallback(name, oldValue, newValue) {
    console.log(`${name} mudou: ${oldValue} → ${newValue}`);
    this.render();
  }

  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.render();
  }

  render() {
    this.shadowRoot.innerHTML = `
      <div>
        <p>Message: ${this.getAttribute('message') || 'Nenhuma'}</p>
        <p>Count: ${this.getAttribute('count') || '0'}</p>
      </div>
    `;
  }
}

customElements.define('child-component', ChildComponent);
```

```html
<child-component message="Ola" count="5"></child-component>

<script>
  const child = document.querySelector('child-component');

  // Atualizar via JavaScript
  child.setAttribute('message', 'Nova mensagem');
  child.setAttribute('count', '10');
</script>
```

### Via Propriedades JavaScript

```javascript
class DataDisplay extends HTMLElement {
  #data = null;

  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.render();
  }

  // Getter/Setter para propriedade complexa
  get data() {
    return this.#data;
  }

  set data(value) {
    this.#data = value;
    if (this.isConnected) {
      this.render();
    }
  }

  render() {
    if (!this.#data) {
      this.shadowRoot.innerHTML = '<p>Carregando...</p>';
      return;
    }

    this.shadowRoot.innerHTML = `
      <style>
        .card { padding: 16px; border: 1px solid #ddd; border-radius: 8px; }
        h3 { margin: 0 0 8px; }
        p { margin: 4px 0; color: #666; }
      </style>
      <div class="card">
        <h3>${this.#data.title}</h3>
        <p>${this.#data.description}</p>
        <p><strong>Items:</strong> ${this.#data.items?.length || 0}</p>
      </div>
    `;
  }
}

customElements.define('data-display', DataDisplay);
```

```javascript
const display = document.querySelector('data-display');

// Passar objeto complexo via propriedade JavaScript
display.data = {
  title: 'Meu Titulo',
  description: 'Descricao detalhada',
  items: [1, 2, 3, 4, 5]
};
```

## 2. Comunicacao Filho → Pai (Custom Events)

### Custom Events Basicos

```javascript
class ButtonCounter extends HTMLElement {
  #count = 0;

  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        button {
          padding: 10px 20px;
          font-size: 16px;
          cursor: pointer;
        }
        .count {
          margin-left: 12px;
          font-weight: bold;
        }
      </style>
      <button>Incrementar</button>
      <span class="count">0</span>
    `;

    const button = this.shadowRoot.querySelector('button');
    const countEl = this.shadowRoot.querySelector('.count');

    button.addEventListener('click', () => {
      this.#count++;
      countEl.textContent = this.#count;

      // Disparar custom event
      this.dispatchEvent(new CustomEvent('count-changed', {
        detail: { count: this.#count },
        bubbles: true,      // Propaga para cima no DOM
        composed: true      // Cruza shadow boundaries
      }));
    });
  }
}

customElements.define('button-counter', ButtonCounter);
```

```html
<div id="parent">
  <h2>Total: <span id="total">0</span></h2>
  <button-counter></button-counter>
  <button-counter></button-counter>
  <button-counter></button-counter>
</div>

<script>
  const parent = document.getElementById('parent');
  const totalEl = document.getElementById('total');
  let total = 0;

  // Escutar eventos dos filhos
  parent.addEventListener('count-changed', (e) => {
    console.log('Contador mudou:', e.detail.count);
    total = Array.from(parent.querySelectorAll('button-counter'))
      .reduce((sum, counter) => sum + (counter._count || 0), 0);
    totalEl.textContent = total;
  });
</script>
```

### Evento com Prevencao

```javascript
class ConfirmDialog extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        .overlay {
          position: fixed;
          inset: 0;
          background: rgba(0,0,0,0.5);
          display: flex;
          align-items: center;
          justify-content: center;
        }
        .dialog {
          background: white;
          padding: 24px;
          border-radius: 8px;
          min-width: 300px;
        }
        .buttons {
          margin-top: 16px;
          display: flex;
          gap: 8px;
          justify-content: flex-end;
        }
        button {
          padding: 8px 16px;
          border: none;
          border-radius: 4px;
          cursor: pointer;
        }
        .confirm { background: #3b82f6; color: white; }
        .cancel { background: #e5e7eb; }
      </style>
      <div class="overlay">
        <div class="dialog">
          <h3><slot name="title">Confirmar</slot></h3>
          <p><slot></slot></p>
          <div class="buttons">
            <button class="cancel">Cancelar</button>
            <button class="confirm">Confirmar</button>
          </div>
        </div>
      </div>
    `;

    this.shadowRoot.querySelector('.cancel').addEventListener('click', () => {
      // Evento cancelavel
      const event = new CustomEvent('cancel', {
        cancelable: true,
        bubbles: true,
        composed: true
      });

      if (this.dispatchEvent(event)) {
        // Se nao foi prevented, fechar
        this.close();
      }
    });

    this.shadowRoot.querySelector('.confirm').addEventListener('click', () => {
      const event = new CustomEvent('confirm', {
        cancelable: true,
        bubbles: true,
        composed: true
      });

      if (this.dispatchEvent(event)) {
        this.close();
      }
    });
  }

  close() {
    this.remove();
  }
}

customElements.define('confirm-dialog', ConfirmDialog);
```

```html
<button id="deleteBtn">Deletar</button>

<script>
  document.getElementById('deleteBtn').addEventListener('click', () => {
    const dialog = document.createElement('confirm-dialog');
    dialog.innerHTML = `
      <span slot="title">Deletar item</span>
      Tem certeza que deseja deletar este item?
    `;

    dialog.addEventListener('confirm', (e) => {
      console.log('Usuario confirmou');
      // Executar acao de delete
    });

    dialog.addEventListener('cancel', (e) => {
      console.log('Usuario cancelou');
      // Poderia prevenir o fechamento:
      // e.preventDefault();
    });

    document.body.appendChild(dialog);
  });
</script>
```

## 3. Event Bus (PubSub Pattern)

Sistema global de mensagens para comunicacao desacoplada:

```javascript
// event-bus.js
class EventBus {
  static #instance = null;
  #listeners = new Map();

  static getInstance() {
    if (!EventBus.#instance) {
      EventBus.#instance = new EventBus();
    }
    return EventBus.#instance;
  }

  on(event, callback) {
    if (!this.#listeners.has(event)) {
      this.#listeners.set(event, new Set());
    }
    this.#listeners.get(event).add(callback);

    // Retornar funcao de unsubscribe
    return () => {
      this.#listeners.get(event)?.delete(callback);
    };
  }

  once(event, callback) {
    const unsubscribe = this.on(event, (...args) => {
      callback(...args);
      unsubscribe();
    });
    return unsubscribe;
  }

  emit(event, data) {
    const listeners = this.#listeners.get(event);
    if (!listeners) return;

    listeners.forEach(callback => {
      try {
        callback(data);
      } catch (error) {
        console.error(`Error in listener for "${event}":`, error);
      }
    });
  }

  off(event, callback) {
    if (callback) {
      this.#listeners.get(event)?.delete(callback);
    } else {
      this.#listeners.delete(event);
    }
  }

  clear() {
    this.#listeners.clear();
  }
}

// Expor instancia global
window.eventBus = EventBus.getInstance();
```

### Componente Publisher

```javascript
class NotificationSender extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        button { padding: 10px 20px; cursor: pointer; }
      </style>
      <button>Enviar Notificacao</button>
    `;

    this.shadowRoot.querySelector('button').addEventListener('click', () => {
      // Publicar evento no bus
      window.eventBus.emit('notification', {
        type: 'info',
        message: 'Nova notificacao enviada!',
        timestamp: Date.now()
      });
    });
  }
}

customElements.define('notification-sender', NotificationSender);
```

### Componente Subscriber

```javascript
class NotificationReceiver extends HTMLElement {
  #unsubscribe = null;

  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        .notifications {
          border: 1px solid #ddd;
          border-radius: 8px;
          padding: 16px;
          max-height: 300px;
          overflow-y: auto;
        }
        .notification {
          padding: 8px;
          margin-bottom: 8px;
          border-radius: 4px;
          background: #eff6ff;
          border-left: 3px solid #3b82f6;
        }
        .time {
          font-size: 12px;
          color: #666;
        }
      </style>
      <div class="notifications">
        <p>Aguardando notificacoes...</p>
      </div>
    `;

    // Inscrever no event bus
    this.#unsubscribe = window.eventBus.on('notification', (data) => {
      this.#addNotification(data);
    });
  }

  disconnectedCallback() {
    // IMPORTANTE: limpar inscricao para evitar memory leak
    this.#unsubscribe?.();
  }

  #addNotification(data) {
    const container = this.shadowRoot.querySelector('.notifications');

    // Remover mensagem inicial
    const placeholder = container.querySelector('p');
    if (placeholder) placeholder.remove();

    const notification = document.createElement('div');
    notification.className = 'notification';
    notification.innerHTML = `
      <div>${data.message}</div>
      <div class="time">${new Date(data.timestamp).toLocaleTimeString()}</div>
    `;

    container.insertBefore(notification, container.firstChild);

    // Limitar a 10 notificacoes
    const items = container.querySelectorAll('.notification');
    if (items.length > 10) {
      items[items.length - 1].remove();
    }
  }
}

customElements.define('notification-receiver', NotificationReceiver);
```

```html
<notification-sender></notification-sender>
<notification-receiver></notification-receiver>
<notification-receiver></notification-receiver>
<!-- Ambos receivers recebem as mesmas notificacoes -->
```

## 4. Shared State (Estado Compartilhado)

### Store Simples com Proxy

```javascript
// store.js
class Store {
  #state;
  #listeners = new Set();

  constructor(initialState = {}) {
    this.#state = new Proxy(initialState, {
      set: (target, property, value) => {
        const oldValue = target[property];
        target[property] = value;

        if (oldValue !== value) {
          this.#notify(property, value, oldValue);
        }

        return true;
      }
    });
  }

  get state() {
    return this.#state;
  }

  subscribe(callback) {
    this.#listeners.add(callback);

    // Retornar unsubscribe
    return () => {
      this.#listeners.delete(callback);
    };
  }

  #notify(property, newValue, oldValue) {
    this.#listeners.forEach(callback => {
      callback({ property, newValue, oldValue, state: this.#state });
    });
  }

  setState(updates) {
    Object.assign(this.#state, updates);
  }
}

// Store global
window.appStore = new Store({
  user: null,
  cart: [],
  theme: 'light'
});
```

### Componentes conectados ao Store

```javascript
class UserDisplay extends HTMLElement {
  #unsubscribe = null;

  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.render();

    // Inscrever no store
    this.#unsubscribe = window.appStore.subscribe(({ property }) => {
      if (property === 'user') {
        this.render();
      }
    });
  }

  disconnectedCallback() {
    this.#unsubscribe?.();
  }

  render() {
    const user = window.appStore.state.user;

    this.shadowRoot.innerHTML = `
      <style>
        .user { padding: 12px; background: #f9fafb; border-radius: 8px; }
      </style>
      <div class="user">
        ${user
          ? `<strong>${user.name}</strong> (${user.email})`
          : 'Nenhum usuario logado'
        }
      </div>
    `;
  }
}

class LoginForm extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        form { display: flex; flex-direction: column; gap: 8px; max-width: 300px; }
        input { padding: 8px; border: 1px solid #ccc; border-radius: 4px; }
        button { padding: 10px; background: #3b82f6; color: white; border: none; border-radius: 4px; cursor: pointer; }
      </style>
      <form>
        <input type="text" name="name" placeholder="Nome" required />
        <input type="email" name="email" placeholder="Email" required />
        <button type="submit">Login</button>
      </form>
    `;

    this.shadowRoot.querySelector('form').addEventListener('submit', (e) => {
      e.preventDefault();

      const formData = new FormData(e.target);
      const user = {
        name: formData.get('name'),
        email: formData.get('email')
      };

      // Atualizar store -- todos os componentes inscritos serao notificados
      window.appStore.state.user = user;

      e.target.reset();
    });
  }
}

class CartDisplay extends HTMLElement {
  #unsubscribe = null;

  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.render();

    this.#unsubscribe = window.appStore.subscribe(({ property }) => {
      if (property === 'cart') {
        this.render();
      }
    });
  }

  disconnectedCallback() {
    this.#unsubscribe?.();
  }

  render() {
    const cart = window.appStore.state.cart;

    this.shadowRoot.innerHTML = `
      <style>
        .cart { padding: 12px; background: #f9fafb; border-radius: 8px; }
        .badge { background: #ef4444; color: white; padding: 2px 8px; border-radius: 12px; font-size: 12px; }
      </style>
      <div class="cart">
        🛒 Carrinho
        <span class="badge">${cart.length}</span>
      </div>
    `;
  }
}

customElements.define('user-display', UserDisplay);
customElements.define('login-form', LoginForm);
customElements.define('cart-display', CartDisplay);
```

```html
<user-display></user-display>
<cart-display></cart-display>
<login-form></login-form>

<script>
  // Qualquer codigo pode atualizar o store
  setTimeout(() => {
    window.appStore.state.cart = ['Item 1', 'Item 2', 'Item 3'];
  }, 3000);
</script>
```

## 5. Context API (Provedor/Consumidor)

Pattern para passar dados atraves de niveis de componentes:

```javascript
// context.js
const contexts = new WeakMap();

function createContext(defaultValue) {
  return {
    Provider: class extends HTMLElement {
      #value = defaultValue;

      connectedCallback() {
        contexts.set(this, this.#value);
      }

      get value() {
        return this.#value;
      }

      set value(val) {
        this.#value = val;
        contexts.set(this, val);

        // Notificar consumidores
        this.dispatchEvent(new CustomEvent('context-change', {
          bubbles: true,
          composed: true,
          detail: { value: val }
        }));
      }
    },

    Consumer: class extends HTMLElement {
      #renderCallback = null;

      connectedCallback() {
        this.#findProvider();

        // Escutar mudancas
        this.addEventListener('context-change', () => {
          this.#findProvider();
        });
      }

      #findProvider() {
        let element = this.parentElement;

        while (element) {
          const value = contexts.get(element);
          if (value !== undefined) {
            this.#renderCallback?.(value);
            return;
          }
          element = element.parentElement;
        }

        // Usar valor padrao
        this.#renderCallback?.(defaultValue);
      }

      render(callback) {
        this.#renderCallback = callback;
        this.#findProvider();
      }
    }
  };
}

// Criar contexto de tema
const ThemeContext = createContext('light');
customElements.define('theme-provider', ThemeContext.Provider);
customElements.define('theme-consumer', ThemeContext.Consumer);
```

```html
<theme-provider id="provider">
  <h1>App com Tema</h1>

  <theme-consumer id="header"></theme-consumer>
  <theme-consumer id="content"></theme-consumer>

  <button id="toggleTheme">Alternar Tema</button>
</theme-provider>

<script>
  const provider = document.getElementById('provider');
  provider.value = 'light';

  const header = document.getElementById('header');
  header.render((theme) => {
    header.innerHTML = `<h2>Header - Tema: ${theme}</h2>`;
  });

  const content = document.getElementById('content');
  content.render((theme) => {
    content.innerHTML = `<p>Conteudo - Tema: ${theme}</p>`;
  });

  document.getElementById('toggleTheme').addEventListener('click', () => {
    provider.value = provider.value === 'light' ? 'dark' : 'light';
  });
</script>
```

## Resumo de Estrategias

| Estrategia | Caso de Uso | Acoplamento | Complexidade |
|---|---|---|---|
| Atributos/Props | Pai → Filho direto | Alto | Baixo |
| Custom Events | Filho → Pai/Ancestral | Baixo | Baixo |
| Event Bus | Qualquer ↔ Qualquer | Muito Baixo | Medio |
| Shared State | Estado global | Medio | Medio |
| Context API | Provedor → Descendentes | Baixo | Alto |

## Exercicios

1. Crie um sistema de carrinho de compras com `product-card` (envia eventos) e `cart-summary` (recebe eventos)
2. Implemente um Event Bus e crie 3 componentes que se comunicam por ele
3. Crie um Store reativo e conecte 4 componentes diferentes a ele
4. Implemente um sistema de tema (light/dark) usando Context API
