# Índice de Exemplos Práticos

Este documento lista todos os exemplos completos e funcionais presentes no livro, organizados por categoria.

## 📦 Componentes Básicos

### Alert Box
**Capítulo**: 1 - Fundamentos
**Descrição**: Componente de alerta com diferentes tipos (success, warning, error)
**Conceitos**: Custom Elements, Shadow DOM básico, atributos

### User Card
**Capítulo**: 1 - Fundamentos
**Descrição**: Card de perfil de usuário com avatar, nome, role e slot para ações
**Conceitos**: Shadow DOM, slots, atributos observados, encapsulamento CSS

### Button Counter
**Capítulo**: 7 - Comunicação
**Descrição**: Botão contador que dispara eventos customizados
**Conceitos**: Custom Events, bubbles, composed

### Styled Card
**Capítulo**: 5 - Estilização
**Descrição**: Card estilizável com variantes (primary, danger) e estados
**Conceitos**: :host, seletores CSS, atributos para estilo

## 📋 Componentes de Formulário

### Custom Input
**Capítulo**: 4 - Ciclo de Vida
**Descrição**: Input customizado com validação e integração com formulários
**Conceitos**: Form-associated, ElementInternals, validação, lifecycle

### CPF Input
**Capítulo**: 6 - Formulários
**Descrição**: Input com máscara e validação de CPF brasileiro
**Conceitos**: Form validation, setValidity, máscaras, regex

### Custom Checkbox
**Capítulo**: 6 - Formulários
**Descrição**: Checkbox customizado acessível
**Conceitos**: Form-associated, ARIA, keyboard navigation

### Custom Select
**Capítulo**: 6 - Formulários
**Descrição**: Select dropdown customizado com estilo
**Conceitos**: Form-associated, slots, keyboard, aria-expanded

### Email Input
**Capítulo**: Exercício Cap. 6
**Sugestão**: Input com validação de email em tempo real

### Password Input
**Capítulo**: Exercício Cap. 6
**Sugestão**: Input de senha com indicador de força

### Rating Input
**Capítulo**: Exercício Cap. 6
**Sugestão**: Seletor de estrelas (1-5) form-associated

## 🎨 Componentes de Layout

### Accordion Item
**Capítulo**: 3 - Templates e Slots
**Descrição**: Item de accordion expansível com animação
**Conceitos**: Templates, slots nomeados, atributo open, transições CSS

### Tab Panel
**Capítulo**: 10 - Acessibilidade
**Descrição**: Sistema de abas com navegação por teclado
**Conceitos**: ARIA roles, keyboard navigation, focus management

### Article Layout
**Capítulo**: 3 - Templates e Slots
**Descrição**: Layout de artigo com múltiplos slots (título, meta, conteúdo, rodapé)
**Conceitos**: Named slots, fallback content, composição

### Form Field
**Capítulo**: 10 - Acessibilidade
**Descrição**: Container de campo de formulário acessível
**Conceitos**: Label association, slots, acessibilidade

## 📊 Componentes de Dados

### Data View
**Capítulo**: 9 - Design Patterns
**Descrição**: Visualizador de dados com múltiplos layouts (list, grid, table, cards)
**Conceitos**: Strategy Pattern, renderização dinâmica

### Optimized List
**Capítulo**: 12 - Performance
**Descrição**: Lista otimizada com batch updates
**Conceitos**: requestAnimationFrame, DocumentFragment, performance

### Virtual List
**Capítulo**: 12 - Performance
**Descrição**: Lista virtual scrolling para milhares de itens
**Conceitos**: Virtual scrolling, performance, DOM optimization

### Data Table
**Capítulo**: Exercício Cap. 8
**Sugestão**: Tabela com sort, filter, pagination

### Data Display
**Capítulo**: 7 - Comunicação
**Descrição**: Componente que recebe objetos complexos via propriedade
**Conceitos**: Propriedades JavaScript vs atributos HTML

## 🔄 Componentes Reativos

### Reactive Counter
**Capítulo**: 8 - Reatividade
**Descrição**: Contador reativo usando Proxy
**Conceitos**: Proxy, reatividade automática, state management

### Todo List
**Capítulo**: 8 - Reatividade
**Descrição**: Todo list completo com filtros e reatividade
**Conceitos**: Deep reactivity, Proxy, filtros, eventos

