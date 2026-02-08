# Capitulo 15: Referencia Rapida - Cheat Sheet

## Lifecycle Callbacks

```javascript
class MyElement extends HTMLElement {
  // Declarar atributos observados
  static observedAttributes = ['attr1', 'attr2'];

  // 1. Elemento criado/instanciado
  constructor() {
    super(); // OBRIGATORIO
    // Inicializar estado
    // Criar shadow root
    // NAO: ler atributos, modificar DOM
  }

  // 2. Elemento inserido no DOM
  connectedCallback() {
    // Ler atributos
    // Renderizar
    // Adicionar event listeners
    // Buscar dados
  }

  // 3. Elemento removido do DOM
  disconnectedCallback() {
    // Limpar: timers, listeners, fetches
  }

  // 4. Atributo observado mudou
  attributeChangedCallback(name, oldValue, newValue) {
    // Reagir a mudancas de atributos
  }

  // 5. Elemento movido para novo documento
  adoptedCallback() {
    // Raro: usado em iframes
  }
}
```

## Atributos e Propriedades

### Ler Atributos

```javascript
// Verificar existencia (boolean)
this.hasAttribute('disabled');

// Obter valor (string ou null)
this.getAttribute('value');

// Obter com default
this.getAttribute('type') || 'text';

// Obter numero
parseInt(this.getAttribute('max'), 10) || 100;

// Todos os atributos
this.getAttributeNames(); // ['id', 'class', ...]

// Tem algum atributo?
this.hasAttributes(); // boolean
```

### Modificar Atributos

```javascript
// Definir/atualizar
this.setAttribute('value', 'novo');

// Remover
this.removeAttribute('disabled');

// Alternar
this.toggleAttribute('active');
this.toggleAttribute('active', true); // force

// Data attributes
this.dataset.userId = '123'; // data-user-id="123"
console.log(this.dataset.userId);
delete this.dataset.userId;
```

### Sincronizar Atributo ↔ Propriedade

```javascript
// String
get value() {
  return this.getAttribute('value') || '';
}
set value(val) {
  this.setAttribute('value', String(val));
}

// Number
get count() {
  return parseInt(this.getAttribute('count'), 10) || 0;
}
set count(val) {
  this.setAttribute('count', String(val));
}

// Boolean
get disabled() {
  return this.hasAttribute('disabled');
}
set disabled(val) {
  this.toggleAttribute('disabled', Boolean(val));
}

// Enum
get size() {
  const value = this.getAttribute('size');
  return ['small', 'medium', 'large'].includes(value) ? value : 'medium';
}
set size(val) {
  const valid = ['small', 'medium', 'large'];
  if (valid.includes(val)) {
    this.setAttribute('size', val);
  }
}
```

## Shadow DOM

```javascript
// Criar
const shadow = this.attachShadow({ mode: 'open' });
// mode: 'open' (acessivel) ou 'closed' (nao acessivel)

// Acessar
element.shadowRoot; // se mode: 'open'

// Adicionar conteudo
shadow.innerHTML = `<style>...</style><div>...</div>`;

// Query dentro do shadow
shadow.querySelector('.class');
shadow.querySelectorAll('button');

// Verificar se elemento tem shadow
if (element.shadowRoot) { }
```

## Slots

```javascript
// Template com slots
shadow.innerHTML = `
  <slot name="header">Fallback Header</slot>
  <slot></slot> <!-- slot padrao -->
`;

// HTML de uso
<my-element>
  <h1 slot="header">Meu Titulo</h1>
  <p>Conteudo padrao</p>
</my-element>

// JavaScript: obter nos atribuidos
const slot = shadow.querySelector('slot[name="header"]');
slot.assignedNodes();        // todos os nos
slot.assignedElements();     // apenas elementos

// Evento quando conteudo muda
slot.addEventListener('slotchange', (e) => {
  const nodes = e.target.assignedNodes();
});
```

## Estilizacao

### Seletores Especiais

```css
/* O proprio elemento host */
:host {
  display: block;
}

/* Host com classe/atributo */
:host(.active) { }
:host([disabled]) { }

/* Host baseado em ancestral */
:host-context(.dark-theme) { }

/* Elementos projetados em slots */
::slotted(p) { }
::slotted(.class) { }

/* Partes expostas para estilizacao externa */
[part="button"] { }

/* Externa: estilizar parte */
my-element::part(button) { }
```

### CSS Custom Properties

```css
/* Definir (atravessa shadow boundary) */
:host {
  --primary-color: #3b82f6;
  --spacing: 16px;
}

/* Usar com fallback */
.button {
  background: var(--primary-color, blue);
  padding: var(--spacing, 8px);
}
```

### Constructable Stylesheets

```javascript
// Criar uma vez
const styles = new CSSStyleSheet();
styles.replaceSync(`
  :host { display: block; }
  .card { padding: 16px; }
`);

// Reutilizar em multiplos componentes
shadow.adoptedStyleSheets = [styles];

// Combinar com estilos locais
const localStyles = new CSSStyleSheet();
localStyles.replaceSync(`.extra { margin: 10px; }`);
shadow.adoptedStyleSheets = [styles, localStyles];
```

