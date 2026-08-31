# Layout Spec — Admin Web

Referência espacial para todas as telas do admin_web. Baseado no padrão **shadcn/ui sidebar-01** e **Shopify Polaris**. Cada componente tem posição, tamanho e comportamento responsivo definidos.

---

## 1. Breakpoints

| Nome | Largura | Shell |
|------|---------|-------|
| Mobile | < 640px | Header azul (brand + chip) |
| Tablet | 640–959px | Header azul (brand + menu + chip) |
| Desktop | ≥ 960px | Sidebar branca (256px) |

**Regra**: breakpoint único em **960px**. Abaixo = header azul. Acima = sidebar branca. Nunca os dois.

---

## 2. Shell — Desktop (≥ 960px)

```
┌──────────┬──────────────────────────────────────────┐
│ SIDEBAR  │ CONTENT AREA                             │
│ 256px    │ ┌──────────────────────────────────────┐ │
│          │ │ STICKY HEADER (h=48px)               │ │
│ [Brand]  │ │ [Breadcrumb]              [Actions]  │ │
│ 64px     │ ├──────────────────────────────────────┤ │
│──────────│ │                                      │ │
│ [Nav]    │ │ PAGE BODY (scrollável)               │ │
│ 40px × N │ │ [Título da página]                   │ │
│          │ │ [Conteúdo]                           │ │
│          │ │                                      │ │
│──────────│ │                                      │ │
│ [User]   │ │                                      │ │
│ 56px     │ └──────────────────────────────────────┘ │
└──────────┴──────────────────────────────────────────┘
```

### Sidebar (256px fixa)
- **Fundo**: `AppColors.surface` (branco)
- **Borda direita**: 1px `AppColors.line`
- **Largura**: 256px fixa

### Brand (topo da sidebar)
- **Altura**: 64px
- **Conteúdo**: ícone `shield_outlined` 22px `primary` + "Flag Platform" 15px w700 `textPrimary`
- **Padding horizontal**: 16px
- **Borda bottom**: 1px `line`
- **Clicável**: `context.go('/')`

### Nav items
- **Altura por item**: 40px
- **Padding horizontal**: 12px
- **Gap ícone-label**: 10px
- **Ícone**: 20px
- **Label**: 14px, w500 (inativo) / w600 (ativo)
- **Raio**: 8px
- **Ativo**: fundo `primary@8%`, texto/ícone `primary`
- **Hover**: fundo `primary@4%`, texto `textPrimary`
- **Inativo**: texto `textSecondary`, ícone `textSecondary`

### User chip (rodapé da sidebar)
- **Altura**: 56px
- **Borda top**: 1px `line`
- **Padding**: 12px
- **Conteúdo**: CircleAvatar 32px (iniciais `primary@8%`) + Column (nome 13px w600 + email 12px `textSecondary`) + ícone `keyboard_arrow_down` 18px
- **PopupMenu**: "Minha conta" + "Sair"

---

## 3. Shell — Mobile (< 960px)

```
┌──────────────────────────────────────────┐
│ HEADER AZUL (h=64px)                     │
│ [Brand] [Menu módulos] [User chip]       │
├──────────────────────────────────────────┤
│ CONTENT AREA                             │
│ ┌──────────────────────────────────────┐ │
│ │ STICKY HEADER (h=48px)               │ │
│ │ [Back btn] [Breadcrumb]              │ │
│ ├──────────────────────────────────────┤ │
│ │ PAGE BODY                            │ │
│ │ [Título + Actions]                   │ │
│ │ [Conteúdo]                           │ │
│ └──────────────────────────────────────┘ │
└──────────────────────────────────────────┘
```

### Header azul
- **Altura**: 64px fixa
- **Fundo**: `AppColors.primary`
- **Conteúdo**: BrandMark + menu horizontal (≥640px) + AdminUserChip

---

## 4. Content Area — Sticky Header + Page Body

### Sticky Header (dentro do conteúdo, NÃO no shell)
- **Altura**: 48px
- **Fundo**: `AppColors.surface` (branco)
- **Borda bottom**: 1px `AppColors.line`
- **Posição**: `sticky top 0` (gruda no topo ao scrollar o body)
- **Conteúdo**:
  - Desktop: Breadcrumb trail (`Módulo › Nome`) à esquerda
  - Mobile: Back button (`← Módulo`) à esquerda
  - Actions à direita (se houver)

### Page Body (abaixo do sticky header)
- **Padding**: 24px all sides
- **Scroll**: body scrolla, sticky header fica fixo
- **Conteúdo**: título da página + ações + corpo

