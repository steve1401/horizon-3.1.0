# Horizon Theme Development Guide

## Project Overview

This is a **Shopify Liquid theme** based on Horizon, using the latest Liquid Storefronts features including theme blocks. The theme is server-rendered, web-native, and built for evergreen browsers with progressive enhancement for older ones.

**Core Philosophy:**

- Zero external dependencies
- Server-rendered HTML via Liquid
- Native browser APIs only
- Functional over pixel-perfect
- Progressive enhancement, not polyfills

## Architecture

### Theme Block System

The cornerstone of this theme is **theme blocks** - reusable components that can be nested, configured in the theme editor, and used both dynamically and statically.

**Dynamic blocks** (merchant-added via editor):

```liquid
{% content_for 'blocks' %}
```

**Static blocks** (developer-placed with fixed IDs):

```liquid
{% content_for 'block', type: 'product-gallery', id: 'main-gallery', settings: {
  enable_zoom: true
} %}
```

All blocks must include `{{ block.shopify_attributes }}` for theme editor functionality.

### Component Framework

JavaScript uses a custom Component base class (`assets/component.js`) that extends `DeclarativeShadowElement`:

**Key features:**

- Automatic ref management via `ref` attributes
- Declarative event handlers via `on:click="/methodName"` attributes
- Typed refs using JSDoc for better DX
- Mutation observers keep refs in sync with DOM changes

**Example:**

```liquid
<product-card data-product-id="{{ product.id }}">
  <img ref="productImage" src="{{ product.featured_image | image_url }}">
  <button ref="addButton" on:click="/handleAddToCart">Add to cart</button>
</product-card>
```

```javascript
/**
 * @typedef {Object} ProductCardRefs
 * @property {HTMLButtonElement} addButton
 * @property {HTMLImageElement} productImage
 */

/**
 * @extends {Component<ProductCardRefs>}
 */
class ProductCard extends Component {
  requiredRefs = ['addButton'];

  async handleAddToCart(event) {
    // this.refs.addButton and this.refs.productImage are auto-populated
    this.refs.addButton.disabled = true;
    // Implementation...
  }
}
```

### Spacing & Styling System

**Spacing uses CSS custom properties** set inline via snippets:

```liquid
<div class="spacing-style" style="{% render 'spacing-style', settings: section.settings %}">
```

This generates:

```css
--padding-block-start: max(20px, calc(var(--spacing-scale) * 32px));
--padding-block-end: 32px;
```

The `spacing-style` class applies these variables using logical properties for RTL support.

**Common spacing snippets:**

- `spacing-style.liquid` - Full padding/margin with responsive scaling
- `spacing-padding.liquid` - Simpler padding-only (no scaling)
- `gap-style.liquid` - Gap values with optional scaling
- `typography-style.liquid` - Font variables from settings

## File Structure

```
assets/          # JS, CSS, and static assets
  component.js   # Base component class - read this first
  base.css       # Global styles (BEM naming, utility classes)
blocks/          # Theme block definitions (reusable components)
sections/        # Section templates (containers for blocks)
snippets/        # Reusable Liquid partials
  section.liquid # Section wrapper with background/overlay support
  spacing-*.liquid  # Spacing CSS variable generators
templates/       # Page templates
config/          # Theme settings
locales/         # Translations (use 't' filter for all text)
.cursor/rules/   # Comprehensive coding standards (READ THESE)
```

## Development Standards

### Liquid Patterns

**Always use Liquid blocks for logic:**

```liquid
{% liquid
  assign section_settings = section.settings
  assign display_image = section_settings.image_override

  if display_image == blank and product != blank
    assign display_image = product.featured_image
  endif
%}
```

**Localization is mandatory:**

```liquid
{{ 'products.add_to_cart' | t }}
{{ 'products.price_range' | t: min: product.price_min | money, max: product.price_max | money }}
```

### CSS Standards

**BEM naming with single-level elements:**

```css
.product-card {
}
.product-card__title {
} /* NOT __wrapper__title */
.product-card--featured {
}
.product-card__title--large {
}
```

**Use CSS variables for settings, scoped to component:**

```liquid
<div style="--button-color: {{ settings.button_color }};">
  <button class="button"><!-- Uses var(--button-color) --></button>
</div>
```

**Logical properties for RTL support:**

```css
padding-inline: 2rem; /* NOT padding-left/right */
margin-block: 1rem; /* NOT margin-top/bottom */
border-inline-end: 1px; /* NOT border-right */
text-align: start; /* NOT left */
```

