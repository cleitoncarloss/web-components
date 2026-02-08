# Capitulo 4: Ciclo de Vida e APIs

## Callbacks do Ciclo de Vida

Todo Custom Element tem 6 callbacks principais que o navegador chama automaticamente:

### 1. `constructor()`

Chamado quando:
- O elemento e criado via `document.createElement('meu-elemento')`
- O elemento e parseado no HTML e instanciado
- Um elemento existente e "upgraded" apos `customElements.define()`

**O que fazer:**
- ✅ Chamar `super()` na primeira linha
- ✅ Criar shadow root (`attachShadow`)
- ✅ Inicializar estado interno
- ✅ Criar referencias privadas

**O que NÃO fazer:**
- ❌ Ler atributos
- ❌ Acessar ou modificar children
- ❌ Fazer requisicoes HTTP
- ❌ Renderizar conteudo

```javascript
class MeuElemento extends HTMLElement {
  // Estado privado
  #state = { count: 0 };
  #shadow;

  constructor() {
    super(); // OBRIGATORIO

    // OK: criar shadow root
    this.#shadow = this.attachShadow({ mode: 'open' });

    // OK: setup inicial
    this._initialized = false;
  }
}
```

### 2. `connectedCallback()`

Chamado quando o elemento e inserido no DOM (pode ser chamado multiplas vezes se o elemento for movido).

**O que fazer:**
- ✅ Ler atributos
- ✅ Renderizar conteudo
- ✅ Buscar dados (fetch)
- ✅ Adicionar event listeners
- ✅ Iniciar timers/animations

```javascript
connectedCallback() {
  // Evitar re-execucao desnecessaria
  if (this._initialized) return;
  this._initialized = true;

  // Agora e seguro ler atributos
  const url = this.getAttribute('data-url');

  // Renderizar
  this.render();

  // Event listeners
  this.shadowRoot.querySelector('button')
    .addEventListener('click', this.handleClick);

  // Fetch de dados
  if (url) {
    this.fetchData(url);
  }
}
```

### 3. `disconnectedCallback()`

Chamado quando o elemento e removido do DOM.

**O que fazer:**
- ✅ Limpar event listeners
- ✅ Cancelar timers (clearInterval, clearTimeout)
- ✅ Abortar requisicoes (AbortController)
- ✅ Desconectar observers (IntersectionObserver, MutationObserver)

```javascript
#timerId = null;
#abortController = null;

connectedCallback() {
  // Iniciar polling
  this.#timerId = setInterval(() => this.update(), 5000);

  // Fetch cancelavel
  this.#abortController = new AbortController();
  fetch('/api/data', { signal: this.#abortController.signal })
    .then(res => res.json())
    .then(data => this.render(data));
}

disconnectedCallback() {
  // IMPORTANTE: cleanup para evitar memory leaks

  // Cancelar timer
  if (this.#timerId) {
    clearInterval(this.#timerId);
    this.#timerId = null;
  }

  // Abortar fetch pendente
  this.#abortController?.abort();

  // Remover listeners globais
  window.removeEventListener('resize', this.handleResize);
  document.removeEventListener('keydown', this.handleKey);
}
```

### 4. `attributeChangedCallback(name, oldValue, newValue)`

Chamado quando um atributo observado muda. Este e um dos callbacks mais importantes e poderosos.

**Importante**: Somente atributos listados em `observedAttributes` disparam este callback.

#### Anatomia Completa