---

## 5. Tela: Home

```
┌──────────────────────────────────────────┐
│ STICKY HEADER: [Breadcrumb: "Início"]    │
├──────────────────────────────────────────┤
│ [24px gap]                               │
│ [Saudação: "Olá, Nome!"]   14px secondary│
│ [16px gap]                               │
│ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐    │
│ │ Card │ │ Card │ │ Card │ │ Card │    │
│ └──────┘ └──────┘ └──────┘ └──────┘    │
│ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐    │
│ │ Card │ │ Card │ │ Card │ │ Card │    │
│ └──────┘ └──────┘ └──────┘ └──────┘    │
└──────────────────────────────────────────┘
```

- **Sticky header**: "Início" (breadcrumb simples, sem link pai)
- **Body**: padding 24px, saudação + GridView de cards
- **Cards**: `KicksterCard` (ícone + título), 4 colunas ≥960px, 2 <960px

---

## 6. Tela: Listagem (ex: Organizações)

```
┌──────────────────────────────────────────┐
│ STICKY HEADER: [Organizações]            │
├──────────────────────────────────────────┤
│ [24px gap]                               │
│ [Título: "Organizações"]     [+ Novo]    │
│ [8px gap]                                │
│ [Filtros: Wrap com 8px spacing]          │
│ [N resultados] [Filtro tipo] [Busca]     │
│ [16px gap]                               │
│ ┌──────────────┐ ┌──────────────┐       │
│ │ Card Org     │ │ Card Org     │       │
│ └──────────────┘ └──────────────┘       │
└──────────────────────────────────────────┘
```

- **Sticky header**: breadcrumb (sem link pai, só texto)
- **Body**: padding 24px
  - Título 20px w700 + actions à direita (Spacer)
  - Filtros (Wrap)
  - Grid de cards

---

## 7. Tela: Detalhe (ex: Organização/123)

```
┌──────────────────────────────────────────┐
│ STICKY HEADER: [Organizações ›]          │
├──────────────────────────────────────────┤
│ [24px gap]                               │
│ [Título: "Nome da Org"]     [Editar]     │
│ [8px gap]                                │
│ [Seção: Identificação]                   │
│ [Campos empilhados]                      │
│ [16px gap]                               │
│ [Seção: Presidente]                      │
│ ...                                     │
└──────────────────────────────────────────┘
```

- **Sticky header**: breadcrumb clicável (`Organizações` → `/organizations`)
- **Body**: padding 24px
  - Título 20px w700 + actions
  - Seções empilhadas (SingleChildScrollView)

---

## 8. Tela: Formulário (ex: Nova Organização)

```
┌──────────────────────────────────────────┐
│ STICKY HEADER: [Organizações ›]          │
├──────────────────────────────────────────┤
│ [24px gap]                               │
│ [Título: "Nova Organização"]  [Salvar]   │
│ [8px gap]                                │
│ [Formulário com campos]                  │
│ [Label + Input (raio 24)]               │
│ [16px gap entre campos]                 │
│ ...                                     │
└──────────────────────────────────────────┘
```

- **Sticky header**: breadcrumb clicável
- **Body**: padding 24px
  - Título + actions (Salvar)
  - Formulário (SingleChildScrollView)

---

## 9. Regras de Espaçamento

| Elemento | Valor |
|----------|-------|
| Sidebar width | 256px |
| Sticky header height | 48px |
| Sticky header padding | 16px horizontal |
| Page body padding | 24px all sides |
| Gap título do body | 0 (título é o primeiro elemento do body) |
| Gap entre título e actions | Spacer (flexível) |
| Gap entre filtros | 8px |
| Gap entre cards | 12px |
| Gap entre seções | 16px |

---

## 10. Comportamento Responsivo

### Sidebar ↔ Header
- **≥ 960px**: sidebar branca (256px) à esquerda
- **< 960px**: header azul (64px) no topo
- **Nunca**: os dois ao mesmo tempo

### Sticky header
- **Desktop**: breadcrumb trail (`Módulo › Nome`)
- **Mobile**: back button (`← Módulo`)
- **Sempre**: sticky top 0, borda bottom, fundo branco

### Grid de cards
- **≥ 960px**: 2+ colunas
- **< 960px**: 1 coluna

### Título do body
- **Listagem**: 20px w700 + actions à direita
- **Detalhe**: 20px w700 + actions à direita
- **Formulário**: 20px w700 + actions à direita
- **Home**: sem título (só saudação)
