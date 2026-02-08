# Capitulo 6: Formularios e ElementInternals

## Form-Associated Custom Elements

A API `ElementInternals` permite que Custom Elements participem nativamente de formularios HTML, incluindo validacao, submission e accessibility.

### Configuracao Basica

```javascript
class CustomInput extends HTMLElement {
  // OBRIGATORIO: declarar que o elemento e associado a formulario
  static formAssociated = true;

  #internals;
  #input;

  constructor() {
    super();
    // Criar internals -- so pode ser chamado no constructor
    this.#internals = this.attachInternals();
    this.attachShadow({ mode: 'open' });
  }

  connectedCallback() {
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          margin-bottom: 16px;
        }
        .wrapper {
          display: flex;
          flex-direction: column;
          gap: 4px;
        }
        label {
          font-weight: 500;
          font-size: 14px;
          color: #374151;
        }
        input {
          padding: 8px 12px;
          border: 1px solid #d1d5db;
          border-radius: 6px;
          font-size: 14px;
          transition: border-color 0.2s;
        }
        input:focus {
          outline: none;
          border-color: #3b82f6;
          box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
        }
        input:invalid {
          border-color: #ef4444;
        }
        .error {
          color: #ef4444;
          font-size: 12px;
          min-height: 16px;
        }
      </style>
      <div class="wrapper">
        <label>${this.getAttribute('label') || 'Campo'}</label>
        <input
          type="${this.getAttribute('type') || 'text'}"
          placeholder="${this.getAttribute('placeholder') || ''}"
          ${this.hasAttribute('required') ? 'required' : ''}
        />
        <div class="error"></div>
      </div>
    `;

    this.#input = this.shadowRoot.querySelector('input');
    this.#setupListeners();
  }

  #setupListeners() {
    // Atualizar valor do form quando input muda
    this.#input.addEventListener('input', (e) => {
      const value = e.target.value;

      // Associar valor ao formulario
      this.#internals.setFormValue(value);

      // Validar
      this.#validate(value);
    });

    this.#input.addEventListener('blur', () => {
      this.#validate(this.#input.value);
    });
  }

  #validate(value) {
    const errorEl = this.shadowRoot.querySelector('.error');

    // Validacao: campo obrigatorio
    if (this.hasAttribute('required') && !value.trim()) {
      this.#internals.setValidity(
        { valueMissing: true },
        'Este campo e obrigatorio',
        this.#input
      );
      errorEl.textContent = this.#internals.validationMessage;
      return false;
    }

    // Validacao: minimo de caracteres
    const minLength = parseInt(this.getAttribute('minlength'), 10);
    if (minLength && value.length < minLength) {
      this.#internals.setValidity(
        { tooShort: true },
        `Minimo ${minLength} caracteres`,
        this.#input
      );
      errorEl.textContent = this.#internals.validationMessage;
      return false;
    }

    // Validacao: email
    if (this.getAttribute('type') === 'email' && value) {
      const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
      if (!emailRegex.test(value)) {
        this.#internals.setValidity(
          { typeMismatch: true },
          'Email invalido',
          this.#input
        );
        errorEl.textContent = this.#internals.validationMessage;
        return false;
      }
    }

    // Valido
    this.#internals.setValidity({});
    errorEl.textContent = '';
    return true;
  }

  // Expor propriedades de validacao
  get form() {
    return this.#internals.form;
  }

  get validity() {
    return this.#internals.validity;
  }

  get validationMessage() {
    return this.#internals.validationMessage;
  }

  get willValidate() {
    return this.#internals.willValidate;
  }

  checkValidity() {
    return this.#internals.checkValidity();
  }

  reportValidity() {
    return this.#internals.reportValidity();
  }

  // Form lifecycle callbacks
  formAssociatedCallback(form) {
    console.log('Associado ao form:', form?.id);
  }

  formDisabledCallback(disabled) {
    this.#input.disabled = disabled;
  }

  formResetCallback() {
    this.#input.value = '';
    this.#internals.setFormValue('');
    this.#internals.setValidity({});
    this.shadowRoot.querySelector('.error').textContent = '';
  }

  formStateRestoreCallback(state, mode) {
    // mode: 'restore' (navegacao back/forward) ou 'autocomplete'
    this.#input.value = state;
    this.#internals.setFormValue(state);
  }
}