```javascript
class DetailedAttributeExample extends HTMLElement {
  // 1. DECLARAR atributos observados (OBRIGATORIO)
  static observedAttributes = ['color', 'size', 'disabled', 'count'];

  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
  }

  connectedCallback() {
    this.render();
  }

  /**
   * attributeChangedCallback e chamado quando:
   * - Atributo e adicionado: oldValue = null, newValue = 'valor'
   * - Atributo e modificado: oldValue = 'antigo', newValue = 'novo'
   * - Atributo e removido: oldValue = 'valor', newValue = null
   *
   * @param {string} name - Nome do atributo que mudou
   * @param {string|null} oldValue - Valor anterior (null se nao existia)
   * @param {string|null} newValue - Novo valor (null se foi removido)
   */
  attributeChangedCallback(name, oldValue, newValue) {
    console.log(`Atributo "${name}" mudou:`);
    console.log(`  Antes: ${oldValue}`);
    console.log(`  Depois: ${newValue}`);

    // IMPORTANTE: Pode ser chamado ANTES de connectedCallback!
    // Verificar se o elemento ja esta conectado
    if (!this.isConnected) {
      console.log('  (elemento ainda nao conectado)');
      return;
    }

    // IMPORTANTE: Evitar loops infinitos
    // NAO chame setAttribute para o mesmo atributo aqui!
    if (oldValue === newValue) {
      console.log('  (valores identicos, ignorando)');
      return;
    }

    // Processar mudanca por atributo
    switch (name) {
      case 'color':
        this.#handleColorChange(newValue);
        break;
      case 'size':
        this.#handleSizeChange(newValue);
        break;
      case 'disabled':
        this.#handleDisabledChange(newValue !== null);
        break;
      case 'count':
        this.#handleCountChange(parseInt(newValue, 10) || 0);
        break;
    }
  }

  #handleColorChange(color) {
    const box = this.shadowRoot.querySelector('.box');
    if (box) {
      box.style.backgroundColor = color || 'gray';
    }
  }

  #handleSizeChange(size) {
    const box = this.shadowRoot.querySelector('.box');
    if (box) {
      const sizes = { small: '50px', medium: '100px', large: '150px' };
      box.style.width = sizes[size] || sizes.medium;
      box.style.height = sizes[size] || sizes.medium;
    }
  }

  #handleDisabledChange(disabled) {
    const box = this.shadowRoot.querySelector('.box');
    if (box) {
      box.style.opacity = disabled ? '0.5' : '1';
      box.style.pointerEvents = disabled ? 'none' : 'auto';
    }
  }

  #handleCountChange(count) {
    const countEl = this.shadowRoot.querySelector('.count');
    if (countEl) {
      countEl.textContent = count;
    }
  }

  render() {
    this.shadowRoot.innerHTML = `
      <style>
        .box {
          width: 100px;
          height: 100px;
          background: gray;
          display: flex;
          align-items: center;
          justify-content: center;
          border-radius: 8px;
          transition: all 0.3s;
        }
        .count {
          font-size: 24px;
          font-weight: bold;
          color: white;
        }
      </style>
      <div class="box">
        <div class="count">0</div>
      </div>
    `;

    // Aplicar valores iniciais dos atributos
    const color = this.getAttribute('color');
    const size = this.getAttribute('size');
    const disabled = this.hasAttribute('disabled');
    const count = this.getAttribute('count');

    if (color) this.#handleColorChange(color);
    if (size) this.#handleSizeChange(size);
    if (disabled) this.#handleDisabledChange(true);
    if (count) this.#handleCountChange(parseInt(count, 10));
  }
}

customElements.define('detailed-attribute-example', DetailedAttributeExample);
```

**Uso HTML:**

```html
<!-- Atributos iniciais -->
<detailed-attribute-example
  color="blue"
  size="large"
  count="5">
</detailed-attribute-example>

<script>
  const el = document.querySelector('detailed-attribute-example');

  // Modificar via JavaScript
  setTimeout(() => {
    el.setAttribute('color', 'red');     // attributeChangedCallback chamado
    el.setAttribute('count', '10');      // attributeChangedCallback chamado
    el.setAttribute('disabled', '');     // attributeChangedCallback chamado
  }, 2000);

  setTimeout(() => {
    el.removeAttribute('disabled');      // attributeChangedCallback chamado
    el.setAttribute('size', 'small');    // attributeChangedCallback chamado
  }, 4000);
</script>
```

#### Sincronizando Atributos e Propriedades

A melhor pratica e manter atributos HTML sincronizados com propriedades JavaScript:

