# Phase 12.8 — Angular Internationalization (i18n)

> **Prerequisites:** Angular components, pipes, build system basics

---

## Table of Contents
1. [i18n Approaches in Angular](#1-i18n-approaches-in-angular)
2. [Angular Built-in i18n (@angular/localize)](#2-angular-built-in-i18n-angularlocalize)
3. [Marking Text for Translation](#3-marking-text-for-translation)
4. [$localize Tag — TypeScript Translations](#4-localize-tag--typescript-translations)
5. [Extracting Translation Files (ng extract-i18n)](#5-extracting-translation-files-ng-extract-i18n)
6. [Translation File Formats](#6-translation-file-formats)
7. [Building for Multiple Locales](#7-building-for-multiple-locales)
8. [Runtime Translation — Transloco](#8-runtime-translation--transloco)
9. [Locale-Aware Pipes](#9-locale-aware-pipes)
10. [Date, Number, and Currency Formatting](#10-date-number-and-currency-formatting)
11. [RTL (Right-to-Left) Support](#11-rtl-right-to-left-support)
12. [Best Practices](#12-best-practices)

---

## 1. i18n Approaches in Angular

Angular offers two fundamentally different approaches:

### Approach 1: Build-Time i18n (`@angular/localize`)
- Translations baked into separate builds per locale
- Maximum performance (no runtime overhead)
- Requires a separate build per language (`en`, `fr`, `de`, etc.)
- Best for: Large production apps with known locales, CDN-hosted apps

### Approach 2: Runtime i18n (`Transloco`, `ngx-translate`)
- Single build, language loaded at runtime from JSON files
- More flexible (add languages without rebuilding)
- Slight runtime overhead
- Best for: Apps requiring dynamic language switching, user-selectable language

---

## 2. Angular Built-in i18n (`@angular/localize`)

### Installation

```bash
ng add @angular/localize
```

This adds `@angular/localize` to your package.json and polyfills.

### How It Works

1. Mark text in templates with `i18n` attribute
2. Run `ng extract-i18n` → generates XLIFF/XLF translation files
3. Translators fill in translations
4. Build with `--localize` → one build per locale

---

## 3. Marking Text for Translation

### Basic Text

```html
<!-- Simple text — just add i18n attribute -->
<h1 i18n>Welcome to our shop</h1>

<!-- With a description for translators -->
<p i18n="Shown on the home page hero section">
  Browse thousands of products
</p>

<!-- With a unique ID (stable across refactors) -->
<button i18n="@@login-button">Log In</button>
```

### Interpolated Text

```html
<!-- Interpolations are fine — Angular handles them -->
<p i18n>Hello, {{ user.name }}! You have {{ count }} messages.</p>
```

### Pluralization with `plural`

Different text based on a count:

```html
<p i18n>
  {count, plural,
    =0 {No items in your cart}
    =1 {One item in your cart}
    other {{{ count }} items in your cart}
  }
</p>
```

### Select (Conditional Text by Variable)

```html
<p i18n>
  {gender, select,
    male   {He logged in}
    female {She logged in}
    other  {They logged in}
  }
</p>
```

### Attribute Translation

```html
<!-- Translate an attribute value -->
<input [attr.placeholder]="placeholder" />
<input i18n-placeholder placeholder="Search products..." />

<!-- Multiple attributes -->
<img [src]="imageUrl" i18n-alt alt="A beautiful product photo" />
```

### Nested Expressions

```html
<p i18n>
  {shippingMethod, select,
    express {Your order arrives {deliveryDate, plural,
      =0 {today}
      =1 {tomorrow}
      other {in {{ deliveryDate }} days}
    }}
    standard {Standard delivery in 3-5 days}
  }
</p>
```

---

## 4. `$localize` Tag — TypeScript Translations

For strings in TypeScript (service messages, error text, dynamic content):

```typescript
import '@angular/localize/init';

@Injectable({ providedIn: 'root' })
export class NotificationService {
  getWelcomeMessage(name: string): string {
    // $localize tag — works in TypeScript files
    return $localize`Welcome back, ${name}!`;
  }

  getErrorMessage(code: number): string {
    return $localize`:@@error-code-message:Error ${code} occurred. Please try again.`;
  }

  getItemCount(count: number): string {
    // With ICU plural in TypeScript
    return $localize`{count, plural, =1 {${count} item} other {${count} items}}`;
  }
}
```

### Metadata Syntax for $localize

```typescript
// :description:source text
$localize`:Used in the cart total area:Total: ${amount}`

// :@@customId:source text
$localize`:@@checkout-total:Total: ${amount}`

// :description@@customId:source text
$localize`:Checkout page total@@checkout-total:Total: ${amount}`
```

---

## 5. Extracting Translation Files (`ng extract-i18n`)

```bash
# Extract to default (XLIFF 1.2 format, ./messages.xlf)
ng extract-i18n

# Specify format
ng extract-i18n --format xlf       # XLIFF 1.2
ng extract-i18n --format xlf2      # XLIFF 2.0
ng extract-i18n --format json      # JSON (for ngx-translate compatibility)
ng extract-i18n --format xmb       # XML Message Bundle

# Specify output file
ng extract-i18n --output-path src/locale --out-file messages.xlf
```

### Generated XLIFF File (messages.xlf)

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<xliff version="1.2" xmlns="urn:oasis:names:tc:xliff:document:1.2">
  <file source-language="en" datatype="plaintext" original="ng2.template">
    <body>
      <trans-unit id="login-button" datatype="html">
        <source>Log In</source>
        <context-group purpose="location">
          <context context-type="sourcefile">src/app/auth/login.component.html</context>
          <context context-type="linenumber">42</context>
        </context-group>
        <note priority="1" from="description">Button text in auth form</note>
      </trans-unit>

      <trans-unit id="welcomeMessage" datatype="html">
        <source>Welcome to our shop</source>
      </trans-unit>
    </body>
  </file>
</xliff>
```

---

## 6. Translation File Formats

### XLIFF 2.0 (Recommended for New Projects)

```xml
<!-- messages.fr.xlf — French translation -->
<?xml version="1.0" encoding="UTF-8" ?>
<xliff version="2.0" xmlns="urn:oasis:names:tc:xliff:document:2.0" srcLang="en" trgLang="fr">
  <file id="ngi18n">
    <unit id="login-button">
      <segment>
        <source>Log In</source>
        <target>Se connecter</target>
      </segment>
    </unit>
    <unit id="welcomeMessage">
      <segment>
        <source>Welcome to our shop</source>
        <target>Bienvenue dans notre boutique</target>
      </segment>
    </unit>
  </file>
</xliff>
```

### Directory Structure

```
src/
└── locale/
    ├── messages.xlf         ← Source (English)
    ├── messages.fr.xlf      ← French translations
    ├── messages.de.xlf      ← German translations
    ├── messages.ar.xlf      ← Arabic translations (RTL)
    └── messages.ja.xlf      ← Japanese translations
```

---

## 7. Building for Multiple Locales

### Configuration in angular.json

```json
{
  "projects": {
    "my-app": {
      "i18n": {
        "sourceLocale": "en-US",
        "locales": {
          "fr": {
            "translation": "src/locale/messages.fr.xlf",
            "baseHref": "/fr/"
          },
          "de": {
            "translation": "src/locale/messages.de.xlf",
            "baseHref": "/de/"
          },
          "ar": {
            "translation": "src/locale/messages.ar.xlf",
            "baseHref": "/ar/"
          }
        }
      },
      "architect": {
        "build": {
          "configurations": {
            "production": {
              "localize": true    // Build all locales
            },
            "fr": {
              "localize": ["fr"]  // Build only French
            }
          }
        }
      }
    }
  }
}
```

### Building

```bash
# Build all locales
ng build --configuration=production
# Creates: dist/my-app/en-US/, dist/my-app/fr/, dist/my-app/de/

# Build single locale
ng build --configuration=production,fr
# Creates: dist/my-app/fr/

# Serve a specific locale for development
ng serve --configuration=fr
```

### Deployment Structure

```
dist/my-app/
├── en-US/
│   └── index.html   ← English version
├── fr/
│   └── index.html   ← French version
└── de/
    └── index.html   ← German version
```

Nginx routing:
```nginx
server {
  listen 80;

  # Route by Accept-Language header
  location / {
    if ($http_accept_language ~* "^fr") {
      return 302 /fr/;
    }
    if ($http_accept_language ~* "^de") {
      return 302 /de/;
    }
    return 302 /en-US/;
  }

  location /fr/ { root /dist/my-app; }
  location /de/ { root /dist/my-app; }
  location /en-US/ { root /dist/my-app; }
}
```

---

## 8. Runtime Translation — Transloco

**Transloco** is the recommended runtime i18n library for Angular (official community library):

### Installation

```bash
ng add @ngneat/transloco
```

### Setup

```typescript
// app.config.ts
import { provideTransloco, TranslocoHttpLoader } from '@ngneat/transloco';

export const appConfig: ApplicationConfig = {
  providers: [
    provideTransloco({
      config: {
        availableLangs: ['en', 'fr', 'de', 'ar'],
        defaultLang: 'en',
        reRenderOnLangChange: true,    // Auto re-render on language switch
        prodMode: environment.production,
        fallbackLang: 'en',
      },
      loader: TranslocoHttpLoader,     // Loads JSON files from /assets/i18n/
    }),
  ],
};
```

### Translation Files

```json
// assets/i18n/en.json
{
  "home": {
    "title": "Welcome to our shop",
    "subtitle": "Browse thousands of products"
  },
  "auth": {
    "login": "Log In",
    "logout": "Log Out",
    "greeting": "Hello, {{ name }}!"
  },
  "cart": {
    "empty": "Your cart is empty",
    "itemCount": "{{ count }} item(s) in cart"
  }
}

// assets/i18n/fr.json
{
  "home": {
    "title": "Bienvenue dans notre boutique",
    "subtitle": "Parcourez des milliers de produits"
  },
  "auth": {
    "login": "Se connecter",
    "logout": "Se déconnecter",
    "greeting": "Bonjour, {{ name }}!"
  },
  "cart": {
    "empty": "Votre panier est vide",
    "itemCount": "{{ count }} article(s) dans le panier"
  }
}
```

### Using Transloco in Templates

```typescript
import { TranslocoModule } from '@ngneat/transloco';

@Component({
  standalone: true,
  imports: [TranslocoModule],
  template: `
    <!-- Using the transloco pipe -->
    <h1>{{ 'home.title' | transloco }}</h1>

    <!-- With parameters -->
    <p>{{ 'auth.greeting' | transloco: { name: user.name } }}</p>

    <!-- Using the transloco directive (preferred for performance) -->
    <div *transloco="let t">
      <h1>{{ t('home.title') }}</h1>
      <p>{{ t('auth.greeting', { name: user.name }) }}</p>
      <button>{{ t('auth.login') }}</button>
    </div>

    <!-- Scoped to a namespace -->
    <div *transloco="let t; read: 'auth'">
      <button>{{ t('login') }}</button>
    </div>
  `,
})
export class HomeComponent {
  user = input.required<User>();
}
```

### Language Switching

```typescript
import { TranslocoService } from '@ngneat/transloco';

@Component({
  template: `
    <select (change)="changeLang($event)">
      <option value="en">English</option>
      <option value="fr">Français</option>
      <option value="de">Deutsch</option>
      <option value="ar">العربية</option>
    </select>
  `,
})
export class LanguageSwitcherComponent {
  private transloco = inject(TranslocoService);

  changeLang(event: Event) {
    const lang = (event.target as HTMLSelectElement).value;
    this.transloco.setActiveLang(lang);
    localStorage.setItem('preferred-lang', lang);
  }
}
```

---

## 9. Locale-Aware Pipes

Angular's built-in pipes use the configured `LOCALE_ID` for formatting:

```typescript
// app.config.ts — configure locale
import { LOCALE_ID } from '@angular/core';
import { registerLocaleData } from '@angular/common';
import localeFr from '@angular/common/locales/fr';

registerLocaleData(localeFr);

providers: [
  { provide: LOCALE_ID, useValue: 'fr-FR' }
]
```

---

## 10. Date, Number, and Currency Formatting

### DatePipe

```html
<!-- Uses LOCALE_ID for formatting -->
{{ today | date }}                     <!-- Jan 15, 2026 (en) / 15 janv. 2026 (fr) -->
{{ today | date:'short' }}             <!-- 1/15/26, 3:30 PM (en) -->
{{ today | date:'longDate' }}          <!-- January 15, 2026 (en) -->
{{ today | date:'dd/MM/yyyy' }}        <!-- 15/01/2026 -->
{{ today | date:'EEEE, MMMM d, y' }}   <!-- Thursday, January 15, 2026 -->

<!-- Explicit locale override -->
{{ today | date:'short':'':'fr' }}     <!-- 15/01/2026 14:30 -->
```

### CurrencyPipe

```html
{{ 1234.56 | currency }}              <!-- $1,234.56 (en-US) -->
{{ 1234.56 | currency:'EUR' }}        <!-- €1,234.56 -->
{{ 1234.56 | currency:'GBP':'symbol-narrow' }} <!-- £1,234.56 -->
{{ 1234.56 | currency:'JPY':'symbol':'1.0-0' }} <!-- ¥1,235 -->
```

### DecimalPipe / PercentPipe

```html
{{ 3.14159 | number:'1.2-4' }}   <!-- 3.1416 (min 1 integer, 2-4 decimal) -->
{{ 0.75 | percent }}              <!-- 75% -->
{{ 0.7534 | percent:'1.1-2' }}   <!-- 75.3% -->
```

---

## 11. RTL (Right-to-Left) Support

For Arabic, Hebrew, Persian (Farsi) languages:

```html
<!-- index.html — set dir attribute -->
<html dir="{{ isRTL ? 'rtl' : 'ltr' }}" lang="{{ currentLang }}">
```

```typescript
@Injectable({ providedIn: 'root' })
export class DirectionService {
  private rtlLanguages = new Set(['ar', 'he', 'fa', 'ur']);
  isRTL = computed(() =>
    this.rtlLanguages.has(this.transloco.getActiveLang())
  );
}
```

### CSS Logical Properties for RTL

```css
/* ❌ Physical properties — break in RTL */
.card { margin-left: 16px; padding-right: 8px; }
.icon { float: left; }

/* ✅ Logical properties — work in both LTR and RTL */
.card { margin-inline-start: 16px; padding-inline-end: 8px; }
.icon { float: inline-start; }
```

---

## 12. Best Practices

1. **Use `@@id` for all translation units** — stable IDs prevent losing translations when source text changes.
2. **Use Transloco for runtime i18n** over `@angular/localize` when language switching is needed.
3. **Separate translation keys by feature** (`auth.login`, `cart.empty`) — avoids key collisions.
4. **Never hardcode locale strings in components** — always use a translation mechanism.
5. **Register locale data** for all languages you support (`registerLocaleData(localeFr)`).
6. **Use CSS logical properties** for RTL support from the start.
7. **Test with pseudo-localization** — replaces characters to verify all text is translatable.
8. **Keep translation files in source control** — they're part of the app, not a build artifact.

---

> **Summary:** Angular's i18n ecosystem offers two paths: build-time (`@angular/localize`) for maximum performance with multiple builds, and runtime (`Transloco`) for flexibility and dynamic language switching. Either way, proper use of locale-aware pipes (`DatePipe`, `CurrencyPipe`, `DecimalPipe`), CSS logical properties for RTL, and stable translation IDs are the keys to a maintainable multilingual Angular application.
