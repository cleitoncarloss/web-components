# Capitulo 11: Testes

## Estrategia de Testes

### Piramide de Testes

```
        /\
       /  \    E2E (poucos)
      /____\
     /      \   Integracao (alguns)
    /________\
   /          \  Unitarios (muitos)
  /__________\
```

## Testes Unitarios

### Setup com @web/test-runner

```bash
npm install --save-dev @web/test-runner @esm-bundle/chai
```

**web-test-runner.config.mjs**:
```javascript
export default {
  files: 'test/**/*.test.js',
  nodeResolve: true,
  coverage: true,
  coverageConfig: {
    threshold: {
      statements: 80,
      branches: 80,
      functions: 80,
      lines: 80
    }
  }
};
```

### Teste Basico

```javascript
// counter-element.test.js
import { expect } from '@esm-bundle/chai';
import '../src/counter-element.js';

describe('CounterElement', () => {
  let element;

  beforeEach(async () => {
    element = document.createElement('counter-element');
    document.body.appendChild(element);
    await customElements.whenDefined('counter-element');
    // Aguardar render
    await new Promise(resolve => setTimeout(resolve, 0));
  });

  afterEach(() => {
    element.remove();
  });

  it('deve renderizar com count inicial de 0', () => {
    const countEl = element.shadowRoot.querySelector('.count');
    expect(countEl.textContent).to.equal('0');
  });

  it('deve incrementar ao clicar no botao +', () => {
    const incrementBtn = element.shadowRoot.querySelector('.increment');
    incrementBtn.click();

    const countEl = element.shadowRoot.querySelector('.count');
    expect(countEl.textContent).to.equal('1');
  });

  it('deve decrementar ao clicar no botao -', () => {
    const decrementBtn = element.shadowRoot.querySelector('.decrement');
    decrementBtn.click();

    const countEl = element.shadowRoot.querySelector('.count');
    expect(countEl.textContent).to.equal('-1');
  });

  it('deve resetar ao clicar no botao reset', () => {
    // Incrementar algumas vezes
    const incrementBtn = element.shadowRoot.querySelector('.increment');
    incrementBtn.click();
    incrementBtn.click();
    incrementBtn.click();

    // Resetar
    const resetBtn = element.shadowRoot.querySelector('.reset');
    resetBtn.click();

    const countEl = element.shadowRoot.querySelector('.count');
    expect(countEl.textContent).to.equal('0');
  });

  it('deve disparar evento count-change ao incrementar', (done) => {
    element.addEventListener('count-change', (e) => {
      expect(e.detail.count).to.equal(1);
      done();
    });

    const incrementBtn = element.shadowRoot.querySelector('.increment');
    incrementBtn.click();
  });
});
```

### Testando Atributos

```javascript
describe('CounterElement - Atributos', () => {
  let element;

  beforeEach(async () => {
    element = document.createElement('counter-element');
    document.body.appendChild(element);
    await customElements.whenDefined('counter-element');
  });

  afterEach(() => {
    element.remove();
  });

  it('deve aceitar atributo step', async () => {
    element.setAttribute('step', '5');
    await new Promise(resolve => setTimeout(resolve, 0));

    const incrementBtn = element.shadowRoot.querySelector('.increment');
    incrementBtn.click();

    const countEl = element.shadowRoot.querySelector('.count');
    expect(countEl.textContent).to.equal('5');
  });

  it('deve sincronizar propriedade com atributo', () => {
    element.step = 10;
    expect(element.getAttribute('step')).to.equal('10');
  });

  it('deve reagir a mudancas de atributo', async () => {
    element.setAttribute('disabled', '');
    await new Promise(resolve => setTimeout(resolve, 0));

    const incrementBtn = element.shadowRoot.querySelector('.increment');
    expect(incrementBtn.disabled).to.be.true;
  });
});
```

### Testando Slots

```javascript
describe('CardElement - Slots', () => {
  let element;

  beforeEach(async () => {
    element = document.createElement('card-element');
    element.innerHTML = `
      <h2 slot="title">Meu Titulo</h2>
      <p slot="content">Meu conteudo</p>
    `;
    document.body.appendChild(element);
    await customElements.whenDefined('card-element');
  });

  afterEach(() => {
    element.remove();
  });

  it('deve projetar conteudo no slot title', () => {
    const slot = element.shadowRoot.querySelector('slot[name="title"]');
    const assignedNodes = slot.assignedNodes();

    expect(assignedNodes.length).to.equal(1);
    expect(assignedNodes[0].textContent).to.equal('Meu Titulo');
  });

  it('deve exibir fallback quando slot esta vazio', async () => {
    const emptyElement = document.createElement('card-element');
    document.body.appendChild(emptyElement);
    await customElements.whenDefined('card-element');

    const slot = emptyElement.shadowRoot.querySelector('slot[name="title"]');
    const assignedNodes = slot.assignedNodes();

    expect(assignedNodes.length).to.equal(0);

    // Verificar fallback content
    expect(slot.textContent.trim()).to.equal('Titulo Padrao');

    emptyElement.remove();
  });

  it('deve reagir ao evento slotchange', (done) => {
    const slot = element.shadowRoot.querySelector('slot[name="title"]');

    slot.addEventListener('slotchange', () => {
      const nodes = slot.assignedNodes();
      expect(nodes.length).to.equal(2);
      done();
    });

    // Adicionar novo elemento ao slot
    const newTitle = document.createElement('span');
    newTitle.setAttribute('slot', 'title');
    newTitle.textContent = 'Segundo Titulo';
    element.appendChild(newTitle);
  });
});
```