```javascript
class SyncedElement extends HTMLElement {
  static observedAttributes = ['value', 'disabled', 'placeholder', 'type'];

  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
  }

  // ============================================
  // PROPRIEDADE: value (string)
  // ============================================
  get value() {
    return this.getAttribute('value') || '';
  }

  set value(val) {
    // Converter para string
    const stringValue = String(val);

    // Atualizar atributo (dispara attributeChangedCallback)
    if (stringValue) {
      this.setAttribute('value', stringValue);
    } else {
      this.removeAttribute('value');
    }
  }

  // ============================================
  // PROPRIEDADE: disabled (boolean)
  // ============================================
  get disabled() {
    return this.hasAttribute('disabled');
  }

  set disabled(val) {
    // Converter para boolean
    const boolValue = Boolean(val);

    if (boolValue) {
      // Atributo booleano: presenca = true
      this.setAttribute('disabled', '');
    } else {
      this.removeAttribute('disabled');
    }
  }

  // ============================================
  // PROPRIEDADE: placeholder (string)
  // ============================================
  get placeholder() {
    return this.getAttribute('placeholder') || '';
  }

  set placeholder(val) {
    if (val) {
      this.setAttribute('placeholder', String(val));
    } else {
      this.removeAttribute('placeholder');
    }
  }

  // ============================================
  // PROPRIEDADE: type (string com validacao)
  // ============================================
  get type() {
    return this.getAttribute('type') || 'text';
  }

  set type(val) {
    const validTypes = ['text', 'email', 'password', 'number', 'tel'];
    const type = String(val).toLowerCase();

    if (validTypes.includes(type)) {
      this.setAttribute('type', type);
    } else {
      console.warn(`Tipo invalido: ${val}. Usando 'text'.`);
      this.setAttribute('type', 'text');
    }
  }

  // ============================================
  // LIFECYCLE: attributeChangedCallback
  // ============================================
  attributeChangedCallback(name, oldValue, newValue) {
    if (oldValue === newValue) return;

    // Atualizar input interno quando atributos mudam
    const input = this.shadowRoot?.querySelector('input');
    if (!input) return;

    switch (name) {
      case 'value':
        input.value = newValue || '';
        break;
      case 'disabled':
        input.disabled = newValue !== null;
        break;
      case 'placeholder':
        input.placeholder = newValue || '';
        break;
      case 'type':
        input.type = newValue || 'text';
        break;
    }
  }

  connectedCallback() {
    this.shadowRoot.innerHTML = `
      <style>
        input {
          padding: 8px 12px;
          border: 1px solid #d1d5db;
          border-radius: 6px;
          font-size: 14px;
          width: 100%;
        }
        input:disabled {
          background: #f3f4f6;
          cursor: not-allowed;
        }
      </style>
      <input type="text" />
    `;

    // Sincronizar valores iniciais
    const input = this.shadowRoot.querySelector('input');
    input.value = this.value;
    input.disabled = this.disabled;
    input.placeholder = this.placeholder;
    input.type = this.type;

    // Sincronizar input interno com propriedade value
    input.addEventListener('input', (e) => {
      // Atualizar propriedade (que atualiza atributo)
      this.value = e.target.value;
    });
  }
}

customElements.define('synced-element', SyncedElement);
```

**Testando a sincronizacao:**

```javascript
const el = document.querySelector('synced-element');

// Todas essas formas funcionam e ficam sincronizadas:

// 1. Via propriedade JavaScript
el.value = 'novo valor';
console.log(el.getAttribute('value')); // 'novo valor'

// 2. Via atributo HTML
el.setAttribute('value', 'outro valor');
console.log(el.value); // 'outro valor'

// 3. Boolean attributes
el.disabled = true;
console.log(el.hasAttribute('disabled')); // true

el.disabled = false;
console.log(el.hasAttribute('disabled')); // false

// 4. Remover atributo
el.removeAttribute('value');
console.log(el.value); // ''

// 5. Validacao automatica
el.type = 'email';     // OK
el.type = 'invalid';   // Console warning, usa 'text'
```

#### Patterns de Atributos Comuns

