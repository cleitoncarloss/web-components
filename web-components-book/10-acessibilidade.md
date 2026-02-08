# Capitulo 10: Acessibilidade

## O Desafio: Shadow DOM e ARIA

O Shadow DOM cria uma barreira para referencias de ID usadas por atributos ARIA como `aria-labelledby`, `aria-describedby` e o atributo `for` de `<label>`.

### O Problema

```html
<!-- ISTO NAO FUNCIONA -->
<label for="my-input">Nome:</label>
<custom-input>
  #shadow-root
    <input id="my-input" type="text" />
  /#shadow-root
</custom-input>
```

O `label` no Light DOM nao pode referenciar o `input` dentro do Shadow DOM.

## Solucoes

### 1. Manter Elementos no Mesmo Escopo

```javascript
class AccessibleInput extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });

    // Label e input no MESMO shadow DOM
    shadow.innerHTML = `
      <style>
        .wrapper { display: flex; flex-direction: column; gap: 4px; }
        label { font-weight: 500; font-size: 14px; }
        input { padding: 8px; border: 1px solid #ccc; border-radius: 4px; }
      </style>
      <div class="wrapper">
        <label for="internal-input">
          <slot name="label">Campo</slot>
        </label>
        <input id="internal-input" type="text" />
      </div>
    `;
  }
}

customElements.define('accessible-input', AccessibleInput);
```

```html
<accessible-input>
  <span slot="label">Nome completo</span>
</accessible-input>
```

### 2. Usar `aria-label` em Vez de ID

```javascript
class LabeledInput extends HTMLElement {
  static observedAttributes = ['label'];

  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `
      <style>
        input { padding: 8px; border: 1px solid #ccc; border-radius: 4px; width: 100%; }
      </style>
      <input type="text" />
    `;

    this.#updateLabel();
  }

  attributeChangedCallback() {
    this.#updateLabel();
  }

  #updateLabel() {
    const input = this.shadowRoot?.querySelector('input');
    if (input) {
      const label = this.getAttribute('label');
      if (label) {
        input.setAttribute('aria-label', label);
      }
    }
  }
}

customElements.define('labeled-input', LabeledInput);
```

```html
<labeled-input label="Nome do usuario"></labeled-input>
```

### 3. Envolver Slots

Projetar conteudo do Light DOM em slots permite que as referencias funcionem:

```javascript
class FormField extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });

    shadow.innerHTML = `
      <style>
        .field { display: flex; flex-direction: column; gap: 8px; margin-bottom: 16px; }
        .label-wrapper { font-weight: 500; }
        .input-wrapper { /* styles */ }
        .hint { font-size: 12px; color: #666; }
      </style>
      <div class="field">
        <div class="label-wrapper">
          <slot name="label"></slot>
        </div>
        <div class="input-wrapper">
          <slot name="input"></slot>
        </div>
        <div class="hint">
          <slot name="hint"></slot>
        </div>
      </div>
    `;
  }
}

customElements.define('form-field', FormField);
```

```html
<!-- Label e input ficam no Light DOM, portanto podem se referenciar -->
<form-field>
  <label slot="label" for="username">Usuario</label>
  <input slot="input" type="text" id="username" />
  <span slot="hint">Minimo 3 caracteres</span>
</form-field>
```

## ElementInternals para ARIA

Use `ElementInternals` para definir role e propriedades ARIA:

```javascript
class AccessibleButton extends HTMLElement {
  #internals;

  constructor() {
    super();
    this.#internals = this.attachInternals();

    // Definir role padrao
    this.#internals.role = 'button';
  }

  connectedCallback() {
    this.attachShadow({ mode: 'open' });

    // Tornar focavel
    if (!this.hasAttribute('tabindex')) {
      this.setAttribute('tabindex', '0');
    }

    // Aplicar label
    const label = this.getAttribute('aria-label');
    if (label) {
      this.#internals.ariaLabel = label;
    }

    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: inline-flex;
          align-items: center;
          justify-content: center;
          padding: 10px 20px;
          background: #3b82f6;
          color: white;
          border-radius: 6px;
          cursor: pointer;
          user-select: none;
        }
        :host(:focus) {
          outline: 2px solid #3b82f6;
          outline-offset: 2px;
        }
        :host([disabled]) {
          opacity: 0.5;
          cursor: not-allowed;
          pointer-events: none;
        }
      </style>
      <slot></slot>
    `;

    // Suporte a teclado
    this.addEventListener('keydown', (e) => {
      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        this.click();
      }
    });
  }

  static get observedAttributes() {
    return ['aria-label', 'aria-pressed', 'disabled'];
  }

  attributeChangedCallback(name, oldValue, newValue) {
    switch (name) {
      case 'aria-label':
        this.#internals.ariaLabel = newValue;
        break;
      case 'aria-pressed':
        this.#internals.ariaPressed = newValue;
        break;
      case 'disabled':
        this.#internals.ariaDisabled = newValue !== null;
        break;
    }
  }
}