### Testando Form-Associated Elements

```javascript
describe('CustomInput - Form Association', () => {
  let form;
  let input;

  beforeEach(async () => {
    form = document.createElement('form');
    input = document.createElement('custom-input');
    input.setAttribute('name', 'username');
    input.setAttribute('required', '');

    form.appendChild(input);
    document.body.appendChild(form);

    await customElements.whenDefined('custom-input');
  });

  afterEach(() => {
    form.remove();
  });

  it('deve associar ao form', () => {
    expect(input.form).to.equal(form);
  });

  it('deve validar campo obrigatorio', () => {
    expect(input.checkValidity()).to.be.false;
    expect(input.validity.valueMissing).to.be.true;
  });

  it('deve aparecer no FormData', () => {
    // Simular input de usuario
    const internalInput = input.shadowRoot.querySelector('input');
    internalInput.value = 'john';
    internalInput.dispatchEvent(new Event('input', { bubbles: true }));

    const formData = new FormData(form);
    expect(formData.get('username')).to.equal('john');
  });

  it('deve resetar com o form', () => {
    const internalInput = input.shadowRoot.querySelector('input');
    internalInput.value = 'john';

    form.reset();

    expect(internalInput.value).to.equal('');
  });

  it('deve desabilitar quando form disabled', () => {
    const fieldset = document.createElement('fieldset');
    fieldset.disabled = true;
    fieldset.appendChild(input);
    form.appendChild(fieldset);

    // formDisabledCallback deve ser chamado
    const internalInput = input.shadowRoot.querySelector('input');
    expect(internalInput.disabled).to.be.true;
  });
});
```

## Testes de Integracao

### Testando Comunicacao entre Componentes

```javascript
describe('Comunicacao Parent-Child', () => {
  let parent;
  let child;

  beforeEach(async () => {
    parent = document.createElement('parent-component');
    child = document.createElement('child-component');
    parent.appendChild(child);
    document.body.appendChild(parent);

    await Promise.all([
      customElements.whenDefined('parent-component'),
      customElements.whenDefined('child-component')
    ]);
  });

  afterEach(() => {
    parent.remove();
  });

  it('deve passar dados de pai para filho via atributo', async () => {
    parent.setAttribute('message', 'Ola do pai');
    await new Promise(resolve => setTimeout(resolve, 0));

    expect(child.getAttribute('message')).to.equal('Ola do pai');
  });

  it('deve receber evento do filho no pai', (done) => {
    parent.addEventListener('child-event', (e) => {
      expect(e.detail.data).to.equal('dados do filho');
      done();
    });

    child.dispatchEvent(new CustomEvent('child-event', {
      detail: { data: 'dados do filho' },
      bubbles: true,
      composed: true
    }));
  });
});
```

### Testando Event Bus

```javascript
describe('EventBus', () => {
  let eventBus;
  let component1;
  let component2;

  beforeEach(() => {
    eventBus = EventBus.getInstance();
    eventBus.clear();

    component1 = document.createElement('component-one');
    component2 = document.createElement('component-two');

    document.body.append(component1, component2);
  });

  afterEach(() => {
    component1.remove();
    component2.remove();
    eventBus.clear();
  });

  it('deve permitir comunicacao entre componentes', (done) => {
    eventBus.on('test-event', (data) => {
      expect(data.message).to.equal('Hello');
      done();
    });

    eventBus.emit('test-event', { message: 'Hello' });
  });

  it('deve remover listener com unsubscribe', () => {
    let callCount = 0;

    const unsubscribe = eventBus.on('test-event', () => {
      callCount++;
    });

    eventBus.emit('test-event');
    expect(callCount).to.equal(1);

    unsubscribe();
    eventBus.emit('test-event');
    expect(callCount).to.equal(1); // Nao incrementou
  });

  it('deve limpar todos os listeners', () => {
    let callCount = 0;

    eventBus.on('event-1', () => callCount++);
    eventBus.on('event-2', () => callCount++);

    eventBus.clear();

    eventBus.emit('event-1');
    eventBus.emit('event-2');

    expect(callCount).to.equal(0);
  });
});
```

## Testes E2E com Playwright

### Setup

```bash
npm install --save-dev @playwright/test
npx playwright install
```