```javascript
class AttributePatterns extends HTMLElement {
  static observedAttributes = [
    'text-value',      // String simples
    'number-value',    // Numero
    'bool-flag',       // Boolean
    'enum-value',      // Enum (valores fixos)
    'json-data',       // JSON
    'csv-list'         // Lista CSV
  ];

  attributeChangedCallback(name, oldValue, newValue) {
    if (oldValue === newValue) return;

    switch (name) {
      // Pattern 1: String simples
      case 'text-value':
        this.textValue = newValue || '';
        break;

      // Pattern 2: Numero
      case 'number-value':
        this.numberValue = newValue ? parseFloat(newValue) : 0;
        if (isNaN(this.numberValue)) {
          console.warn('number-value deve ser um numero valido');
          this.numberValue = 0;
        }
        break;

      // Pattern 3: Boolean (presenca = true)
      case 'bool-flag':
        this.boolFlag = newValue !== null;
        break;

      // Pattern 4: Enum (valores fixos)
      case 'enum-value':
        const validValues = ['small', 'medium', 'large'];
        if (validValues.includes(newValue)) {
          this.enumValue = newValue;
        } else {
          console.warn(`enum-value deve ser: ${validValues.join(', ')}`);
          this.enumValue = 'medium'; // default
        }
        break;

      // Pattern 5: JSON
      case 'json-data':
        try {
          this.jsonData = newValue ? JSON.parse(newValue) : null;
        } catch (e) {
          console.error('json-data deve ser JSON valido:', e);
          this.jsonData = null;
        }
        break;

      // Pattern 6: Lista CSV
      case 'csv-list':
        this.csvList = newValue
          ? newValue.split(',').map(item => item.trim())
          : [];
        break;
    }

    this.render();
  }

  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.render();
  }

  render() {
    this.shadowRoot.innerHTML = `
      <style>
        .info { padding: 12px; background: #f9fafb; border-radius: 6px; margin-bottom: 8px; }
        .label { font-weight: bold; }
      </style>
      <div class="info">
        <div><span class="label">Text:</span> ${this.textValue || 'N/A'}</div>
        <div><span class="label">Number:</span> ${this.numberValue || 0}</div>
        <div><span class="label">Boolean:</span> ${this.boolFlag ? 'true' : 'false'}</div>
        <div><span class="label">Enum:</span> ${this.enumValue || 'N/A'}</div>
        <div><span class="label">JSON:</span> ${JSON.stringify(this.jsonData)}</div>
        <div><span class="label">CSV:</span> ${this.csvList?.join(', ') || 'N/A'}</div>
      </div>
    `;
  }
}

customElements.define('attribute-patterns', AttributePatterns);
```

**Uso:**

```html
<attribute-patterns
  text-value="Hello World"
  number-value="42.5"
  bool-flag
  enum-value="large"
  json-data='{"name":"John","age":30}'
  csv-list="apple, banana, orange">
</attribute-patterns>

<script>
  const el = document.querySelector('attribute-patterns');

  // Testar patterns
  setTimeout(() => {
    el.setAttribute('number-value', '99.9');
    el.setAttribute('enum-value', 'small');
    el.removeAttribute('bool-flag');
    el.setAttribute('csv-list', 'red, green, blue, yellow');
    el.setAttribute('json-data', '{"city":"NYC","temp":25}');
  }, 2000);
</script>
```

#### Armadilhas Comuns (Evite!)

```javascript
class AttributePitfalls extends HTMLElement {
  static observedAttributes = ['value'];

  attributeChangedCallback(name, oldValue, newValue) {
    // ❌ ERRADO: Loop infinito!
    // this.setAttribute('value', newValue.toUpperCase());

    // ✅ CERTO: Guardar em propriedade interna
    this._value = newValue;

    // ❌ ERRADO: Assumir que shadowRoot existe
    // this.shadowRoot.querySelector('.value').textContent = newValue;

    // ✅ CERTO: Verificar primeiro
    const el = this.shadowRoot?.querySelector('.value');
    if (el) {
      el.textContent = newValue;
    }

    // ❌ ERRADO: Nao verificar se valores sao iguais
    // this.render(); // Re-render desnecessario

    // ✅ CERTO: Verificar mudanca
    if (oldValue !== newValue) {
      this.render();
    }
  }
}
```

### 5. `adoptedCallback()`

Chamado quando o elemento e movido para um novo documento (raro -- geralmente via `document.adoptNode()` ou iframes).

```javascript
adoptedCallback() {
  console.log('Elemento adotado por novo documento');
  // Re-inicializar referencias ao documento, se necessario
}
```

### 6. `connectedMoveCallback()` (novo)

Chamado quando o elemento e movido no DOM via API `moveBefore()` (substitui o par `disconnectedCallback` + `connectedCallback`).

```javascript
connectedMoveCallback() {
  // Otimizacao: elemento movido mas nao removido/re-inserido
  // Estado pode ser preservado
  console.log('Elemento reposicionado no DOM');
}
```

## Ordem de Execucao

```javascript
class LifecycleDemo extends HTMLElement {
  static observedAttributes = ['color'];

  constructor() {
    super();
    console.log('1. constructor()');
    this.attachShadow({ mode: 'open' });
  }

  attributeChangedCallback(name, oldValue, newValue) {
    console.log('2. attributeChangedCallback()', name, oldValue, newValue);
    // Pode ser chamado ANTES de connectedCallback se atributo esta no HTML
  }

  connectedCallback() {
    console.log('3. connectedCallback()');
    this.render();
  }

  disconnectedCallback() {
    console.log('4. disconnectedCallback()');
  }

  render() {
    this.shadowRoot.innerHTML = `<p style="color: ${this.getAttribute('color')}">Hello</p>`;
  }
}

customElements.define('lifecycle-demo', LifecycleDemo);
```

```html
<lifecycle-demo color="red"></lifecycle-demo>
```

Console:
```
1. constructor()
2. attributeChangedCallback() color null red
3. connectedCallback()
```

Se depois fizermos:
```javascript
document.querySelector('lifecycle-demo').remove();
```

Console:
```
4. disconnectedCallback()
```

## ElementInternals API

A API `ElementInternals` fornece recursos avancados para Custom Elements:

### Form Association

```javascript
class CustomInput extends HTMLElement {
  // OBRIGATORIO para form-associated elements
  static formAssociated = true;

  #internals;

  constructor() {
    super();
    this.#internals = this.attachInternals();
    this.attachShadow({ mode: 'open' });
  }

  connectedCallback() {
    this.shadowRoot.innerHTML = `
      <style>
        input { padding: 8px; border: 1px solid #ccc; border-radius: 4px; width: 100%; }
        input:invalid { border-color: red; }
      </style>
      <input type="text" />
    `;

    const input = this.shadowRoot.querySelector('input');

    input.addEventListener('input', (e) => {
      const value = e.target.value;

      // Associar valor ao form
      this.#internals.setFormValue(value);

      // Validacao
      this.validate(value);
    });
  }

  validate(value) {
    if (!value && this.hasAttribute('required')) {
      this.#internals.setValidity(
        { valueMissing: true },
        'Este campo e obrigatorio',
        this.shadowRoot.querySelector('input')
      );
    } else if (value.length < 3) {
      this.#internals.setValidity(
        { tooShort: true },
        'Minimo 3 caracteres',
        this.shadowRoot.querySelector('input')
      );
    } else {
      // Valido
      this.#internals.setValidity({});
    }
  }

  // Form lifecycle callbacks
  formAssociatedCallback(form) {
    console.log('Associado ao form:', form);
  }

  formResetCallback() {
    this.shadowRoot.querySelector('input').value = '';
    this.#internals.setFormValue('');
  }

  formDisabledCallback(disabled) {
    this.shadowRoot.querySelector('input').disabled = disabled;
  }

  formStateRestoreCallback(state, mode) {
    // mode: 'restore' (navegacao) ou 'autocomplete' (preenchimento)
    this.shadowRoot.querySelector('input').value = state;
    this.#internals.setFormValue(state);
  }

  // Expor propriedades de validacao
  get form() { return this.#internals.form; }
  get validity() { return this.#internals.validity; }
  get validationMessage() { return this.#internals.validationMessage; }
  get willValidate() { return this.#internals.willValidate; }

  checkValidity() { return this.#internals.checkValidity(); }
  reportValidity() { return this.#internals.reportValidity(); }
}

customElements.define('custom-input', CustomInput);
```

```html
<form id="meuForm">
  <label>Nome:
    <custom-input name="nome" required></custom-input>
  </label>
  <button type="submit">Enviar</button>
</form>

<script>
  document.getElementById('meuForm').addEventListener('submit', (e) => {
    e.preventDefault();
    const formData = new FormData(e.target);
    console.log('Nome:', formData.get('nome'));
  });
</script>
```

### Custom States

```javascript
class ToggleButton extends HTMLElement {
  #internals;

  constructor() {
    super();
    this.#internals = this.attachInternals();
    this.attachShadow({ mode: 'open' });
  }

  connectedCallback() {
    this.shadowRoot.innerHTML = `
      <style>
        button {
          padding: 10px 20px;
          border: 2px solid #ccc;
          background: white;
          border-radius: 4px;
          cursor: pointer;
          transition: all 0.3s;
        }
        /* Estado customizado usando :state() */
        :host(:state(on)) button {
          background: #22c55e;
          border-color: #16a34a;
          color: white;
        }
      </style>
      <button><slot>Toggle</slot></button>
    `;

    this.shadowRoot.querySelector('button').addEventListener('click', () => {
      this.toggle();
    });
  }

  get on() {
    return this.#internals.states.has('on');
  }

  toggle() {
    if (this.on) {
      this.#internals.states.delete('on');
    } else {
      this.#internals.states.add('on');
    }

    this.dispatchEvent(new CustomEvent('toggle', {
      detail: { on: this.on },
      bubbles: true,
      composed: true
    }));
  }
}

customElements.define('toggle-button', ToggleButton);
```

CSS externo pode acessar o estado:

```css
toggle-button:state(on) {
  box-shadow: 0 0 10px rgba(34, 197, 94, 0.5);
}
```

### Acessibilidade com ElementInternals

```javascript
class AccessibleButton extends HTMLElement {
  #internals;

  constructor() {
    super();
    this.#internals = this.attachInternals();

    // Definir role e propriedades ARIA
    this.#internals.role = 'button';
    this.#internals.ariaLabel = 'Botao customizado';
  }

  connectedCallback() {
    if (!this.hasAttribute('tabindex')) {
      this.setAttribute('tabindex', '0');
    }

    this.addEventListener('keydown', (e) => {
      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        this.click();
      }
    });
  }

  set ariaLabel(value) {
    this.#internals.ariaLabel = value;
  }
}

customElements.define('accessible-button', AccessibleButton);
```

## Metodos e Propriedades Importantes

### `hasAttribute()` e `getAttribute()`

```javascript
class AttributeMethods extends HTMLElement {
  connectedCallback() {
    // hasAttribute() - Verifica se atributo existe (retorna boolean)
    if (this.hasAttribute('required')) {
      console.log('Campo e obrigatorio');
    }

    // getAttribute() - Obtem valor do atributo (retorna string ou null)
    const placeholder = this.getAttribute('placeholder');
    console.log('Placeholder:', placeholder); // 'Digite aqui' ou null

    // Atributos booleanos
    const disabled = this.hasAttribute('disabled');
    console.log('Disabled:', disabled); // true ou false

    // Valor padrao se atributo nao existir
    const type = this.getAttribute('type') || 'text';
    const max = parseInt(this.getAttribute('max'), 10) || 100;
  }
}
```

### `setAttribute()` e `removeAttribute()`

```javascript
class SetAttributeExample extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <button>Toggle</button>
    `;

    const button = this.shadowRoot.querySelector('button');

    button.addEventListener('click', () => {
      // setAttribute() - Define/atualiza atributo
      if (this.hasAttribute('active')) {
        // removeAttribute() - Remove atributo
        this.removeAttribute('active');
      } else {
        this.setAttribute('active', '');
      }

      // Multiplos atributos
      this.setAttribute('data-clicks', this.clicks++);
      this.setAttribute('last-clicked', new Date().toISOString());
    });
  }

  clicks = 0;
}
```

### `toggleAttribute()`

```javascript
class ToggleExample extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `<button>Toggle</button>`;

    this.shadowRoot.querySelector('button').addEventListener('click', () => {
      // toggleAttribute() - Alterna presenca do atributo
      this.toggleAttribute('active');

      // Com segundo parametro (force)
      const shouldBeDisabled = Math.random() > 0.5;
      this.toggleAttribute('disabled', shouldBeDisabled);
    });
  }
}
```

### `getAttributeNames()`

```javascript
class AttributeInspector extends HTMLElement {
  connectedCallback() {
    // getAttributeNames() - Retorna array com nomes de todos os atributos
    const attrNames = this.getAttributeNames();
    console.log('Atributos:', attrNames);
    // ['id', 'class', 'data-value', 'title']

    // Iterar sobre todos os atributos
    attrNames.forEach(name => {
      const value = this.getAttribute(name);
      console.log(`${name} = "${value}"`);
    });

    // Copiar todos os atributos para outro elemento
    const clone = document.createElement('div');
    attrNames.forEach(name => {
      clone.setAttribute(name, this.getAttribute(name));
    });
  }
}
```

### `hasAttributes()`

```javascript
class HasAttributesExample extends HTMLElement {
  connectedCallback() {
    // hasAttributes() - Verifica se tem ALGUM atributo
    if (this.hasAttributes()) {
      console.log('Elemento tem atributos');
      console.log('Total:', this.getAttributeNames().length);
    } else {
      console.log('Elemento sem atributos');
    }
  }
}
```

### `dataset` (data-* attributes)

```javascript
class DatasetExample extends HTMLElement {
  static observedAttributes = ['data-user-id', 'data-user-name'];