customElements.define('accessible-button', AccessibleButton);
```

## Navegacao por Teclado

### Tab Order e Focus Management

```javascript
class TabPanel extends HTMLElement {
  #currentIndex = 0;

  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });

    shadow.innerHTML = `
      <style>
        :host { display: block; }
        .tablist {
          display: flex;
          border-bottom: 2px solid #e5e7eb;
          gap: 4px;
        }
        .tab {
          padding: 12px 20px;
          background: transparent;
          border: none;
          border-bottom: 2px solid transparent;
          cursor: pointer;
          font-size: 14px;
          color: #6b7280;
          transition: all 0.2s;
        }
        .tab:hover {
          color: #1f2937;
          background: #f9fafb;
        }
        .tab:focus {
          outline: 2px solid #3b82f6;
          outline-offset: -2px;
        }
        .tab[aria-selected="true"] {
          color: #3b82f6;
          border-bottom-color: #3b82f6;
          font-weight: 600;
        }
        .panels { padding: 20px 0; }
        .panel { display: none; }
        .panel[aria-hidden="false"] { display: block; }
      </style>
      <div role="tablist" class="tablist">
        <button role="tab" aria-selected="true" tabindex="0">Tab 1</button>
        <button role="tab" aria-selected="false" tabindex="-1">Tab 2</button>
        <button role="tab" aria-selected="false" tabindex="-1">Tab 3</button>
      </div>
      <div class="panels">
        <div role="tabpanel" class="panel" aria-hidden="false">
          <slot name="panel-1">Conteudo Tab 1</slot>
        </div>
        <div role="tabpanel" class="panel" aria-hidden="true">
          <slot name="panel-2">Conteudo Tab 2</slot>
        </div>
        <div role="tabpanel" class="panel" aria-hidden="true">
          <slot name="panel-3">Conteudo Tab 3</slot>
        </div>
      </div>
    `;

    const tablist = shadow.querySelector('[role="tablist"]');
    const tabs = shadow.querySelectorAll('[role="tab"]');
    const panels = shadow.querySelectorAll('[role="tabpanel"]');

    // Navegacao com setas
    tablist.addEventListener('keydown', (e) => {
      let newIndex = this.#currentIndex;

      if (e.key === 'ArrowRight') {
        newIndex = (this.#currentIndex + 1) % tabs.length;
      } else if (e.key === 'ArrowLeft') {
        newIndex = (this.#currentIndex - 1 + tabs.length) % tabs.length;
      } else if (e.key === 'Home') {
        newIndex = 0;
      } else if (e.key === 'End') {
        newIndex = tabs.length - 1;
      } else {
        return;
      }

      e.preventDefault();
      this.#selectTab(newIndex, tabs, panels);
    });

    // Click nas tabs
    tabs.forEach((tab, index) => {
      tab.addEventListener('click', () => {
        this.#selectTab(index, tabs, panels);
      });
    });
  }

  #selectTab(index, tabs, panels) {
    this.#currentIndex = index;

    // Atualizar tabs
    tabs.forEach((tab, i) => {
      const isSelected = i === index;
      tab.setAttribute('aria-selected', isSelected);
      tab.tabIndex = isSelected ? 0 : -1;

      if (isSelected) {
        tab.focus();
      }
    });

    // Atualizar panels
    panels.forEach((panel, i) => {
      panel.setAttribute('aria-hidden', i !== index);
    });

    // Dispatch evento
    this.dispatchEvent(new CustomEvent('tab-change', {
      detail: { index },
      bubbles: true
    }));
  }
}