customElements.define('custom-input', CustomInput);
```

### Uso em Formularios

```html
<form id="cadastro">
  <custom-input
    name="nome"
    label="Nome completo"
    required
    minlength="3"
  ></custom-input>

  <custom-input
    name="email"
    label="E-mail"
    type="email"
    required
  ></custom-input>

  <custom-input
    name="telefone"
    label="Telefone"
    placeholder="(00) 00000-0000"
  ></custom-input>

  <button type="submit">Cadastrar</button>
  <button type="reset">Limpar</button>
</form>

<script>
  const form = document.getElementById('cadastro');

  form.addEventListener('submit', (e) => {
    e.preventDefault();

    // Validacao nativa funciona!
    if (!form.checkValidity()) {
      form.reportValidity();
      return;
    }

    // FormData funciona nativamente
    const formData = new FormData(form);
    console.log('Nome:', formData.get('nome'));
    console.log('Email:', formData.get('email'));
    console.log('Telefone:', formData.get('telefone'));

    // Enviar dados
    fetch('/api/cadastro', {
      method: 'POST',
      body: formData
    });
  });
</script>
```

## Estados de Validacao

A API `setValidity()` aceita um objeto com flags padrao do HTML5:

```javascript
this.#internals.setValidity({
  valueMissing: true,    // Campo obrigatorio vazio
  typeMismatch: true,    // Tipo incorreto (ex: email invalido)
  patternMismatch: true, // Nao corresponde ao pattern
  tooLong: true,         // Valor muito longo
  tooShort: true,        // Valor muito curto
  rangeUnderflow: true,  // Valor abaixo do min
  rangeOverflow: true,   // Valor acima do max
  stepMismatch: true,    // Valor nao corresponde ao step
  badInput: true,        // Entrada invalida (ex: texto em numero)
  customError: true      // Erro customizado
}, 'Mensagem de erro', elementoQueTemOErro);
```

### Validacao Customizada Complexa

```javascript
class CPFInput extends HTMLElement {
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
        input {
          padding: 8px;
          border: 1px solid #ccc;
          border-radius: 4px;
        }
        input:invalid { border-color: red; }
        .error { color: red; font-size: 12px; }
      </style>
      <input type="text" placeholder="000.000.000-00" maxlength="14" />
      <div class="error"></div>
    `;

    const input = this.shadowRoot.querySelector('input');

    // Mascara de CPF
    input.addEventListener('input', (e) => {
      let value = e.target.value.replace(/\D/g, '');
      value = value.replace(/(\d{3})(\d)/, '$1.$2');
      value = value.replace(/(\d{3})(\d)/, '$1.$2');
      value = value.replace(/(\d{3})(\d{1,2})$/, '$1-$2');
      e.target.value = value;

      // Validar CPF
      this.#validateCPF(value.replace(/\D/g, ''));
    });
  }

  #validateCPF(cpf) {
    const errorEl = this.shadowRoot.querySelector('.error');

    // Validar comprimento
    if (cpf.length !== 11) {
      this.#internals.setValidity(
        { customError: true },
        'CPF deve ter 11 digitos',
        this.shadowRoot.querySelector('input')
      );
      errorEl.textContent = 'CPF incompleto';
      this.#internals.setFormValue('');
      return false;
    }

    // Validar CPFs conhecidos como invalidos
    if (/^(\d)\1{10}$/.test(cpf)) {
      this.#internals.setValidity(
        { customError: true },
        'CPF invalido',
        this.shadowRoot.querySelector('input')
      );
      errorEl.textContent = 'CPF invalido';
      this.#internals.setFormValue('');
      return false;
    }

    // Validar digitos verificadores
    let soma = 0;
    for (let i = 0; i < 9; i++) {
      soma += parseInt(cpf.charAt(i)) * (10 - i);
    }
    let resto = 11 - (soma % 11);
    let digito1 = resto === 10 || resto === 11 ? 0 : resto;

    if (digito1 !== parseInt(cpf.charAt(9))) {
      this.#internals.setValidity(
        { customError: true },
        'CPF invalido',
        this.shadowRoot.querySelector('input')
      );
      errorEl.textContent = 'CPF invalido';
      this.#internals.setFormValue('');
      return false;
    }

    soma = 0;
    for (let i = 0; i < 10; i++) {
      soma += parseInt(cpf.charAt(i)) * (11 - i);
    }
    resto = 11 - (soma % 11);
    let digito2 = resto === 10 || resto === 11 ? 0 : resto;

    if (digito2 !== parseInt(cpf.charAt(10))) {
      this.#internals.setValidity(
        { customError: true },
        'CPF invalido',
        this.shadowRoot.querySelector('input')
      );
      errorEl.textContent = 'CPF invalido';
      this.#internals.setFormValue('');
      return false;
    }

    // CPF valido
    this.#internals.setValidity({});
    errorEl.textContent = '';
    this.#internals.setFormValue(cpf);
    return true;
  }

  formResetCallback() {
    this.shadowRoot.querySelector('input').value = '';
    this.#internals.setFormValue('');
    this.#internals.setValidity({});
    this.shadowRoot.querySelector('.error').textContent = '';
  }

  get form() { return this.#internals.form; }
  get validity() { return this.#internals.validity; }
  get validationMessage() { return this.#internals.validationMessage; }
  checkValidity() { return this.#internals.checkValidity(); }
  reportValidity() { return this.#internals.reportValidity(); }
}

customElements.define('cpf-input', CPFInput);
```

## Checkbox e Radio Customizados

### Custom Checkbox

```javascript
class CustomCheckbox extends HTMLElement {
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
        :host {
          display: inline-flex;
          align-items: center;
          gap: 8px;
          cursor: pointer;
          user-select: none;
        }
        .checkbox {
          width: 20px;
          height: 20px;
          border: 2px solid #d1d5db;
          border-radius: 4px;
          display: flex;
          align-items: center;
          justify-content: center;
          transition: all 0.2s;
        }
        :host([checked]) .checkbox {
          background: #3b82f6;
          border-color: #3b82f6;
        }
        .checkmark {
          display: none;
          color: white;
          font-size: 14px;
        }
        :host([checked]) .checkmark {
          display: block;
        }
        :host([disabled]) {
          opacity: 0.5;
          cursor: not-allowed;
        }
      </style>
      <div class="checkbox">
        <span class="checkmark">✓</span>
      </div>
      <slot></slot>
    `;