  connectedCallback() {
    // Acessar data attributes via dataset
    console.log('User ID:', this.dataset.userId);       // data-user-id
    console.log('User Name:', this.dataset.userName);   // data-user-name

    // Modificar data attributes
    this.dataset.lastSeen = new Date().toISOString();   // data-last-seen
    this.dataset.status = 'online';                     // data-status

    // Verificar existencia
    if ('userId' in this.dataset) {
      console.log('User ID exists');
    }

    // Deletar
    delete this.dataset.status;

    // IMPORTANTE: dataset usa camelCase, mas HTML usa kebab-case
    this.dataset.myLongAttributeName = 'value';
    // Cria: data-my-long-attribute-name="value"
  }

  attributeChangedCallback(name, oldValue, newValue) {
    // data-* attributes tambem disparam este callback se observados
    if (name.startsWith('data-')) {
      console.log(`Data attribute mudou: ${name}`);
    }
  }
}
```

### `isConnected`

```javascript
class ConnectedCheck extends HTMLElement {
  constructor() {
    super();
    console.log('Constructor - isConnected:', this.isConnected); // false
  }

  connectedCallback() {
    console.log('Connected - isConnected:', this.isConnected); // true

    // Util para verificar antes de operacoes DOM
    if (this.isConnected) {
      this.startPolling();
    }
  }

