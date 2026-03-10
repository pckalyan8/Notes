# 🏗️ PHASE 2 — HTML Mastery
### From Zero to Deep HTML Expertise
#### Angular Frontend Mastery Roadmap — Study Guide

> **Goal of this phase:** Understand HTML not just as "tags you write" but as a structured, semantic, accessible language that forms the backbone of every web page. Deep HTML knowledge makes your CSS and JavaScript dramatically more effective.

---

## TABLE OF CONTENTS

1. [2.1 — HTML Basics](#21-html-basics)
2. [2.2 — Semantic HTML](#22-semantic-html)
3. [2.3 — HTML Forms](#23-html-forms)
4. [2.4 — HTML Media](#24-html-media)
5. [2.5 — HTML Tables](#25-html-tables)
6. [2.6 — HTML Metadata & Head Elements](#26-html-metadata--head-elements)
7. [2.7 — HTML APIs & Interfaces](#27-html-apis--interfaces)
8. [2.8 — HTML5 Advanced](#28-html5-advanced)

---

## 2.1 HTML Basics

### What is HTML?

HTML (HyperText Markup Language) is the standard language for creating web pages. It describes the **structure** and **meaning** of content — not its appearance (that's CSS) or behavior (that's JavaScript). Think of HTML as the skeleton of a building: it defines what rooms exist and how they connect, but not the paint color or lighting.

HTML was created by Tim Berners-Lee in 1991 and has evolved through HTML 2, 3.2, 4.01, XHTML, and finally the living standard we use today: **HTML5** (maintained by WHATWG).

---

### HTML Document Structure

Every valid HTML document follows a specific structure. Here is the minimal skeleton:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>My Page</title>
  </head>
  <body>
    <h1>Hello, World!</h1>
    <p>This is a paragraph.</p>
  </body>
</html>
```

Let's break down each part:

**`<!DOCTYPE html>`** — This is the Document Type Declaration. It tells the browser to render the page using the modern HTML5 standard. Without it, browsers fall into "quirks mode," where they try to imitate old, inconsistent behavior from the 1990s. Always include this as the very first line.

**`<html lang="en">`** — The root element that wraps everything. The `lang` attribute is critically important for accessibility: screen readers use it to know which language to speak the content in. Always set it to the appropriate BCP 47 language tag (e.g., `"en"`, `"fr"`, `"ar"`, `"zh-CN"`).

**`<head>`** — Contains metadata about the page — information that is not displayed directly in the browser window but is essential for the browser, search engines, and social platforms. This is where you put your `<title>`, `<meta>` tags, and `<link>` tags.

**`<body>`** — Contains everything that is actually rendered and visible to the user: text, images, forms, navigation, etc.

---

### Tags, Elements, and Attributes

Understanding the difference between these three terms is foundational:

An **element** is the complete unit: the opening tag + content + closing tag. A **tag** is just the markup notation (the `<>` syntax). An **attribute** provides additional information about an element and always lives inside the opening tag.

```html
<!-- This is an element. The tag is <a>, the attribute is href, 
     and "Click here" is the content -->
<a href="https://example.com">Click here</a>

<!-- Self-closing element (void element) — no closing tag needed -->
<img src="photo.jpg" alt="A sunset over mountains" />
<input type="text" placeholder="Enter your name" />
<br />
<hr />
```

**Void elements** (also called self-closing elements) are elements that cannot have any child content by definition — `<img>`, `<input>`, `<br>`, `<hr>`, `<meta>`, `<link>`. In HTML5, the trailing slash (`/>`) is optional but recommended for clarity.

---

### Block vs Inline Elements

This is one of the most important foundational concepts, because it directly affects layout behavior in CSS.

**Block-level elements** start on a new line and take up the full width available by default. They stack vertically. Examples: `<div>`, `<p>`, `<h1>`–`<h6>`, `<ul>`, `<ol>`, `<li>`, `<header>`, `<section>`, `<article>`, `<form>`, `<table>`.

**Inline elements** flow within the text, side by side, and only take up as much width as their content requires. Examples: `<span>`, `<a>`, `<strong>`, `<em>`, `<img>`, `<input>`, `<label>`, `<code>`.

```html
<!-- Block: each paragraph appears on its own line -->
<p>First paragraph.</p>
<p>Second paragraph.</p>

<!-- Inline: these stay within the flow of text -->
<p>
  This text has a <strong>bold word</strong> and an
  <a href="#">inline link</a> inside it.
</p>
```

> **Important:** You can change this behavior entirely with CSS (`display: inline`, `display: block`, `display: flex`), but the HTML default behavior affects how browsers parse the document and how accessibility tools interpret it.

---

### Comments and Whitespace

```html
<!-- This is an HTML comment. It is not rendered in the browser. -->
<!-- Use comments to explain complex structures, 
     not to comment out production code -->

<p>
  HTML collapses multiple spaces and line breaks
  into a single space.     This text has extra spaces
  but will render normally.
</p>
```

**Whitespace collapsing** is a key behavior to internalize: browsers treat sequences of whitespace (spaces, tabs, newlines) as a single space. This means your HTML source can be indented beautifully for readability without affecting rendering. The exception is the `<pre>` element, which preserves all whitespace exactly.

---

### ✅ Best Practices for HTML Basics

Always declare `<!DOCTYPE html>`. Always set `lang` on the `<html>` element. Use proper indentation (2 or 4 spaces) consistently. Quote all attribute values (both `"double"` and `'single'` quotes are valid, but pick one style and be consistent — double quotes are conventional). Write lowercase tag and attribute names. Include `alt` text on every `<img>`. Validate your HTML with the [W3C Validator](https://validator.w3.org/) during learning.

---

## 2.2 Semantic HTML

### What Does "Semantic" Mean?

Semantic HTML means using HTML elements that carry **meaning** about the role and purpose of their content — not just elements that look a certain way. The word semantic comes from "semantics," meaning the meaning of language.

Compare these two code blocks — they may look identical in a browser with basic CSS, but they are fundamentally different in meaning:

```html
<!-- Non-semantic: tells the browser nothing about purpose -->
<div class="header">
  <div class="nav">
    <div class="nav-item">Home</div>
    <div class="nav-item">About</div>
  </div>
</div>

<!-- Semantic: self-describing, meaningful structure -->
<header>
  <nav>
    <ul>
      <li><a href="/">Home</a></li>
      <li><a href="/about">About</a></li>
    </ul>
  </nav>
</header>
```

The semantic version communicates to browsers, screen readers, search engine crawlers, and other developers exactly what each piece of content is and how it relates to the rest of the page.

---

### Why Semantics Matter

**Accessibility:** Screen readers announce landmark regions (header, nav, main, footer) to users, allowing them to jump directly to the content they need. A blind user can press a keyboard shortcut to skip straight to `<main>` without reading through navigation. This is impossible with a `<div>`-based layout.

**SEO:** Search engines give more weight to content inside semantically appropriate elements. A heading in an `<h1>` is more important to a search crawler than text in a styled `<div>`.

**Maintainability:** Semantic HTML is self-documenting. A developer joining your project immediately understands the page structure without needing to decode CSS class names.

**Browser default styles:** Browsers apply sensible default styles to semantic elements (headings are bold and large, `<strong>` is bold, `<em>` is italic), giving you a reasonable baseline.

---

### Structural Landmark Elements

These define the major regions of a page:

```html
<body>

  <!-- Page-level header: logo, site name, global navigation -->
  <header>
    <a href="/" aria-label="Home">
      <img src="logo.svg" alt="Company Logo" />
    </a>
    <nav aria-label="Main navigation">
      <ul>
        <li><a href="/">Home</a></li>
        <li><a href="/products">Products</a></li>
        <li><a href="/contact">Contact</a></li>
      </ul>
    </nav>
  </header>

  <!-- The primary unique content of the page. 
       There should be ONLY ONE <main> per page. -->
  <main>

    <!-- A self-contained piece of content that makes 
         sense on its own (blog post, news article, forum post) -->
    <article>
      <header>
        <!-- <header> inside <article> is the article's header, not the page header -->
        <h1>Getting Started with Angular Signals</h1>
        <time datetime="2026-01-15">January 15, 2026</time>
        <address>By <a href="/authors/jane">Jane Smith</a></address>
      </header>

      <p>Angular Signals represent a fundamental shift...</p>

      <!-- A thematically related group of content within a larger document.
           Should typically have its own heading. -->
      <section>
        <h2>What are Signals?</h2>
        <p>A signal is a reactive primitive that...</p>
      </section>

      <section>
        <h2>Creating Your First Signal</h2>
        <p>To create a signal, import the signal function...</p>
      </section>

      <footer>
        <!-- article's footer: tags, share buttons, related links -->
        <p>Tags: Angular, Signals, Reactivity</p>
      </footer>
    </article>

    <!-- Tangentially related content: sidebars, 
         pull quotes, related articles -->
    <aside>
      <h2>Related Articles</h2>
      <ul>
        <li><a href="/rxjs-vs-signals">RxJS vs Signals</a></li>
      </ul>
    </aside>

  </main>

  <!-- Page-level footer: copyright, links, legal -->
  <footer>
    <p>&copy; 2026 Company Inc.</p>
    <nav aria-label="Footer navigation">
      <a href="/privacy">Privacy Policy</a>
      <a href="/terms">Terms of Service</a>
    </nav>
  </footer>

</body>
```

**Key distinction: `<article>` vs `<section>`**

- `<article>` is for self-contained, independently distributable content. Ask yourself: "Could this be published on its own RSS feed or synced to another site and still make sense?" If yes → `<article>`. Blog posts, news stories, forum posts, comments, product cards, and tweets all qualify.
- `<section>` is for thematically grouping related content within a larger document. It needs a heading. Use it to divide a long article into chapters, or a homepage into "Features," "Pricing," "Testimonials" blocks.
- `<div>` is the fallback for when no semantic element fits. Use it purely for styling/layout purposes.

---

### Heading Hierarchy

Headings (`<h1>` through `<h6>`) define the document outline. This outline is how screen readers navigate a page, how search engines understand topic hierarchy, and how automated tools generate tables of contents.

```html
<!-- CORRECT: logical, nested hierarchy -->
<h1>The Complete Guide to TypeScript</h1>       <!-- Page title: only ONE h1 -->
  <h2>Part 1: Basic Types</h2>
    <h3>Primitive Types</h3>
    <h3>Object Types</h3>
  <h2>Part 2: Advanced Types</h2>
    <h3>Generics</h3>
      <h4>Generic Constraints</h4>
      <h4>Default Type Parameters</h4>
    <h3>Mapped Types</h3>

<!-- WRONG: skipping heading levels is confusing for assistive tech -->
<h1>Page Title</h1>
<h3>Subsection</h3>  <!-- Skipped h2! -->
<h5>Detail</h5>      <!-- Skipped h4! -->
```

> **One `<h1>` per page** is the strong recommendation. It represents the topic of the entire page. Think of it like a book title — you only have one.

---

### Lists: `<ul>`, `<ol>`, `<dl>`

```html
<!-- Unordered list: items without meaningful order (navigation, feature lists) -->
<ul>
  <li>Angular</li>
  <li>TypeScript</li>
  <li>RxJS</li>
</ul>

<!-- Ordered list: items with meaningful sequence (steps, rankings) -->
<ol>
  <li>Install Node.js</li>
  <li>Install Angular CLI: <code>npm install -g @angular/cli</code></li>
  <li>Create a new project: <code>ng new my-app</code></li>
</ol>

<!-- Definition list: key-value pairs, glossaries, metadata -->
<dl>
  <dt>Signal</dt>
  <dd>A reactive primitive that holds a value and notifies consumers when the value changes.</dd>
  
  <dt>Computed</dt>
  <dd>A derived signal that automatically recalculates when its dependencies change.</dd>
  
  <dt>Effect</dt>
  <dd>A side-effect function that runs whenever its signal dependencies change.</dd>
</dl>
```

---

### Other Important Semantic Elements

```html
<!-- Quotations -->
<blockquote cite="https://angular.dev">
  <p>Angular is a platform for building mobile and desktop web applications.</p>
  <footer>— <cite>angular.dev</cite></footer>
</blockquote>

<p>As the saying goes, <q>premature optimization is the root of all evil</q>.</p>

<!-- Code and technical content -->
<p>Run <code>ng generate component</code> to create a new component.</p>

<pre><code>
const count = signal(0);
count.set(1);
</code></pre>

<!-- Keyboard input -->
<p>Press <kbd>Ctrl</kbd> + <kbd>S</kbd> to save.</p>

<!-- Emphasis vs importance -->
<p>
  Angular is <em>really</em> different from AngularJS.  <!-- Stress emphasis, voice inflection -->
  This is <strong>critically important</strong> to understand.  <!-- Serious importance/urgency -->
</p>

<!-- Time -->
<time datetime="2026-03-15T09:00">March 15, 2026</time>

<!-- Abbreviations -->
<abbr title="Single Page Application">SPA</abbr>

<!-- Mark/highlight -->
<p>Search results for "Angular": found <mark>Angular</mark> Signals.</p>

<!-- Small print (legal, copyright) -->
<small>&copy; 2026 Company Inc. All rights reserved.</small>

<!-- Figure and caption (images, diagrams, code samples) -->
<figure>
  <img src="signal-diagram.png" alt="Diagram showing signal data flow" />
  <figcaption>Fig. 1: How Angular Signals propagate changes</figcaption>
</figure>
```

---

## 2.3 HTML Forms

### Why Forms Are Critical

Forms are how users interact with your application — authentication, search, checkout, settings, data entry. A well-built HTML form leverages the browser's built-in validation, accessibility features, and keyboard behavior for free. A poorly built form creates UX nightmares that require massive JavaScript workarounds.

---

### The `<form>` Element

```html
<form 
  action="/api/login"      <!-- Where to send the data (URL) -->
  method="POST"            <!-- HTTP method: GET or POST -->
  enctype="multipart/form-data"  <!-- Required for file uploads -->
  novalidate               <!-- Disable browser validation (for custom validation) -->
>
  <!-- form fields go here -->
</form>
```

In Angular, you rarely use `action` and `method` because Angular handles form submission with JavaScript. But understanding the HTML behavior is important for SSR and fallback scenarios.

---

### Input Types — The Complete Reference

HTML5 introduced many new input types that give you semantic meaning, mobile keyboard optimization, and browser-native validation:

```html
<!-- Text inputs -->
<input type="text" placeholder="Full name" />
<input type="email" placeholder="you@example.com" />     <!-- Validates email format -->
<input type="password" />                                <!-- Masks input -->
<input type="search" placeholder="Search..." />          <!-- Shows clear button in some browsers -->
<input type="url" placeholder="https://example.com" />   <!-- Validates URL format -->
<input type="tel" placeholder="+1 555-0100" />           <!-- Shows number pad on mobile -->

<!-- Numeric inputs -->
<input type="number" min="0" max="100" step="5" />       <!-- Numeric spinner -->
<input type="range" min="0" max="100" value="50" />      <!-- Slider -->

<!-- Date and time inputs -->
<input type="date" min="2020-01-01" max="2026-12-31" />  <!-- Date picker -->
<input type="time" />
<input type="datetime-local" />
<input type="month" />
<input type="week" />

<!-- Selection inputs -->
<input type="checkbox" id="agree" name="agree" value="yes" />
<input type="radio" id="opt1" name="option" value="option1" />

<!-- Special inputs -->
<input type="color" value="#ff6600" />                   <!-- Color picker -->
<input type="file" accept=".pdf,.doc" multiple />        <!-- File upload -->
<input type="hidden" name="csrf_token" value="abc123" /> <!-- Hidden data -->

<!-- Buttons (prefer <button> element instead) -->
<input type="submit" value="Submit" />
<input type="reset" value="Clear Form" />
<input type="button" value="Click Me" />
```

> **Mobile keyboard tip:** Using the correct input type (`email`, `tel`, `number`, `url`) automatically shows the optimized keyboard on iOS and Android — no JavaScript needed. This is a free UX improvement many developers miss.

---

### Labels — The Most Overlooked Element

Every form control **must** have an associated label. This is both an accessibility requirement and good UX — clicking the label focuses/activates the associated input.

```html
<!-- Method 1: explicit association with for/id (preferred) -->
<label for="username">Username</label>
<input type="text" id="username" name="username" />

<!-- Method 2: wrapping (implicit association) -->
<label>
  Email address
  <input type="email" name="email" />
</label>

<!-- Method 3: aria-label (when no visible label is possible) -->
<input type="search" aria-label="Search the site" />

<!-- Method 4: aria-labelledby (label text lives elsewhere) -->
<h2 id="billing-heading">Billing Address</h2>
<input type="text" aria-labelledby="billing-heading" />

<!-- WRONG: placeholder is NOT a label substitute -->
<!-- The placeholder disappears on input, leaving users confused -->
<input type="text" placeholder="Enter username" />  <!-- No label = accessibility failure -->
```

---

### Select, Textarea, and Other Controls

```html
<!-- Dropdown select -->
<label for="country">Country</label>
<select id="country" name="country">
  <option value="">-- Select country --</option>          <!-- Default/placeholder option -->
  <optgroup label="North America">                        <!-- Grouping options -->
    <option value="us">United States</option>
    <option value="ca">Canada</option>
    <option value="mx" selected>Mexico</option>           <!-- Pre-selected -->
  </optgroup>
  <optgroup label="Europe">
    <option value="gb">United Kingdom</option>
    <option value="de">Germany</option>
  </optgroup>
</select>

<!-- Multi-select -->
<select name="skills" multiple size="5">
  <option value="angular">Angular</option>
  <option value="react">React</option>
  <option value="vue">Vue</option>
</select>

<!-- Textarea -->
<label for="bio">Biography</label>
<textarea 
  id="bio" 
  name="bio" 
  rows="5" 
  cols="40"
  maxlength="500"
  placeholder="Tell us about yourself..."
></textarea>

<!-- Fieldset and legend (grouping related controls) -->
<fieldset>
  <legend>Notification Preferences</legend>
  
  <input type="checkbox" id="email-notif" name="notif" value="email" />
  <label for="email-notif">Email notifications</label>
  
  <input type="checkbox" id="sms-notif" name="notif" value="sms" />
  <label for="sms-notif">SMS notifications</label>
</fieldset>

<!-- Datalist: input with autocomplete suggestions -->
<label for="framework">Framework</label>
<input type="text" id="framework" list="frameworks" />
<datalist id="frameworks">
  <option value="Angular" />
  <option value="React" />
  <option value="Vue" />
  <option value="Svelte" />
</datalist>

<!-- Button element (preferred over input type=submit) -->
<button type="submit">Submit Form</button>
<button type="reset">Reset</button>
<button type="button" onclick="handleClick()">Custom Action</button>
```

> **`<button>` vs `<input type="button">`:** Always prefer `<button>`. It can contain HTML content (icons, formatted text), supports `::before`/`::after` pseudo-elements, and is semantically clearer. Reserve `<input type="submit">` only when working with legacy code.

---

### HTML5 Form Validation

The browser can validate many common patterns natively without any JavaScript:

```html
<form>
  <!-- Required field -->
  <input type="text" name="name" required />

  <!-- Length constraints -->
  <input type="password" name="password" minlength="8" maxlength="64" required />

  <!-- Numeric range -->
  <input type="number" name="age" min="18" max="120" required />

  <!-- Custom pattern (regex) -->
  <input 
    type="text" 
    name="postcode" 
    pattern="[A-Z]{1,2}[0-9]{1,2} ?[0-9][A-Z]{2}"
    title="Please enter a valid UK postcode (e.g. SW1A 1AA)"
  />

  <!-- Email (built-in format validation) -->
  <input type="email" name="email" required />

  <!-- You can style validation states with CSS pseudo-classes -->
  <!-- input:valid { border-color: green; }   -->
  <!-- input:invalid { border-color: red; }   -->
  <!-- input:required { border-left: 3px solid orange; } -->

  <button type="submit">Submit</button>
</form>
```

In Angular, you'll typically disable browser validation with `novalidate` and use Angular's Reactive Forms validation instead — but it's essential to understand the native capability because it's free, performant, and accessible.

---

## 2.4 HTML Media

### Images: The Deep Dive

Images are often the largest assets on a page and critical for performance. Modern HTML gives you powerful tools to serve the right image to the right device.

```html
<!-- Basic image: always include alt text -->
<img 
  src="hero.jpg" 
  alt="A developer writing code on a laptop in a coffee shop"
  width="800"     <!-- Specify width and height to prevent layout shift (CLS) -->
  height="600"
/>

<!-- alt="" (empty) for decorative images that add no information -->
<img src="decorative-divider.png" alt="" />
```

**The `alt` attribute rules:**
- For informative images: describe what the image shows and why it's there.
- For decorative images (pure visual flourish): use `alt=""` so screen readers skip it.
- For images that are links: describe the link destination, not the image (`<a href="/home"><img src="logo.png" alt="Return to homepage" /></a>`).
- Never say "image of" or "photo of" — screen readers already announce it as an image.

---

### Responsive Images with `srcset` and `sizes`

This is one of the most impactful performance techniques you can apply. Instead of loading a single large image for all devices, you provide multiple sizes and let the browser choose the best one.

```html
<!-- Resolution switching: same image, different sizes -->
<img 
  src="photo-800.jpg"                    <!-- Fallback for old browsers -->
  srcset="
    photo-400.jpg 400w,                  <!-- 400px wide version -->
    photo-800.jpg 800w,                  <!-- 800px wide version -->
    photo-1600.jpg 1600w                 <!-- 1600px wide version -->
  "
  sizes="
    (max-width: 600px) 100vw,            <!-- On mobile: use full viewport width -->
    (max-width: 1200px) 50vw,            <!-- On tablet: use 50% of viewport -->
    800px                                <!-- On desktop: always 800px -->
  "
  alt="Mountain landscape"
  width="800"
  height="600"
  loading="lazy"                         <!-- Defer off-screen images -->
/>
```

The `srcset` tells the browser which image files are available and their widths. The `sizes` tells the browser how large the image will actually be displayed at different viewport widths. The browser combines these with the device's pixel density to pick the optimal file.

---

### The `<picture>` Element — Art Direction and Format Selection

Use `<picture>` when you need to serve different image formats or crops based on screen size or browser support:

```html
<!-- Format switching: serve AVIF to browsers that support it, 
     WebP as fallback, JPEG as final fallback -->
<picture>
  <source srcset="photo.avif" type="image/avif" />   <!-- AVIF: best compression, ~2026 widely supported -->
  <source srcset="photo.webp" type="image/webp" />   <!-- WebP: widely supported -->
  <img src="photo.jpg" alt="A serene lake at sunrise" loading="lazy" />
</picture>

<!-- Art direction: different crops for different screen sizes -->
<picture>
  <!-- Portrait crop for mobile -->
  <source 
    media="(max-width: 768px)" 
    srcset="hero-portrait.jpg"
  />
  <!-- Wide landscape crop for desktop -->
  <img 
    src="hero-landscape.jpg" 
    alt="Our team at the annual conference"
  />
</picture>
```

> **Performance note:** AVIF offers 50% smaller file sizes compared to JPEG with equivalent quality. WebP offers ~30% reduction. Always provide `<img>` as the fallback inside `<picture>` — it's what non-supporting browsers use.

---

### Video

```html
<video 
  width="720" 
  height="405"
  controls             <!-- Show playback controls -->
  autoplay             <!-- Start playing immediately (use with muted!) -->
  muted                <!-- Required for autoplay to work in most browsers -->
  loop                 <!-- Repeat when it ends -->
  poster="thumbnail.jpg"  <!-- Image shown before video loads -->
  preload="metadata"   <!-- Load just metadata (duration), not the full video -->
>
  <!-- Multiple source formats for cross-browser support -->
  <source src="video.webm" type="video/webm" />  <!-- Preferred: better compression -->
  <source src="video.mp4" type="video/mp4" />    <!-- Fallback: universal support -->
  
  <!-- Captions for accessibility (WCAG requirement) -->
  <track 
    kind="subtitles" 
    src="captions-en.vtt" 
    srclang="en" 
    label="English"
    default              <!-- Load this track by default -->
  />
  <track kind="subtitles" src="captions-es.vtt" srclang="es" label="Spanish" />
  
  <!-- Fallback message for very old browsers -->
  <p>Your browser doesn't support HTML5 video. 
     <a href="video.mp4">Download the video</a> instead.</p>
</video>
```

> **Autoplay policy:** Modern browsers block autoplay of videos with sound. If you need background/decorative videos, always use `autoplay muted`. This also respects users on metered connections.

---

### Audio

```html
<audio controls>
  <source src="podcast.ogg" type="audio/ogg" />
  <source src="podcast.mp3" type="audio/mpeg" />
  <p>Your browser doesn't support audio playback.</p>
</audio>
```

---

### iframes and Security

`<iframe>` embeds another HTML document. The key security attribute is `sandbox`:

```html
<!-- Without sandbox: full access to the parent page's context (dangerous) -->
<iframe src="https://untrusted-site.com"></iframe>

<!-- With sandbox: severely restricts what the embedded content can do -->
<iframe 
  src="https://untrusted-widget.com"
  sandbox="allow-scripts allow-same-origin"
  <!-- Other sandbox values:
       allow-forms         → Allow form submissions
       allow-popups        → Allow new windows
       allow-top-navigation → Allow navigating the top window
       allow-modals        → Allow alert/confirm dialogs
  -->
  title="Weather widget"   <!-- Required for accessibility: describes the frame -->
  loading="lazy"           <!-- Defer loading until visible -->
  width="400"
  height="300"
></iframe>

<!-- Common legitimate use: embedded maps, videos, third-party widgets -->
<iframe 
  src="https://www.youtube.com/embed/VIDEO_ID"
  title="Introduction to Angular Signals (YouTube video)"
  allowfullscreen
  loading="lazy"
></iframe>
```

---

## 2.5 HTML Tables

### When to Use Tables

Tables are for **tabular data** — data that has a meaningful relationship across both rows and columns. They are NOT for layout (that's a legacy practice from the 1990s that caused enormous accessibility and maintenance problems).

Valid table use cases: financial reports, comparison charts, schedules, data grids, sports standings.

---

### Full Table Structure

```html
<table>
  <!-- Caption: the table's title (accessible, useful for screen readers) -->
  <caption>Q1 2026 Revenue by Region</caption>

  <!-- Table header group -->
  <thead>
    <tr>
      <!-- scope="col" tells screen readers this header applies to its column -->
      <th scope="col">Region</th>
      <th scope="col">Revenue</th>
      <th scope="col">Growth</th>
      <th scope="col">Target</th>
    </tr>
  </thead>

  <!-- Table body (the actual data) -->
  <tbody>
    <tr>
      <!-- scope="row" for row headers -->
      <th scope="row">North America</th>
      <td>$4.2M</td>
      <td>+12%</td>
      <td>$4.0M</td>
    </tr>
    <tr>
      <th scope="row">Europe</th>
      <td>$3.1M</td>
      <td>+8%</td>
      <td>$3.5M</td>
    </tr>
  </tbody>

  <!-- Table footer (totals, summaries) -->
  <tfoot>
    <tr>
      <th scope="row">Total</th>
      <td>$7.3M</td>
      <td>+10%</td>
      <td>$7.5M</td>
    </tr>
  </tfoot>
</table>
```

---

### Spanning Cells

```html
<table>
  <caption>Weekly Schedule</caption>
  <thead>
    <tr>
      <th scope="col">Time</th>
      <th scope="col">Monday</th>
      <th scope="col">Tuesday</th>
      <th scope="col">Wednesday</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">9:00 AM</th>
      <!-- colspan spans across multiple columns -->
      <td colspan="2">Team standup (Mon-Tue)</td>
      <td>1:1 with manager</td>
    </tr>
    <tr>
      <th scope="row">10:00 AM</th>
      <td>Deep work</td>
      <!-- rowspan spans across multiple rows -->
      <td rowspan="2">Design review (2hrs)</td>
      <td>Deep work</td>
    </tr>
    <tr>
      <th scope="row">11:00 AM</th>
      <td>Deep work</td>
      <!-- Cell omitted because of rowspan above -->
      <td>Code review</td>
    </tr>
  </tbody>
</table>
```

---

## 2.6 HTML Metadata & Head Elements

The `<head>` section is invisible to users but profoundly important. It's where you configure how your page is interpreted, indexed, and shared.

---

### Essential Meta Tags

```html
<head>
  <!-- Character encoding: must be first in <head>, within first 1024 bytes -->
  <meta charset="UTF-8" />

  <!-- Viewport: CRITICAL for responsive design. Without this, 
       mobile browsers render at desktop width and scale down -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />

  <!-- Page title: shown in browser tab, bookmarks, and search results.
       Should be unique per page, 50-60 characters max -->
  <title>Angular Signals Guide — Frontend Mastery</title>

  <!-- Meta description: shown in search result snippets, 150-160 chars -->
  <meta 
    name="description" 
    content="A comprehensive guide to Angular Signals, covering signal creation, computed values, effects, and real-world patterns for reactive state management."
  />

  <!-- Canonical URL: prevents duplicate content SEO issues 
       when same content is accessible from multiple URLs -->
  <link rel="canonical" href="https://example.com/angular-signals-guide" />
  
  <!-- Robots: control search engine indexing -->
  <meta name="robots" content="index, follow" />
  <!-- Or to prevent indexing: content="noindex, nofollow" -->
</head>
```

---

### Open Graph and Twitter Card (Social Sharing)

These tags control how your page appears when shared on social media platforms like LinkedIn, Twitter/X, Facebook, Slack, and iMessage:

```html
<!-- Open Graph (used by Facebook, LinkedIn, Discord, Slack, iMessage) -->
<meta property="og:type" content="article" />
<meta property="og:title" content="Angular Signals: The Complete Guide" />
<meta property="og:description" content="Master Angular's new reactive primitive with practical examples." />
<meta property="og:image" content="https://example.com/og-image-signals.jpg" />
<meta property="og:image:width" content="1200" />
<meta property="og:image:height" content="630" />
<meta property="og:url" content="https://example.com/angular-signals" />
<meta property="og:site_name" content="Frontend Mastery" />

<!-- Twitter/X Cards -->
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:site" content="@yourtwitterhandle" />
<meta name="twitter:title" content="Angular Signals: The Complete Guide" />
<meta name="twitter:description" content="Master Angular's new reactive primitive." />
<meta name="twitter:image" content="https://example.com/twitter-card-signals.jpg" />
```

> **OG image size:** The standard is 1200×630px. It's shown when your URL is pasted in Slack, Teams, WhatsApp, or social media. A missing OG image means your shared link looks bare.

---

### Link Tags: Stylesheet, Icons, Preloading

```html
<!-- Stylesheet link -->
<link rel="stylesheet" href="styles.css" />

<!-- Favicon: browser tab icon -->
<link rel="icon" type="image/svg+xml" href="/favicon.svg" />
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png" />
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png" />

<!-- PWA manifest -->
<link rel="manifest" href="/manifest.webmanifest" />

<!-- Preconnect: establish connection to third-party domains early.
     Use for domains you'll definitely load resources from. -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

<!-- Preload: load critical resources immediately (hero image, key font) -->
<link 
  rel="preload" 
  href="/fonts/inter-var.woff2" 
  as="font" 
  type="font/woff2" 
  crossorigin
/>
<link rel="preload" href="hero-image.avif" as="image" />

<!-- Prefetch: load resources that will be needed on the NEXT page -->
<link rel="prefetch" href="/products-page-bundle.js" />
```

---

### Script Loading Strategies

```html
<!-- Default: blocks HTML parsing until script downloads and executes.
     AVOID putting scripts in <head> without async/defer. -->
<script src="app.js"></script>

<!-- async: downloads in parallel, executes as soon as downloaded.
     Order not guaranteed. Good for independent scripts (analytics). -->
<script src="analytics.js" async></script>

<!-- defer: downloads in parallel, executes AFTER HTML is fully parsed.
     Order is preserved. BEST CHOICE for most scripts. -->
<script src="app.js" defer></script>

<!-- ES Module: automatically deferred. Enables import/export. -->
<script type="module" src="main.js"></script>

<!-- Inline script (no src needed for defer/async, executes immediately) -->
<script>
  // Inline scripts run synchronously during HTML parsing
  window.__APP_CONFIG__ = { apiUrl: 'https://api.example.com' };
</script>
```

**Loading order summary:**

| Type | Download | Execution timing | Order preserved |
|------|----------|-----------------|-----------------|
| Normal `<script>` | Blocks parsing | Immediately | Yes |
| `async` | Parallel | When downloaded | No |
| `defer` | Parallel | After HTML parsed | Yes |
| `type="module"` | Parallel | After HTML parsed | Yes |

---

### JSON-LD Structured Data

JSON-LD (JavaScript Object Notation for Linked Data) adds machine-readable context to your content, enabling Google's rich results (star ratings, FAQs, breadcrumbs in search results):

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Angular Signals: The Complete Guide",
  "datePublished": "2026-01-15",
  "author": {
    "@type": "Person",
    "name": "Jane Smith"
  },
  "publisher": {
    "@type": "Organization",
    "name": "Frontend Mastery",
    "logo": {
      "@type": "ImageObject",
      "url": "https://example.com/logo.png"
    }
  }
}
</script>
```

---

## 2.7 HTML APIs & Interfaces

### Custom Data Attributes (`data-*`)

The `data-*` attributes let you embed custom, private data in HTML elements without polluting the global namespace or using non-standard attributes. They're commonly used to pass data to JavaScript or Angular.

```html
<!-- Any attribute starting with "data-" is valid HTML -->
<button 
  data-product-id="SKU-001"
  data-category="electronics"
  data-price="299.99"
  data-in-stock="true"
>
  Add to Cart
</button>

<script>
  const button = document.querySelector('button');
  
  // Access via dataset property (converts kebab-case to camelCase)
  console.log(button.dataset.productId);  // "SKU-001"
  console.log(button.dataset.category);   // "electronics"
  console.log(button.dataset.price);      // "299.99" (always a string!)
  
  // In Angular, you can also read them via @HostBinding or ElementRef
</script>
```

In Angular, `data-*` attributes are useful for passing config to third-party scripts, for testing selectors (`data-testid`), and for non-Angular JavaScript interop.

---

### The `<template>` Element

The `<template>` element holds HTML that is **not rendered** when the page loads. It's a placeholder for content that will be cloned and inserted by JavaScript (or Angular) at runtime. Angular heavily uses this concept internally for `@defer`, `ng-template`, and structural directives.

```html
<!-- Content inside <template> is parsed but not rendered -->
<template id="card-template">
  <div class="card">
    <h3 class="card-title"></h3>
    <p class="card-body"></p>
  </div>
</template>

<script>
  // Clone the template to create real instances
  const template = document.getElementById('card-template');
  const clone = template.content.cloneNode(true);
  clone.querySelector('.card-title').textContent = 'Angular Signals';
  document.body.appendChild(clone);
</script>
```

---

### The `<dialog>` Element

The native `<dialog>` element provides built-in modal/dialog behavior with accessibility features (focus trapping, `aria-modal`, `Escape` key handling) that previously required heavy JavaScript libraries:

```html
<dialog id="confirm-dialog" aria-labelledby="dialog-title">
  <h2 id="dialog-title">Confirm Deletion</h2>
  <p>Are you sure you want to delete this item? This action cannot be undone.</p>
  <div>
    <button id="cancel-btn">Cancel</button>
    <button id="confirm-btn">Delete</button>
  </div>
</dialog>

<button onclick="document.getElementById('confirm-dialog').showModal()">
  Delete Item
</button>

<script>
  const dialog = document.getElementById('confirm-dialog');
  
  document.getElementById('cancel-btn').addEventListener('click', () => {
    dialog.close(); // Closes dialog, returns focus to trigger
  });
  
  document.getElementById('confirm-btn').addEventListener('click', () => {
    // Perform deletion...
    dialog.close('confirmed'); // Can pass a return value
  });
  
  // dialog.show() → non-modal (doesn't block background)
  // dialog.showModal() → modal (blocks background with backdrop)
</script>
```

> **Why this matters for Angular:** Angular Material's `MatDialog` works similarly. Understanding the native `<dialog>` element helps you understand what Angular Material is building on top of, and gives you a lightweight native alternative for simple cases.

---

### Drag and Drop API

```html
<!-- Make an element draggable -->
<div 
  draggable="true" 
  ondragstart="handleDragStart(event)"
  id="drag-item-1"
>
  Draggable Item
</div>

<!-- Define a drop zone -->
<div 
  ondragover="event.preventDefault()" 
  ondrop="handleDrop(event)"
  class="drop-zone"
>
  Drop here
</div>
```

---

### The Popover API

A modern native alternative to custom popover/tooltip libraries (supported in all major browsers since 2024):

```html
<!-- The popover -->
<div id="my-tooltip" popover>
  <p>This is a native popover! No JavaScript needed for basic usage.</p>
</div>

<!-- Button that toggles it -->
<button popovertarget="my-tooltip">Show Tooltip</button>

<!-- You can also specify the action -->
<button popovertarget="my-tooltip" popovertargetaction="show">Show</button>
<button popovertarget="my-tooltip" popovertargetaction="hide">Hide</button>
```

The Popover API provides automatic top-layer rendering (always appears above other content), light-dismiss behavior (click outside to close), and focus management — for free.

---

## 2.8 HTML5 Advanced

### Web Storage (localStorage & sessionStorage)

Both are key-value stores accessible via JavaScript. They differ only in persistence:

```html
<!-- No HTML markup needed — these are JavaScript APIs -->
<script>
  // localStorage: persists until explicitly cleared (survives browser restart)
  localStorage.setItem('theme', 'dark');
  localStorage.setItem('user', JSON.stringify({ id: 42, name: 'Alice' }));
  
  const theme = localStorage.getItem('theme');   // "dark"
  const user = JSON.parse(localStorage.getItem('user'));
  localStorage.removeItem('theme');
  localStorage.clear();  // Remove everything
  
  // sessionStorage: cleared when the browser tab is closed
  sessionStorage.setItem('activeTab', 'profile');
  const tab = sessionStorage.getItem('activeTab');
</script>
```

**Limitations:** Both are synchronous (can block the main thread), string-only (must serialize objects with JSON), limited to ~5MB per origin, accessible by any JavaScript on the same origin (XSS risk for sensitive data). For structured data needing more storage, use IndexedDB.

---

### Geolocation API

```html
<button onclick="getLocation()">Get My Location</button>

<script>
  function getLocation() {
    if ('geolocation' in navigator) {
      navigator.geolocation.getCurrentPosition(
        // Success callback
        (position) => {
          const lat = position.coords.latitude;
          const lng = position.coords.longitude;
          console.log(`Location: ${lat}, ${lng}`);
        },
        // Error callback
        (error) => {
          console.error('Geolocation error:', error.message);
        },
        // Options
        { enableHighAccuracy: true, timeout: 5000, maximumAge: 0 }
      );
    } else {
      console.log('Geolocation not supported');
    }
  }
</script>
```

> **Browser behavior:** This API requires user permission and HTTPS. The browser shows a permission prompt the first time it's called. Always handle the error case (user denies, location unavailable, timeout).

---

### History API (Client-Side Routing)

This is the foundation of how Angular's Router works. The History API lets you change the URL in the browser bar without triggering a full page reload:

```html
<script>
  // pushState: adds a new entry to browser history
  history.pushState(
    { page: 'products', filter: 'angular' },  // State object (arbitrary data)
    '',                                         // Title (mostly ignored by browsers)
    '/products?filter=angular'                 // New URL (must be same origin)
  );
  
  // replaceState: replaces current history entry (no new back-button entry)
  history.replaceState({ page: 'home' }, '', '/');
  
  // Listen for back/forward navigation
  window.addEventListener('popstate', (event) => {
    console.log('Navigation occurred:', event.state);
    // Angular's Router handles this automatically
  });
  
  // navigate programmatically
  history.back();
  history.forward();
  history.go(-2); // Go back 2 entries
</script>
```

Angular's Router uses `pushState` under the hood for every `router.navigate()` call. Understanding this helps you debug routing issues, especially around browser back-button behavior.

---

### Web Workers

Web Workers run JavaScript in a background thread, keeping the main thread free for UI rendering. This is essential for CPU-intensive operations like data processing, image manipulation, or complex calculations:

```html
<!-- Main page (main.html) -->
<script>
  const worker = new Worker('computation-worker.js');
  
  // Send data to the worker
  worker.postMessage({ task: 'sort', data: largeArray });
  
  // Receive results from the worker
  worker.addEventListener('message', (event) => {
    console.log('Result from worker:', event.data);
    renderResults(event.data);  // Update UI on main thread
  });
  
  worker.addEventListener('error', (error) => {
    console.error('Worker error:', error);
  });
</script>
```

```javascript
// computation-worker.js (runs in background thread)
// No access to DOM, window, or document!
self.addEventListener('message', (event) => {
  const { task, data } = event.data;
  
  if (task === 'sort') {
    const sorted = data.sort((a, b) => a - b);  // CPU-intensive work
    self.postMessage(sorted);  // Send result back to main thread
  }
});
```

> **Angular integration:** Angular's CDK doesn't directly abstract Workers, but you can use them in Angular services. Libraries like `Comlink` make the Worker API much easier to use in TypeScript.

---

### MutationObserver

Watches for changes in the DOM tree — element addition/removal, attribute changes, text content changes. Angular uses this internally for some change detection scenarios:

```html
<div id="target">Watch me for changes!</div>

<script>
  const observer = new MutationObserver((mutations) => {
    mutations.forEach((mutation) => {
      if (mutation.type === 'childList') {
        console.log('Child nodes changed:', mutation.addedNodes, mutation.removedNodes);
      }
      if (mutation.type === 'attributes') {
        console.log(`Attribute "${mutation.attributeName}" changed`);
      }
    });
  });
  
  observer.observe(document.getElementById('target'), {
    childList: true,         // Watch for child additions/removals
    subtree: true,           // Watch all descendants
    attributes: true,        // Watch attribute changes
    attributeFilter: ['class', 'data-state'],  // Only these attributes
    characterData: true,     // Watch text content changes
  });
  
  // Stop observing
  observer.disconnect();
</script>
```

---

## ✅ HTML Best Practices Summary

**Structure:** Always use `<!DOCTYPE html>`, `lang` attribute, and proper `<head>`/`<body>`. Validate your HTML at [validator.w3.org](https://validator.w3.org).

**Semantics:** Choose the most semantically appropriate element. Use heading hierarchy logically. Use `<button>` for actions and `<a>` for navigation.

**Accessibility:** Every image needs `alt` text. Every form input needs a label. Use landmark elements. Never rely solely on color to convey information.

**Performance:** Set `width` and `height` on images to prevent CLS. Use `loading="lazy"` for off-screen images. Use `srcset`/`<picture>` for responsive images. Use `defer` on scripts.

**Forms:** Use the correct `input type` for semantic and mobile-keyboard benefits. Always associate labels with inputs. Never use `placeholder` as a label substitute.

**Security:** Use `sandbox` on third-party iframes. Use `rel="noopener noreferrer"` on external links that open in a new tab. Never embed sensitive data in `data-*` attributes.

```html
<!-- External links security best practice -->
<a href="https://external-site.com" target="_blank" rel="noopener noreferrer">
  External Link
</a>
<!-- noopener: prevents the new page from accessing window.opener
     noreferrer: prevents sending the referrer header and implies noopener -->
```

---

*End of Phase 2 — HTML Mastery*

> **Up next:** Phase 3 — CSS Mastery, where you'll learn to control the presentation and layout of all the HTML structures you've just mastered.
