# Capitulo 1: Introducao e Fundamentos

## O que sao Web Components?

Web Components sao um conjunto de APIs nativas do navegador que permitem criar elementos HTML customizados, reutilizaveis e encapsulados. Nao dependem de nenhum framework ou biblioteca -- sao padroes da plataforma web.

As tres tecnologias centrais sao:

1. **Custom Elements** -- API JavaScript para definir novos elementos HTML
2. **Shadow DOM** -- Arvore DOM encapsulada, isolada do documento principal
3. **HTML Templates & Slots** -- Markup reutilizavel que nao renderiza ate ser clonado

## Por que aprender Web Components?

- **Zero dependencias**: Funcionam nativamente em todos os navegadores modernos
- **Interoperabilidade**: Funcionam com React, Vue, Angular ou qualquer framework
- **Encapsulamento real**: CSS e DOM isolados via Shadow DOM
- **Longevidade**: Baseados em padroes web, nao em frameworks que mudam a cada ano
- **Adocao enterprise**: GitHub, Google, Salesforce, Adobe, Microsoft, SAP usam em producao

## Custom Elements

Custom Elements permitem definir novas tags HTML com comportamento proprio.

### Regras de nomenclatura

- Deve comecar com letra minuscula
- **Deve conter um hifen** (`-`) -- isso diferencia de elementos HTML nativos
- Nao pode conflitar com nomes reservados

```
// Validos
my-card
app-header
user-profile-card

// Invalidos
mycard        (sem hifen)
My-Card       (letra maiuscula)
123-element   (comeca com numero)
```

### Autonomous Custom Elements

Estendem `HTMLElement` e implementam todo o comportamento do zero:

```javascript
class MeuBotao extends HTMLElement {
  constructor() {
    super(); // Sempre chamar super() primeiro
  }

  connectedCallback() {
    // Chamado quando o elemento e inserido no DOM
    this.innerHTML = `<button>${this.getAttribute('label') || 'Clique'}</button>`;

    this.querySelector('button').addEventListener('click', () => {
      this.dispatchEvent(new CustomEvent('meu-click', {
        bubbles: true,
        detail: { timestamp: Date.now() }
      }));
    });
  }
}

// Registrar o elemento
customElements.define('meu-botao', MeuBotao);
```

```html
<!-- Uso no HTML -->
<meu-botao label="Enviar"></meu-botao>

<script>
  document.querySelector('meu-botao')
    .addEventListener('meu-click', (e) => {
      console.log('Clicado em:', e.detail.timestamp);
    });
</script>
```

### Customized Built-in Elements

Estendem um elemento HTML existente para adicionar comportamento:

```javascript
class BotaoConfirmacao extends HTMLButtonElement {
  connectedCallback() {
    this.addEventListener('click', (e) => {
      if (!confirm('Tem certeza?')) {
        e.preventDefault();
        e.stopPropagation();
      }
    });
  }
}

customElements.define('botao-confirmacao', BotaoConfirmacao, { extends: 'button' });
```

```html
<!-- Uso com atributo "is" -->
<button is="botao-confirmacao">Deletar</button>
```

> **Importante**: Safari NAO suporta Customized Built-in Elements e nao planeja implementar.
> Para compatibilidade total, prefira Autonomous Custom Elements.

### O Registry: `customElements`

A API `customElements` gerencia o registro de elementos:

```javascript
// Registrar
customElements.define('meu-elemento', MinhaClasse);

// Obter a classe de um elemento registrado
const Classe = customElements.get('meu-elemento');

// Verificar se ja foi definido (retorna Promise)
await customElements.whenDefined('meu-elemento');

// Aguardar multiplos elementos
await Promise.all([
  customElements.whenDefined('app-header'),
  customElements.whenDefined('app-footer'),
  customElements.whenDefined('app-sidebar')
]);
console.log('Todos os componentes prontos!');

// Obter o nome de uma classe registrada
const nome = customElements.getName(MinhaClasse); // 'meu-elemento'
```

### Element Upgrades

Quando o HTML e parseado antes do JavaScript carregar, elementos customizados existem como `HTMLUnknownElement`. Quando `customElements.define()` e chamado, eles sao "atualizados" (upgraded):

```css
/* Estilo antes do componente ser definido (carregando) */
meu-card:not(:defined) {
  opacity: 0;
  min-height: 100px;
  background: #f0f0f0;
  animation: pulse 1.5s infinite;
}

/* Estilo depois de definido */
meu-card:defined {
  opacity: 1;
  transition: opacity 0.3s;
}

@keyframes pulse {
  0%, 100% { opacity: 0.4; }
  50% { opacity: 0.7; }
}
```

## Primeiro Componente Completo

Vamos criar um componente `user-card` do zero:

```javascript
class UserCard extends HTMLElement {
  constructor() {
    super();
    // Criar Shadow DOM para encapsulamento
    this.attachShadow({ mode: 'open' });
  }

  // Atributos que queremos observar
  static observedAttributes = ['name', 'role', 'avatar'];

  connectedCallback() {
    this.render();
  }

  attributeChangedCallback(name, oldValue, newValue) {
    // Re-renderizar quando atributos mudam
    if (oldValue !== newValue && this.isConnected) {
      this.render();
    }
  }

  render() {
    const name = this.getAttribute('name') || 'Usuario';
    const role = this.getAttribute('role') || 'Membro';
    const avatar = this.getAttribute('avatar') || '';

    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: inline-block;
          font-family: system-ui, sans-serif;
        }
        .card {
          border: 1px solid #e0e0e0;
          border-radius: 12px;
          padding: 20px;
          text-align: center;
          max-width: 250px;
          box-shadow: 0 2px 8px rgba(0,0,0,0.1);
        }
        .avatar {
          width: 80px;
          height: 80px;
          border-radius: 50%;
          background: #3b82f6;
          color: white;
          display: flex;
          align-items: center;
          justify-content: center;
          font-size: 2em;
          margin: 0 auto 12px;
        }
        .avatar img {
          width: 100%;
          height: 100%;
          border-radius: 50%;
          object-fit: cover;
        }
        .name {
          font-size: 1.2em;
          font-weight: bold;
          margin: 0 0 4px;
        }
        .role {
          color: #666;
          font-size: 0.9em;
          margin: 0;
        }
      </style>
      <div class="card">
        <div class="avatar">
          ${avatar
            ? `<img src="${avatar}" alt="${name}" />`
            : name.charAt(0).toUpperCase()
          }
        </div>
        <p class="name">${name}</p>
        <p class="role">${role}</p>
        <slot></slot>
      </div>
    `;
  }
}

customElements.define('user-card', UserCard);
```

```html
<!-- Uso -->
<user-card name="Cleiton" role="Front-end Developer">
  <a href="#">Ver perfil</a>
</user-card>

<user-card
  name="Maria"
  role="Designer"
  avatar="https://i.pravatar.cc/80?img=5">
</user-card>
```

## Exercicios

1. Crie um componente `alert-box` que aceite atributos `type` (success, warning, error) e exiba um icone e cor diferentes para cada tipo
2. Crie um componente `countdown-timer` que aceite um atributo `seconds` e faca uma contagem regressiva
3. Use `:not(:defined)` para criar um efeito de loading enquanto o componente carrega