  disconnectedCallback() {
    console.log('Disconnected - isConnected:', this.isConnected); // false
  }

  someAsyncMethod() {
    setTimeout(() => {
      // Verificar se ainda esta conectado antes de atualizar
      if (this.isConnected) {
        this.render();
      } else {
        console.log('Elemento foi removido, cancelando render');
      }
    }, 1000);
  }
}
```

### `matches()`, `closest()`, `querySelector()`

```javascript
class SelectorMethods extends HTMLElement {
  connectedCallback() {
    // matches() - Verifica se elemento corresponde a seletor CSS
    if (this.matches('.active')) {
      console.log('Elemento tem classe active');
    }

    if (this.matches('[data-type="button"]')) {
      console.log('Elemento e um botao');
    }

    // closest() - Encontra ancestral mais proximo que corresponde ao seletor
    const form = this.closest('form');
    const container = this.closest('.container');

    // querySelector() - Busca dentro do elemento (Light DOM)
    const header = this.querySelector('h1');
    const buttons = this.querySelectorAll('button');

    // Buscar no Shadow DOM
    const shadowButton = this.shadowRoot?.querySelector('button');
  }
}
```

### `classList` (manipulacao de classes)

```javascript
class ClassListExample extends HTMLElement {
  connectedCallback() {
    // add() - Adicionar classe
    this.classList.add('active');
    this.classList.add('primary', 'large'); // Multiplas classes

    // remove() - Remover classe
    this.classList.remove('inactive');

    // toggle() - Alternar classe
    this.classList.toggle('selected');
    this.classList.toggle('hidden', false); // Force remove

    // contains() - Verificar se tem classe
    if (this.classList.contains('active')) {
      console.log('Elemento esta ativo');
    }

    // replace() - Substituir classe
    this.classList.replace('old-class', 'new-class');

    // Iterar sobre classes
    this.classList.forEach(className => {
      console.log('Classe:', className);
    });

    // Numero de classes
    console.log('Total de classes:', this.classList.length);
  }
}
```

### `style` (manipulacao de estilos inline)

```javascript
class StyleManipulation extends HTMLElement {
  connectedCallback() {
    // Definir estilos individuais
    this.style.backgroundColor = '#3b82f6';
    this.style.color = 'white';
    this.style.padding = '16px';
    this.style.borderRadius = '8px';

    // Propriedades CSS com hifen usam camelCase
    this.style.fontSize = '16px';        // font-size
    this.style.marginTop = '20px';       // margin-top
    this.style.boxShadow = '0 2px 4px rgba(0,0,0,0.1)'; // box-shadow

    // cssText - Definir multiplos estilos de uma vez
    this.style.cssText = `
      background: linear-gradient(45deg, #667eea, #764ba2);
      color: white;
      padding: 20px;
      border-radius: 12px;
    `;

    // getPropertyValue() - Ler valor de propriedade
    const bgColor = this.style.getPropertyValue('background-color');

    // setProperty() - Definir com prioridade
    this.style.setProperty('color', 'red', 'important');

    // removeProperty() - Remover propriedade
    this.style.removeProperty('background-color');

    // CSS Custom Properties (variaveis)
    this.style.setProperty('--primary-color', '#3b82f6');
    const primaryColor = this.style.getPropertyValue('--primary-color');
  }
}
```

### `dispatchEvent()` e Custom Events

```javascript
class EventDispatcher extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `<button>Click Me</button>`;

    this.shadowRoot.querySelector('button').addEventListener('click', () => {
      // Evento simples
      this.dispatchEvent(new Event('button-clicked'));

      // Custom Event com dados
      this.dispatchEvent(new CustomEvent('item-selected', {
        detail: {
          itemId: 123,
          itemName: 'Product A',
          timestamp: Date.now()
        },
        bubbles: true,    // Propaga para cima no DOM
        composed: true,   // Cruza shadow boundaries
        cancelable: true  // Pode ser cancelado com preventDefault()
      }));

      // Verificar se evento foi cancelado
      const event = new CustomEvent('before-submit', {
        cancelable: true,
        bubbles: true,
        composed: true
      });

      const wasNotCancelled = this.dispatchEvent(event);
      if (wasNotCancelled) {
        console.log('Evento nao foi cancelado, continuar');
      } else {
        console.log('Evento foi cancelado');
      }
    });
  }
}
```

### `cloneNode()` e `replaceWith()`

```javascript
class CloneExample extends HTMLElement {
  connectedCallback() {
    // cloneNode(false) - Clone superficial (sem filhos)
    const shallowClone = this.cloneNode(false);

    // cloneNode(true) - Clone profundo (com todos os filhos)
    const deepClone = this.cloneNode(true);

    // Clonar preserva atributos mas NAO event listeners
    console.log('Atributos clonados:', deepClone.getAttributeNames());

    // replaceWith() - Substituir elemento
    const replacement = document.createElement('div');
    replacement.textContent = 'Novo elemento';
    // this.replaceWith(replacement); // Este elemento sera removido e substituido
  }
}
```

## Exemplo Completo: Componente com Todos os Conceitos

```javascript
class ComprehensiveComponent extends HTMLElement {
  // 1. Observar atributos
  static observedAttributes = ['value', 'disabled', 'type', 'data-id'];