### Shopping Cart
**Capítulo**: 8 - Reatividade
**Descrição**: Carrinho de compras com computed values
**Conceitos**: Computed properties, cache, subtotal/desconto/total

### Reactive Form
**Capítulo**: Exercício Cap. 8
**Sugestão**: Formulário com validação reativa

## 🎭 Componentes Interativos

### Confirm Dialog
**Capítulo**: 7 - Comunicação
**Descrição**: Dialog de confirmação com eventos canceláveis
**Conceitos**: Custom Events, cancelable, overlay

### Accessible Dialog
**Capítulo**: 10 - Acessibilidade
**Descrição**: Modal dialog totalmente acessível
**Conceitos**: Focus trap, ARIA, keyboard (Escape), restore focus

### Live Notification
**Capítulo**: 10 - Acessibilidade
**Descrição**: Notificação toast com live regions
**Conceitos**: aria-live, role="status", auto-dismiss

### Toggle Button
**Capítulo**: 4 - Ciclo de Vida
**Descrição**: Botão toggle com custom states
**Conceitos**: ElementInternals.states, :state() selector

### Dropdown
**Capítulo**: Exercício Cap. 10
**Sugestão**: Dropdown acessível com navegação por teclado

## 🎯 Componentes Complexos

### Chart Adapter
**Capítulo**: 9 - Design Patterns
**Descrição**: Adapter para biblioteca de gráficos (Chart.js)
**Conceitos**: Adapter Pattern, lazy loading, integração de bibliotecas

### Dynamic Component
**Capítulo**: 12 - Performance
**Descrição**: Componente com carregamento dinâmico
**Conceitos**: Dynamic import, loading states, error handling

### Lazy Loader
**Capítulo**: 12 - Performance
**Descrição**: Sistema de lazy loading com Intersection Observer
**Conceitos**: IntersectionObserver, dynamic import, registry

### Monitored Component
**Capítulo**: 12 - Performance
**Descrição**: Componente com métricas de performance
**Conceitos**: Performance API, measurement

## 🏢 Micro Frontends

### MFE Product Catalog
**Capítulo**: 13 - Micro Frontends
**Descrição**: Catálogo de produtos independente
**Conceitos**: Micro frontend, comunicação via eventos, isolamento

### MFE Shopping Cart
**Capítulo**: 13 - Micro Frontends
**Descrição**: Carrinho de compras como micro frontend
**Conceitos**: Event-driven architecture, cross-MFE communication

### MFE Header/Navigation
**Capítulo**: 13 - Micro Frontends
**Descrição**: Header e navegação como MFEs
**Conceitos**: Shared components, CDN delivery

## 🔧 Utilitários e Patterns

### Event Bus
**Capítulo**: 7 - Comunicação
**Descrição**: Sistema global de mensagens (PubSub)
**Conceitos**: Observer Pattern, Singleton, unsubscribe

### Global Store
**Capítulo**: 7 - Comunicação
**Descrição**: Store de estado compartilhado com Proxy
**Conceitos**: State management, Proxy, subscriptions

### Component Factory
**Capítulo**: 9 - Design Patterns
**Descrição**: Factory para criar componentes dinamicamente
**Conceitos**: Factory Pattern, registry, batch creation

### Service Registry
**Capítulo**: 9 - Design Patterns
**Descrição**: Registry singleton para serviços
**Conceitos**: Singleton Pattern, dependency injection

## 🎨 Mixins

### Logging Mixin
**Capítulo**: 9 - Design Patterns
**Descrição**: Adiciona logging automático aos lifecycle callbacks
**Conceitos**: Mixin Pattern, inheritance, composition

### Theme Mixin
**Capítulo**: 9 - Design Patterns
**Descrição**: Adiciona suporte a temas (light/dark)
**Conceitos**: Mixin, prefers-color-scheme, auto theme

### Fetch Mixin
**Capítulo**: 9 - Design Patterns
**Descrição**: Adiciona capacidade de fetch com cache e abort
**Conceitos**: Mixin, AbortController, cache

## 📚 Componentes Completos (Exemplos Integrados)