customElements.define('tab-panel', TabPanel);
```

### Dialog/Modal Acessivel

```javascript
class AccessibleDialog extends HTMLElement {
  #previousFocus = null;

  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });

    shadow.innerHTML = `
      <style>
        .overlay {
          position: fixed;
          inset: 0;
          background: rgba(0, 0, 0, 0.5);
          display: flex;
          align-items: center;
          justify-content: center;
          z-index: 1000;
        }
        .dialog {
          background: white;
          border-radius: 8px;
          padding: 24px;
          max-width: 500px;
          width: 90%;
          max-height: 80vh;
          overflow-y: auto;
          box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
        }
        .dialog:focus {
          outline: none;
        }
        .header {
          display: flex;
          justify-content: space-between;
          align-items: center;
          margin-bottom: 16px;
        }
        .close-btn {
          background: none;
          border: none;
          font-size: 24px;
          cursor: pointer;
          padding: 4px;
          line-height: 1;
          color: #6b7280;
        }
        .close-btn:hover { color: #1f2937; }
      </style>
      <div class="overlay" role="dialog" aria-modal="true" aria-labelledby="dialog-title">
        <div class="dialog" tabindex="-1">
          <div class="header">
            <h2 id="dialog-title"><slot name="title">Dialog</slot></h2>
            <button class="close-btn" aria-label="Fechar" title="Fechar">×</button>
          </div>
          <div class="content">
            <slot></slot>
          </div>
        </div>
      </div>
    `;

    // Salvar foco anterior
    this.#previousFocus = document.activeElement;

    // Focar no dialog
    const dialog = shadow.querySelector('.dialog');
    dialog.focus();

    // Trap focus dentro do dialog
    this.#setupFocusTrap(shadow);

    // Botao de fechar
    shadow.querySelector('.close-btn').addEventListener('click', () => {
      this.close();
    });

    // Fechar com ESC
    this.addEventListener('keydown', (e) => {
      if (e.key === 'Escape') {
        this.close();
      }
    });

    // Fechar ao clicar no overlay
    shadow.querySelector('.overlay').addEventListener('click', (e) => {
      if (e.target.classList.contains('overlay')) {
        this.close();
      }
    });
  }

  #setupFocusTrap(shadow) {
    const dialog = shadow.querySelector('.dialog');
    const focusableElements = dialog.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );

    if (focusableElements.length === 0) return;

    const firstElement = focusableElements[0];
    const lastElement = focusableElements[focusableElements.length - 1];

    dialog.addEventListener('keydown', (e) => {
      if (e.key !== 'Tab') return;

      if (e.shiftKey) {
        // Shift + Tab
        if (shadow.activeElement === firstElement) {
          e.preventDefault();
          lastElement.focus();
        }
      } else {
        // Tab
        if (shadow.activeElement === lastElement) {
          e.preventDefault();
          firstElement.focus();
        }
      }
    });
  }

  close() {
    // Restaurar foco
    this.#previousFocus?.focus();

    this.dispatchEvent(new CustomEvent('close', { bubbles: true }));
    this.remove();
  }
}