## Classes e Estilos

```javascript
// Classes
this.classList.add('active');
this.classList.remove('inactive');
this.classList.toggle('selected');
this.classList.contains('active');
this.classList.replace('old', 'new');

// Estilos inline
this.style.backgroundColor = '#3b82f6';
this.style.padding = '16px';
this.style.cssText = 'color: red; font-size: 14px;';

// CSS Custom Properties
this.style.setProperty('--color', 'red');
this.style.getPropertyValue('--color');
this.style.removeProperty('--color');
```

## Eventos

```javascript
// Disparar evento simples
this.dispatchEvent(new Event('my-event'));

// Custom Event com dados
this.dispatchEvent(new CustomEvent('item-selected', {
  detail: { id: 123, name: 'Item' },
  bubbles: true,     // propaga para cima
  composed: true,    // cruza shadow boundaries
  cancelable: true   // pode ser cancelado
}));

// Verificar se foi cancelado
const event = new CustomEvent('before-save', { cancelable: true });
const notCancelled = this.dispatchEvent(event);
if (!notCancelled) {
  console.log('Evento cancelado');
}

// Escutar
element.addEventListener('my-event', (e) => {
  console.log(e.detail);
});
```

## Form-Associated Elements

```javascript
class CustomInput extends HTMLElement {
  static formAssociated = true; // OBRIGATORIO

  constructor() {
    super();
    this._internals = this.attachInternals();
  }

  connectedCallback() {
    const input = this.shadowRoot.querySelector('input');

    input.addEventListener('input', (e) => {
      // Associar valor ao form
      this._internals.setFormValue(e.target.value);

      // Validacao
      if (!e.target.value && this.hasAttribute('required')) {
        this._internals.setValidity(
          { valueMissing: true },
          'Campo obrigatorio',
          input
        );
      } else {
        this._internals.setValidity({});
      }
    });
  }

  // Lifecycle do formulario
  formResetCallback() {
    this.shadowRoot.querySelector('input').value = '';
  }

  formDisabledCallback(disabled) {
    this.shadowRoot.querySelector('input').disabled = disabled;
  }

  // Propriedades de validacao
  get form() { return this._internals.form; }
  get validity() { return this._internals.validity; }
  checkValidity() { return this._internals.checkValidity(); }
}
```

## Reatividade com Proxy

```javascript
class ReactiveElement extends HTMLElement {
  constructor() {
    super();
    this._state = new Proxy({}, {
      set: (target, property, value) => {
        target[property] = value;
        this.render(); // Re-render automatico
        return true;
      }
    });
  }

  get state() {
    return this._state;
  }

  connectedCallback() {
    this.state.count = 0; // Dispara render
  }

  render() {
    this.shadowRoot.innerHTML = `
      <div>Count: ${this.state.count}</div>
    `;
  }
}
```

## Performance

```javascript
// Lazy Loading com Intersection Observer
const observer = new IntersectionObserver((entries) => {
  entries.forEach(async (entry) => {
    if (entry.isIntersecting) {
      const tagName = entry.target.tagName.toLowerCase();
      if (!customElements.get(tagName)) {
        await import(`./components/${tagName}.js`);
      }
      observer.unobserve(entry.target);
    }
  });
}, { rootMargin: '200px' });

observer.observe(element);

// Batch updates com requestAnimationFrame
scheduleRender() {
  if (this._pendingRender) return;
  this._pendingRender = true;

  requestAnimationFrame(() => {
    this.render();
    this._pendingRender = false;
  });
}

// Debounce
debounce(func, wait) {
  let timeout;
  return (...args) => {
    clearTimeout(timeout);
    timeout = setTimeout(() => func.apply(this, args), wait);
  };
}
```

## Patterns Comuns

### Observer Pattern

```javascript
class Observable {
  _observers = new Set();

  subscribe(callback) {
    this._observers.add(callback);
    return () => this._observers.delete(callback);
  }

  notify(data) {
    this._observers.forEach(cb => cb(data));
  }
}
```

### Mixin Pattern

```javascript
const LoggingMixin = (Base) => class extends Base {
  connectedCallback() {
    console.log(`[${this.tagName}] connected`);
    super.connectedCallback?.();
  }
};

class MyElement extends LoggingMixin(HTMLElement) {}
```

### Factory Pattern

```javascript
class ComponentFactory {
  static create(type, props = {}) {
    const element = document.createElement(type);
    Object.entries(props).forEach(([key, value]) => {
      element.setAttribute(key, value);
    });
    return element;
  }
}

const button = ComponentFactory.create('my-button', {
  variant: 'primary',
  size: 'large'
});
```

## Testes