### JavaScript Standards

**Use Component framework for all interactive elements:**

- Import from `@theme/component`
- Define typed refs with JSDoc
- Use `on:event="/methodName"` for event binding
- No external dependencies
- Use `async/await` over `.then()`
- Prefer `for (const item of items)` over `.forEach()`

**Communication patterns:**

- Parent → Child: Call public methods directly
- Child → Parent: Dispatch custom events with typed details
- Sibling → Sibling: Events via document

### HTML Standards

**Use modern native elements:**

- `<details>` + `<summary>` for accordions
- `<dialog>` for modals
- `popover` attribute for floating content
- `<search>` for search forms
- Native HTML5 input types and validation

**ID naming:**

```html
<dialog id="ProductModal-{{ product.id }}-{{ section.id }}">
  <form id="ProductForm-{{ product.id }}-{{ block.id }}"></form>
</dialog>
```

Always append section/block IDs for uniqueness.

## Critical Workflows

### Development Server

```bash
shopify theme dev
```

Launches local dev server with live reload.

### Theme Check (Linting)

```bash
shopify theme check
```

Run before committing. CI runs this automatically.

### Build Schemas

```bash
npm run build:schemas
```

Compiles TypeScript schema definitions into `{% schema %}` blocks in `.liquid` files. Schemas live in `/schemas` folder for type safety and reusability.

### Testing Strategy

- Visual testing via Shopify theme editor preview
- Browser testing in last 2 versions of major browsers
- Accessibility testing for WCAG AA compliance
- No unit tests - this is a Shopify theme

## Key Patterns to Follow

### Section Structure

```liquid
<div class="section-background color-{{ section.settings.color_scheme }}"></div>
<div class="section section--{{ section.settings.section_width }} color-{{ section.settings.color_scheme }}">
  <div class="section-content-wrapper spacing-style"
       style="{% render 'spacing-style', settings: section.settings %}"
       {{ section.shopify_attributes }}>
    {% content_for 'blocks' %}
  </div>
</div>

{% stylesheet %}
  /* Scoped CSS here */
{% endstylesheet %}

{% schema %}
{
  "name": "t:names.section_name",
  "blocks": [{"type": "@theme"}],
  "settings": [...]
}
{% endschema %}
```

### Snippet Documentation

```liquid
{% doc %}
  Component description

  @param product - {Object} Product object (required)
  @param show_vendor - {Boolean} Display vendor (default: false)

  @example
  {% render 'product-card', product: product, show_vendor: true %}
{% enddoc %}
```

### Resource List Layouts

Use `snippets/resource-list.liquid` for grid/carousel/bento layouts:

```liquid
{% render 'resource-list',
  collection: collection,
  settings: {
    layout_type: 'grid',
    columns: 4,
    mobile_columns: 2
  }
%}
```

## Common Gotchas

1. **Refs update automatically** - Don't cache DOM elements in constructors; use `this.refs` which is kept in sync via MutationObserver
2. **Spacing scales above 20px** - Values > 20px use `calc(var(--spacing-scale) * value)` for responsive scaling
3. **Schema IDs use snake_case** - Must match pattern `^[a-z][a-z0-9_]*$`
4. **Translation keys required** - All user-facing text must use `| t` filter with keys in `locales/en.default.json`
5. **Block attributes required** - Always include `{{ block.shopify_attributes }}` or editor won't work
6. **Color-mix is progressive enhancement** - Only use for non-critical features (iOS <16.2 doesn't support it)
7. **No npm packages** - This theme has no `package.json` for runtime. Use native browser APIs only.

## Resources

- Comprehensive standards in `.cursor/rules/` directory
- Component framework code: `assets/component.js`
- Shopify Liquid docs: https://shopify.dev/docs/themes
- Theme Check: https://shopify.dev/docs/themes/tools/theme-check
- Browser baseline: https://web.dev/baseline/

## Notes for AI Agents

- When creating sections/blocks, use existing files in those directories as templates
- Check `.cursor/rules/*.mdc` for detailed standards on HTML, CSS, JS, schemas, localization, and accessibility
- Always use the Component framework for JavaScript - don't create vanilla custom elements
- Spacing and typography use a sophisticated CSS variable system - don't hardcode values
- This theme prioritizes semantic HTML and native browser features over JavaScript solutions