customElements.define('accessible-dialog', AccessibleDialog);
```

## Live Regions

Anunciar mudancas dinamicas para screen readers:

```javascript
class LiveNotification extends HTMLElement {
  constructor() {
    super();
    this.attachInternals();
  }

  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });

    // aria-live="polite" aguarda pausa na leitura
    // aria-live="assertive" interrompe imediatamente
    const priority = this.getAttribute('priority') || 'polite';

    shadow.innerHTML = `
      <style>
        :host {
          position: fixed;
          bottom: 20px;
          right: 20px;
          max-width: 400px;
          z-index: 9999;
        }
        .notification {
          background: white;
          border-left: 4px solid #3b82f6;
          padding: 16px;
          border-radius: 8px;
          box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
          animation: slideIn 0.3s ease;
        }
        @keyframes slideIn {
          from {
            transform: translateX(100%);
            opacity: 0;
          }
          to {
            transform: translateX(0);
            opacity: 1;
          }
        }
        .success { border-left-color: #22c55e; }
        .warning { border-left-color: #f59e0b; }
        .error { border-left-color: #ef4444; }
      </style>
      <div
        class="notification ${this.getAttribute('type') || ''}"
        role="status"
        aria-live="${priority}"
        aria-atomic="true"
      >
        <slot></slot>
      </div>
    `;

    // Auto-remover apos timeout
    const duration = parseInt(this.getAttribute('duration'), 10) || 5000;
    if (duration > 0) {
      setTimeout(() => this.remove(), duration);
    }
  }
}

customElements.define('live-notification', LiveNotification);
```

```javascript
// Uso
function showNotification(message, type = 'info', priority = 'polite') {
  const notification = document.createElement('live-notification');
  notification.setAttribute('type', type);
  notification.setAttribute('priority', priority);
  notification.textContent = message;
  document.body.appendChild(notification);
}

showNotification('Dados salvos com sucesso!', 'success');
showNotification('Erro ao carregar dados', 'error', 'assertive');
```

## Checklist de Acessibilidade

### ✅ Semantica e ARIA

- [ ] Usar roles ARIA apropriados (`role="button"`, `role="dialog"`, etc)
- [ ] Definir `aria-label` ou `aria-labelledby` para elementos interativos
- [ ] Usar `aria-describedby` para descricoes adicionais
- [ ] Definir `aria-live` para conteudo dinamico
- [ ] Usar `aria-hidden` para esconder conteudo decorativo
- [ ] Definir `aria-expanded`, `aria-pressed` para estados
- [ ] Usar `aria-required`, `aria-invalid` em forms

### ✅ Teclado

- [ ] Todos os elementos interativos sao focaveis (`tabindex="0"`)
- [ ] Suporte para `Enter` e `Space` em botoes customizados
- [ ] Navegacao com setas em listas, tabs, menus
- [ ] `Home`/`End` para ir ao inicio/fim
- [ ] `Escape` para fechar modals/dropdowns
- [ ] Focus trap em modals
- [ ] Indicadores visuais de foco (`:focus`)

### ✅ Visual

- [ ] Contraste de cores >= 4.5:1 (texto normal) ou >= 3:1 (texto grande)
- [ ] Tamanho de fonte >= 16px para corpo de texto
- [ ] Alvos de toque >= 44x44px (mobile)
- [ ] Suporte a `prefers-reduced-motion`
- [ ] Suporte a `prefers-color-scheme`
- [ ] Zoom ate 200% sem perda de funcionalidade

### ✅ Screen Readers

- [ ] Testar com NVDA (Windows), JAWS (Windows), VoiceOver (Mac/iOS)
- [ ] Conteudo e anunciado na ordem correta
- [ ] Mudancas dinamicas sao anunciadas (live regions)
- [ ] Imagens tem `alt` text apropriado
- [ ] Links tem texto descritivo (evitar "clique aqui")

## Exercicios

1. Crie um `<accessible-dropdown>` com navegacao por teclado completa
2. Implemente um `<accessible-carousel>` com ARIA e controles de teclado
3. Teste seus componentes com um screen reader e corrija problemas encontrados
4. Crie um `<accessible-form>` que valide e anuncie erros adequadamente