### Comprehensive Component
**Capítulo**: 4 - Ciclo de Vida
**Descrição**: Exemplo integrando todos os conceitos de lifecycle
**Conceitos**: Todos os callbacks, atributos, propriedades, eventos

### Detailed Attribute Example
**Capítulo**: 4 - Ciclo de Vida
**Descrição**: Demonstração completa de attributeChangedCallback
**Conceitos**: observedAttributes, sincronização, validação

### Synced Element
**Capítulo**: 4 - Ciclo de Vida
**Descrição**: Sincronização completa atributos ↔ propriedades
**Conceitos**: Getters/setters, reflection, tipos de dados

### Attribute Patterns
**Capítulo**: 4 - Ciclo de Vida
**Descrição**: Todos os patterns de atributos (string, number, boolean, enum, JSON, CSV)
**Conceitos**: Type conversion, validation, parsing

## 🎓 Exemplos por Conceito

### Shadow DOM
- User Card (Cap. 1)
- Styled Card (Cap. 5)
- Encapsulated Widget (Cap. 2)

### Slots
- Accordion (Cap. 3)
- Article Layout (Cap. 3)
- Profile Card (Cap. 3)
- Slot Monitor (Cap. 3)

### Custom Events
- Button Counter (Cap. 7)
- Confirm Dialog (Cap. 7)
- Todo Item → Todo List (Cap. 7)

### Form Integration
- Custom Input (Cap. 6)
- CPF Input (Cap. 6)
- Custom Checkbox (Cap. 6)
- Custom Select (Cap. 6)

### Estilização
- Themed Component (Cap. 5)
- Design System Button (Cap. 5)
- CSS Custom Properties examples (Cap. 5)
- Constructable Stylesheets (Cap. 5)

### Acessibilidade
- Accessible Button (Cap. 10)
- Tab Panel (Cap. 10)
- Accessible Dialog (Cap. 10)
- Live Notification (Cap. 10)

### Performance
- Virtual List (Cap. 12)
- Lazy Loader (Cap. 12)
- Optimized List (Cap. 12)
- Batched Element (Cap. 12)

### Reatividade
- Reactive Counter (Cap. 8)
- Todo List (Cap. 8)
- Shopping Cart (Cap. 8)
- Deep Reactive (Cap. 8)

## 📝 Como Usar os Exemplos

Todos os exemplos são completos e funcionais. Para usar:

1. **Copie o código JavaScript** do exemplo
2. **Salve em um arquivo** (ex: `my-component.js`)
3. **Importe no HTML**:
   ```html
   <script type="module" src="my-component.js"></script>
   <my-component></my-component>
   ```

### Exemplo Completo

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Teste Web Component</title>
</head>
<body>
  <h1>Meu Componente</h1>
  <user-card
    name="João Silva"
    role="Desenvolvedor"
    avatar="https://i.pravatar.cc/80">
  </user-card>

  <script type="module">
    // Cole aqui o código do componente...
    class UserCard extends HTMLElement {
      // ...
    }
    customElements.define('user-card', UserCard);
  </script>
</body>
</html>
```

## 🔗 Navegação Rápida

- **Iniciantes**: Comece com User Card, Alert Box, Accordion
- **Formulários**: Custom Input → CPF Input → Custom Checkbox
- **Interativo**: Todo List → Shopping Cart → Tab Panel
- **Avançado**: Virtual List → Chart Adapter → Micro Frontends
- **Patterns**: Event Bus → Component Factory → Mixins

## ✨ Projetos Sugeridos (Combinando Exemplos)

### Projeto 1: Sistema de Tarefas Completo
Combine:
- Todo List (reativo)
- Custom Input (formulário)
- Accessible Dialog (modal de confirmação)
- Live Notification (feedback)

### Projeto 2: E-commerce
Combine:
- Product Catalog (MFE)
- Shopping Cart (MFE)
- Custom Input (busca)
- Tab Panel (categorias)

### Projeto 3: Dashboard
Combine:
- Chart Adapter (gráficos)
- Data View (múltiplas visualizações)
- Virtual List (logs)
- Theme Mixin (temas)

### Projeto 4: Design System
Combine:
- Styled Card
- Custom Button (várias variantes)
- Custom Input/Checkbox/Select
- Themed Component

Todos os componentes são **production-ready** e seguem as **best practices** mais recentes de 2025-2026! 🚀