  // 2. Propriedades privadas
  #internals;
  #shadowRoot;
  #value = '';

  constructor() {
    super();
    console.log('Constructor - isConnected:', this.isConnected);

    this.#shadowRoot = this.attachShadow({ mode: 'open' });
    this.#internals = this.attachInternals();

    // Definir role
    this.#internals.role = 'textbox';
  }

  // 3. Lifecycle: connectedCallback
  connectedCallback() {
    console.log('Connected - isConnected:', this.isConnected);
    console.log('Atributos iniciais:', this.getAttributeNames());

    this.render();
    this.#attachEventListeners();

    // Aplicar classes
    this.classList.add('custom-input');
  }

  // 4. Lifecycle: disconnectedCallback
  disconnectedCallback() {
    console.log('Disconnected');
    // Cleanup aqui
  }

  // 5. Lifecycle: attributeChangedCallback
  attributeChangedCallback(name, oldValue, newValue) {
    console.log(`Atributo "${name}": "${oldValue}" → "${newValue}"`);

    if (oldValue === newValue) return;
    if (!this.isConnected) return;

    switch (name) {
      case 'value':
        this.#handleValueChange(newValue);
        break;
      case 'disabled':
        this.#handleDisabledChange(newValue !== null);
        break;
      case 'type':
        this.#handleTypeChange(newValue);
        break;
      case 'data-id':
        console.log('Data ID:', this.dataset.id);
        break;
    }
  }