**playwright.config.js**:
```javascript
export default {
  testDir: './e2e',
  use: {
    baseURL: 'http://localhost:8080',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure'
  },
  webServer: {
    command: 'npm run serve',
    port: 8080
  }
};
```

### Testes E2E

```javascript
// e2e/todo-app.spec.js
import { test, expect } from '@playwright/test';

test.describe('Todo App', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/');
  });

  test('deve adicionar nova tarefa', async ({ page }) => {
    // Playwright atravessa Shadow DOM automaticamente
    await page.locator('todo-app input[type="text"]').fill('Comprar leite');
    await page.locator('todo-app button[type="submit"]').click();

    // Verificar que tarefa foi adicionada
    const todoItem = page.locator('todo-app .todo-item').first();
    await expect(todoItem).toContainText('Comprar leite');
  });

  test('deve marcar tarefa como concluida', async ({ page }) => {
    // Adicionar tarefa
    await page.locator('todo-app input[type="text"]').fill('Fazer exercicios');
    await page.locator('todo-app button[type="submit"]').click();

    // Marcar como concluida
    await page.locator('todo-app .todo-checkbox').first().check();

    // Verificar classe .completed
    const todoItem = page.locator('todo-app .todo-item').first();
    await expect(todoItem).toHaveClass(/completed/);
  });

  test('deve filtrar tarefas', async ({ page }) => {
    // Adicionar algumas tarefas
    const input = page.locator('todo-app input[type="text"]');
    const submitBtn = page.locator('todo-app button[type="submit"]');

    await input.fill('Tarefa 1');
    await submitBtn.click();
    await input.fill('Tarefa 2');
    await submitBtn.click();

    // Marcar primeira como concluida
    await page.locator('todo-app .todo-checkbox').first().check();

    // Filtrar ativas
    await page.locator('todo-app button[data-filter="active"]').click();

    // Verificar que apenas 1 tarefa aparece
    const visibleTodos = page.locator('todo-app .todo-item');
    await expect(visibleTodos).toHaveCount(1);
    await expect(visibleTodos).toContainText('Tarefa 2');
  });

  test('deve persistir tarefas apos reload', async ({ page }) => {
    // Adicionar tarefa
    await page.locator('todo-app input[type="text"]').fill('Tarefa persistente');
    await page.locator('todo-app button[type="submit"]').click();

    // Recarregar pagina
    await page.reload();

    // Verificar que tarefa ainda existe
    const todoItem = page.locator('todo-app .todo-item').first();
    await expect(todoItem).toContainText('Tarefa persistente');
  });
});
```

### Testes de Acessibilidade

```bash
npm install --save-dev @axe-core/playwright
```

```javascript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Acessibilidade', () => {
  test('nao deve ter violacoes de acessibilidade', async ({ page }) => {
    await page.goto('/');

    const accessibilityScanResults = await new AxeBuilder({ page })
      .analyze();

    expect(accessibilityScanResults.violations).toEqual([]);
  });

  test('deve ter contraste adequado', async ({ page }) => {
    await page.goto('/');

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2aa'])
      .analyze();

    const contrastViolations = results.violations.filter(
      v => v.id === 'color-contrast'
    );

    expect(contrastViolations).toHaveLength(0);
  });

  test('deve ser navegavel por teclado', async ({ page }) => {
    await page.goto('/');

    // Tab ate o primeiro input
    await page.keyboard.press('Tab');

    // Verificar que input esta focado
    const focused = await page.evaluateHandle(() => document.activeElement);
    const tagName = await focused.evaluate(el => el.tagName.toLowerCase());

    expect(tagName).toBe('input');
  });
});
```

## Snapshot Testing

```javascript
describe('CounterElement - Snapshots', () => {
  it('deve corresponder ao snapshot', async () => {
    const element = document.createElement('counter-element');
    document.body.appendChild(element);
    await customElements.whenDefined('counter-element');

    const html = element.shadowRoot.innerHTML;

    // Primeira vez: gera snapshot
    // Proximas vezes: compara com snapshot salvo
    expect(html).toMatchSnapshot();

    element.remove();
  });
});
```

## Coverage

```bash
# Rodar testes com coverage
npm test -- --coverage

# Gerar relatorio HTML
npm test -- --coverage --coverage-report html
```

**Analisar coverage**:
- Statements: % de linhas executadas
- Branches: % de condicionais testadas (if/else)
- Functions: % de funcoes chamadas
- Lines: % de linhas de codigo executadas

**Metas recomendadas**:
- Unitarios: >= 80%
- Integracao: >= 60%
- E2E: funcionalidades criticas

## Exercicios

1. Escreva testes unitarios completos para um componente `<user-form>`
2. Crie testes de integracao para um sistema de carrinho de compras
3. Implemente testes E2E para um fluxo de login completo
4. Adicione testes de acessibilidade automatizados com axe-core