    this.addEventListener('click', () => {
      if (!this.hasAttribute('disabled')) {
        this.toggle();
      }
    });

    // Suporte a teclado
    this.setAttribute('tabindex', '0');
    this.setAttribute('role', 'checkbox');
    this.addEventListener('keydown', (e) => {
      if (e.key === ' ' || e.key === 'Enter') {
        e.preventDefault();
        this.toggle();
      }
    });

    // Definir valor inicial
    this.#updateFormValue();
  }

  toggle() {
    if (this.hasAttribute('checked')) {
      this.removeAttribute('checked');
    } else {
      this.setAttribute('checked', '');
    }
    this.#updateFormValue();
    this.dispatchEvent(new Event('change', { bubbles: true }));
  }

  #updateFormValue() {
    const checked = this.hasAttribute('checked');
    const value = this.getAttribute('value') || 'on';

    // Checkbox marcado envia valor, desmarcado envia null
    this.#internals.setFormValue(checked ? value : null);

    // Atualizar aria
    this.setAttribute('aria-checked', checked);
  }

  static get observedAttributes() {
    return ['checked', 'disabled'];
  }

  attributeChangedCallback(name, oldValue, newValue) {
    if (name === 'checked') {
      this.#updateFormValue();
    }
  }

  formDisabledCallback(disabled) {
    if (disabled) {
      this.setAttribute('disabled', '');
    } else {
      this.removeAttribute('disabled');
    }
  }

  formResetCallback() {
    this.removeAttribute('checked');
    this.#updateFormValue();
  }

  get checked() {
    return this.hasAttribute('checked');
  }

  set checked(value) {
    if (value) {
      this.setAttribute('checked', '');
    } else {
      this.removeAttribute('checked');
    }
  }
}

customElements.define('custom-checkbox', CustomCheckbox);
```

```html
<form>
  <custom-checkbox name="aceito" value="sim">
    Aceito os termos de uso
  </custom-checkbox>

  <custom-checkbox name="newsletter" value="sim" checked>
    Quero receber novidades
  </custom-checkbox>

  <button type="submit">Enviar</button>