  // 6. Getters e Setters
  get value() {
    return this.#value;
  }

  set value(val) {
    this.#value = String(val);
    this.setAttribute('value', this.#value);
  }

  get disabled() {
    return this.hasAttribute('disabled');
  }

  set disabled(val) {
    this.toggleAttribute('disabled', Boolean(val));
  }

  // 7. Metodos privados
  #handleValueChange(value) {
    this.#value = value || '';
    const input = this.#shadowRoot.querySelector('input');
    if (input) input.value = this.#value;
  }

  #handleDisabledChange(disabled) {
    const input = this.#shadowRoot.querySelector('input');
    if (input) input.disabled = disabled;

    // Modificar estilos
    this.style.opacity = disabled ? '0.5' : '1';
  }

  #handleTypeChange(type) {
    const input = this.#shadowRoot.querySelector('input');
    if (input) input.type = type || 'text';
  }

  #attachEventListeners() {
    const input = this.#shadowRoot.querySelector('input');

    input.addEventListener('input', (e) => {
      this.#value = e.target.value;
      this.setAttribute('value', this.#value);

      // Disparar custom event
      this.dispatchEvent(new CustomEvent('value-change', {
        detail: { value: this.#value },
        bubbles: true,
        composed: true
      }));
    });
  }

  // 8. Render
  render() {
    this.#shadowRoot.innerHTML = `
      <style>
        :host {
          display: inline-block;
        }
        :host([disabled]) {
          opacity: 0.5;
          pointer-events: none;
        }
        input {
          padding: 8px 12px;
          border: 1px solid #d1d5db;
          border-radius: 6px;
          font-size: 14px;
        }
      </style>
      <input
        type="${this.getAttribute('type') || 'text'}"
        value="${this.getAttribute('value') || ''}"
        placeholder="${this.getAttribute('placeholder') || ''}"
        ${this.hasAttribute('disabled') ? 'disabled' : ''}
      />
    `;
  }
}

customElements.define('comprehensive-component', ComprehensiveComponent);
```

**Uso completo:**

```html
<comprehensive-component
  id="myInput"
  class="form-control"
  type="email"
  value="test@example.com"
  placeholder="Digite seu email"
  data-id="123"
  data-user-name="João">
</comprehensive-component>

<script>
  const input = document.getElementById('myInput');

  // Testar metodos e propriedades
  console.log('Tem atributos?', input.hasAttributes());
  console.log('Atributos:', input.getAttributeNames());
  console.log('Value:', input.value);
  console.log('Dataset ID:', input.dataset.id);
  console.log('Dataset userName:', input.dataset.userName);
  console.log('Disabled?', input.disabled);
  console.log('Tem classe form-control?', input.classList.contains('form-control'));

  // Modificar
  setTimeout(() => {
    input.value = 'novo@email.com';
    input.disabled = true;
    input.classList.add('error');
    input.style.borderColor = 'red';
    input.setAttribute('data-error', 'true');
  }, 2000);

  // Escutar evento customizado
  input.addEventListener('value-change', (e) => {
    console.log('Valor mudou:', e.detail.value);
  });
</script>
```

## Exercicios

1. Crie um componente que faca polling de uma API a cada 5 segundos e limpe corretamente o timer em `disconnectedCallback`
2. Crie um form-associated custom element que valide um CPF brasileiro
3. Crie um componente que use custom states (`:state()`) para representar diferentes estados (loading, success, error)
4. Crie um componente que observe 5 atributos diferentes (string, number, boolean, enum, json) e sincronize com propriedades JavaScript
5. Implemente um componente que use `classList`, `style`, `dataset` e `dispatchEvent` em conjunto