```javascript
// Unit test
describe('MyElement', () => {
  let element;

  beforeEach(async () => {
    element = document.createElement('my-element');
    document.body.appendChild(element);
    await customElements.whenDefined('my-element');
  });

  afterEach(() => {
    element.remove();
  });

  it('should render', () => {
    expect(element.shadowRoot).toBeTruthy();
  });

  it('should handle click', () => {
    const button = element.shadowRoot.querySelector('button');
    button.click();
    expect(element.getAttribute('clicked')).toBe('true');
  });
});

// E2E com Playwright
test('should add item', async ({ page }) => {
  await page.goto('/');
  await page.locator('my-list input').fill('New item');
  await page.locator('my-list button').click();
  await expect(page.locator('my-list .item')).toHaveCount(1);
});
```

## Acessibilidade

```javascript
class AccessibleElement extends HTMLElement {
  constructor() {
    super();
    this._internals = this.attachInternals();

    // Definir role e ARIA
    this._internals.role = 'button';
    this._internals.ariaLabel = 'Meu Botao';
  }

  connectedCallback() {
    // Tornar focavel
    if (!this.hasAttribute('tabindex')) {
      this.setAttribute('tabindex', '0');
    }

    // Suporte a teclado
    this.addEventListener('keydown', (e) => {
      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        this.click();
      }
    });
  }
}
```

## SSR com Declarative Shadow DOM

```html
<my-card>
  <template shadowrootmode="open">
    <style>
      .card { border: 1px solid #ddd; padding: 16px; }
    </style>
    <div class="card">
      <slot name="title">Titulo</slot>
      <slot></slot>
    </div>
  </template>
  <h2 slot="title">Meu Card</h2>
  <p>Conteudo renderizado no servidor</p>
</my-card>
```

## Micro Frontends

```javascript
// Host application
<script type="module" src="https://cdn.team-a.com/mfe-header.js"></script>
<script type="module" src="https://cdn.team-b.com/mfe-catalog.js"></script>

<mfe-header></mfe-header>
<mfe-catalog></mfe-catalog>

// Comunicacao entre MFEs
window.addEventListener('cart:add-item', (e) => {
  console.log('Item adicionado:', e.detail);
});

window.dispatchEvent(new CustomEvent('cart:add-item', {
  detail: { product: {...} }
}));
```

## Armadilhas Comuns

```javascript
// ❌ ERRADO
class BadPractices extends HTMLElement {
  constructor() {
    super();
    // NAO ler atributos aqui
    this.value = this.getAttribute('value');

    // NAO adicionar filhos aqui
    this.innerHTML = '<div>content</div>';
  }

  attributeChangedCallback(name, old, new) {
    // Loop infinito!
    this.setAttribute(name, new.toUpperCase());

    // Assumir que shadowRoot existe
    this.shadowRoot.querySelector('div');

    // Re-render desnecessario
    this.render();
  }
}

// ✅ CERTO
class GoodPractices extends HTMLElement {
  constructor() {
    super();
    this._value = null;
    this.attachShadow({ mode: 'open' });
  }

  connectedCallback() {
    // Ler atributos aqui
    this._value = this.getAttribute('value');
    this.render();
  }

  attributeChangedCallback(name, old, new) {
    // Verificar mudanca
    if (old === new) return;

    // Verificar se conectado
    if (!this.isConnected) return;

    // Verificar se shadowRoot existe
    const el = this.shadowRoot?.querySelector('div');
    if (el) {
      el.textContent = new;
    }
  }
}
```

## Checklist de Boas Praticas

### Constructor
- [ ] Sempre chamar `super()` primeiro
- [ ] Criar shadow root aqui
- [ ] Inicializar estado/propriedades privadas
- [ ] NAO ler atributos
- [ ] NAO modificar children
- [ ] NAO fazer requisicoes

### connectedCallback
- [ ] Ler atributos
- [ ] Renderizar conteudo
- [ ] Adicionar event listeners
- [ ] Iniciar timers/polling
- [ ] Buscar dados externos
- [ ] Pode ser chamado multiplas vezes

### disconnectedCallback
- [ ] Limpar timers (clearInterval, clearTimeout)
- [ ] Abortar fetches (AbortController)
- [ ] Remover event listeners globais
- [ ] Desconectar observers

### attributeChangedCallback
- [ ] Declarar `static observedAttributes`
- [ ] Verificar `oldValue !== newValue`
- [ ] Verificar `isConnected` antes de DOM ops
- [ ] Nao criar loops (setAttribute no mesmo atributo)
- [ ] Pode ser chamado antes de connectedCallback

### Geral
- [ ] Usar nomes com hifen (kebab-case)
- [ ] Sincronizar atributos ↔ propriedades
- [ ] Custom events com `bubbles: true, composed: true`
- [ ] Limpar recursos em disconnectedCallback
- [ ] Suporte a acessibilidade (ARIA, keyboard)
- [ ] Testes unitarios e E2E

## Links Uteis

- [MDN Web Components](https://developer.mozilla.org/en-US/docs/Web/API/Web_components)
- [web.dev](https://web.dev/articles/custom-elements-v1)
- [WHATWG Spec](https://html.spec.whatwg.org/multipage/custom-elements.html)
- [Can I Use](https://caniuse.com/?search=web%20components)