</form>
```

## Select Customizado

```javascript
class CustomSelect extends HTMLElement {
  static formAssociated = true;

  #internals;
  #isOpen = false;

  constructor() {
    super();
    this.#internals = this.attachInternals();
    this.attachShadow({ mode: 'open' });
  }

  connectedCallback() {
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: inline-block;
          position: relative;
          min-width: 200px;
        }
        .select {
          padding: 8px 32px 8px 12px;
          border: 1px solid #d1d5db;
          border-radius: 6px;
          background: white;
          cursor: pointer;
          user-select: none;
          display: flex;
          align-items: center;
          justify-content: space-between;
        }
        .select:hover {
          border-color: #3b82f6;
        }
        .arrow {
          position: absolute;
          right: 12px;
          transition: transform 0.2s;
          pointer-events: none;
        }
        :host([open]) .arrow {
          transform: rotate(180deg);
        }
        .dropdown {
          position: absolute;
          top: calc(100% + 4px);
          left: 0;
          right: 0;
          background: white;
          border: 1px solid #d1d5db;
          border-radius: 6px;
          box-shadow: 0 4px 12px rgba(0,0,0,0.1);
          max-height: 200px;
          overflow-y: auto;
          display: none;
          z-index: 1000;
        }
        :host([open]) .dropdown {
          display: block;
        }
        ::slotted(option) {
          padding: 8px 12px;
          cursor: pointer;
          display: block;
        }
        ::slotted(option:hover) {
          background: #f3f4f6;
        }
        ::slotted(option[selected]) {
          background: #eff6ff;
          color: #3b82f6;
        }
      </style>
      <div class="select">
        <span class="label">Selecione...</span>
        <span class="arrow">▼</span>
      </div>
      <div class="dropdown">
        <slot></slot>
      </div>
    `;

    this.shadowRoot.querySelector('.select').addEventListener('click', () => {
      this.toggleAttribute('open');
    });

    // Fechar ao clicar fora
    document.addEventListener('click', (e) => {
      if (!e.composedPath().includes(this)) {
        this.removeAttribute('open');
      }
    });

    // Listener nas options
    this.addEventListener('click', (e) => {
      if (e.target.tagName === 'OPTION') {
        this.selectOption(e.target);
      }
    });

    // Selecionar opcao inicial
    const selected = this.querySelector('option[selected]');
    if (selected) {
      this.selectOption(selected);
    }
  }

  selectOption(option) {
    // Desmarcar todas
    this.querySelectorAll('option').forEach(opt => {
      opt.removeAttribute('selected');
    });

    // Marcar a selecionada
    option.setAttribute('selected', '');

    // Atualizar display
    const label = this.shadowRoot.querySelector('.label');
    label.textContent = option.textContent;

    // Atualizar form value
    this.#internals.setFormValue(option.value);

    // Fechar dropdown
    this.removeAttribute('open');

    // Disparar evento
    this.dispatchEvent(new Event('change', { bubbles: true }));
  }

  formResetCallback() {
    this.querySelectorAll('option').forEach(opt => {
      opt.removeAttribute('selected');
    });
    this.shadowRoot.querySelector('.label').textContent = 'Selecione...';
    this.#internals.setFormValue('');
  }

  get value() {
    return this.querySelector('option[selected]')?.value || '';
  }

  set value(val) {
    const option = this.querySelector(`option[value="${val}"]`);
    if (option) {
      this.selectOption(option);
    }
  }
}

customElements.define('custom-select', CustomSelect);
```

```html
<form>
  <label>
    Estado:
    <custom-select name="estado">
      <option value="sp">Sao Paulo</option>
      <option value="rj" selected>Rio de Janeiro</option>
      <option value="mg">Minas Gerais</option>
      <option value="rs">Rio Grande do Sul</option>
    </custom-select>
  </label>

  <button type="submit">Enviar</button>
</form>
```

## Exercicios

1. Crie um `email-input` com validacao de email em tempo real
2. Crie um `password-input` com indicador de forca da senha
3. Crie um `rating-input` (estrelas 1-5) form-associated
4. Crie um `date-range-picker` que envie data inicial e final para o form
