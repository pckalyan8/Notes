# 🎨 PHASE 3 — CSS Mastery
### Complete CSS Expertise: From Fundamentals to Modern Features
#### Angular Frontend Mastery Roadmap — Study Guide

> **Goal of this phase:** CSS is far deeper than "making things look pretty." It is a powerful layout system, animation engine, and design language. Mastering CSS means understanding the cascade, the box model, every modern layout tool, and how to write scalable, maintainable styles at team scale.

---

## TABLE OF CONTENTS

1. [3.1 — CSS Fundamentals](#31-css-fundamentals)
2. [3.2 — CSS Selectors & Combinators](#32-css-selectors--combinators)
3. [3.3 — CSS Layout — Classic](#33-css-layout--classic)
4. [3.4 — CSS Flexbox](#34-css-flexbox)
5. [3.5 — CSS Grid](#35-css-grid)
6. [3.6 — CSS Responsive Design](#36-css-responsive-design)
7. [3.7 — CSS Typography](#37-css-typography)
8. [3.8 — CSS Colors & Backgrounds](#38-css-colors--backgrounds)
9. [3.9 — CSS Animations & Transitions](#39-css-animations--transitions)
10. [3.10 — CSS Modern Features (2024–2026)](#310-css-modern-features-20242026)
11. [3.11 — CSS Architecture & Methodologies](#311-css-architecture--methodologies)
12. [3.12 — CSS Preprocessors](#312-css-preprocessors)
13. [3.13 — CSS in Angular Context](#313-css-in-angular-context)

---

## 3.1 CSS Fundamentals

### What is CSS and How Does It Work?

CSS (Cascading Style Sheets) is the language that controls the visual presentation of HTML. The word "Cascading" is the most important part — it describes how multiple CSS rules from multiple sources are combined, prioritized, and applied to produce the final styled result.

When a browser renders a page, it reads all CSS (from your stylesheets, inline styles, and its own defaults), applies the cascade algorithm to resolve conflicts, and produces a final "computed style" for every element. Understanding this cascade algorithm is the single most important conceptual step in mastering CSS.

---

### CSS Syntax

```css
/* Anatomy of a CSS rule */
selector {
  property: value;
  property: value;
}

/* Real examples */
h1 {
  font-size: 2rem;
  color: #1a1a2e;
  font-weight: 700;
}

.card {
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  padding: 1.5rem;
}
```

A **declaration** is a single `property: value` pair. A **declaration block** is the group wrapped in `{}`. A **rule** is the selector + declaration block together.

---

### How CSS Is Applied: Three Methods

```html
<!-- Method 1: External stylesheet (BEST for most CSS) 
     Cached by browser, shared across pages, maintainable -->
<link rel="stylesheet" href="styles.css" />

<!-- Method 2: Internal (embedded) — useful for page-specific styles
     or email HTML, but generally avoid -->
<style>
  h1 { color: navy; }
</style>

<!-- Method 3: Inline styles — highest specificity, hardest to override.
     Use only for dynamic styles applied via JavaScript/Angular -->
<p style="color: red; font-size: 14px;">Inline styled text</p>
```

---

### The Cascade

The cascade is the algorithm that determines which CSS rule "wins" when multiple rules target the same element and property. It evaluates rules in this order of priority (highest to lowest):

**1. Origin and Importance** — where the rule comes from:
- `!important` rules in user stylesheets (very rare, accessibility overrides)
- `!important` rules in author stylesheets (your CSS)
- Normal rules in author stylesheets (your CSS)
- Normal rules in user stylesheets
- Browser default styles (user agent stylesheet)

**2. Specificity** — how specific the selector is (covered in 3.2).

**3. Order of Appearance** — if two rules have identical specificity, the one that appears **later** in the CSS wins.

```css
/* Both rules target the same element's color */
p { color: blue; }   /* Rule 1 */
p { color: green; }  /* Rule 2 — wins because it appears later */

/* !important overrides everything (use sparingly) */
p { color: red !important; }  /* Wins over all non-!important rules */
```

> **Best practice:** Avoid `!important`. If you find yourself needing it, it usually means your specificity architecture has a problem that should be fixed at the root, not patched with `!important`.

---

### Inheritance

Some CSS properties are **inherited** — their value flows down from parent elements to children automatically. This models how visual properties naturally propagate. Others are **not inherited** because it would be confusing if they did.

```css
/* INHERITED properties (typography-related, text-related) */
/* color, font-family, font-size, font-weight, line-height,
   text-align, text-transform, list-style, cursor, visibility */

body {
  font-family: 'Inter', sans-serif;  /* ALL text on the page inherits this */
  color: #333;                       /* ALL text on the page inherits this */
  line-height: 1.6;
}

/* NOT INHERITED (box model, layout, backgrounds) */
/* width, height, margin, padding, border, background,
   display, position, flex, grid properties */
```

You can explicitly control inheritance with these values:

```css
.child {
  color: inherit;   /* Force inheritance even for non-inherited props */
  color: initial;   /* Reset to browser default */
  color: unset;     /* Behaves as inherit for inherited props, initial for non-inherited */
  color: revert;    /* Revert to user-agent stylesheet value */
}
```

---

### CSS Reset vs Normalize

Browsers have their own default stylesheets (user agent stylesheets) that differ subtly between browsers. This is why an unstyled `<h1>` looks slightly different in Chrome vs Firefox vs Safari.

A **CSS Reset** removes all browser default styles completely, giving you a blank slate. You then restyle everything from scratch.

A **CSS Normalize** (e.g., normalize.css) preserves useful defaults but makes them consistent across browsers.

Modern approach (2026): Use a minimal reset embedded directly in your CSS:

```css
/* Modern minimal reset — a common starting point */
*, *::before, *::after {
  box-sizing: border-box;   /* Make box model intuitive (see below) */
  margin: 0;
  padding: 0;
}

html {
  font-size: 16px;
  scroll-behavior: smooth;
}

body {
  min-height: 100dvh;           /* Use dynamic viewport height */
  font-family: system-ui, sans-serif;
  line-height: 1.5;
  -webkit-font-smoothing: antialiased; /* Better font rendering on Mac */
}

img, video, svg {
  display: block;         /* Remove inline bottom gap */
  max-width: 100%;        /* Never overflow container */
}

input, button, textarea, select {
  font: inherit;           /* Inputs don't inherit font by default — fix that */
}

p, h1, h2, h3, h4, h5, h6 {
  overflow-wrap: break-word;  /* Prevent long words from overflowing */
}
```

---

### The Box Model

Every element in CSS is a rectangular box. The box model defines how the size of that box is calculated:

```
┌─────────────────────────────────────┐
│              MARGIN                 │  <- Outside the element, transparent
│  ┌───────────────────────────────┐  │
│  │            BORDER             │  │  <- The visible edge
│  │  ┌─────────────────────────┐  │  │
│  │  │         PADDING         │  │  │  <- Space inside border, takes background color
│  │  │  ┌───────────────────┐  │  │  │
│  │  │  │     CONTENT       │  │  │  │  <- Where text/children go
│  │  │  │  (width × height) │  │  │  │
│  │  │  └───────────────────┘  │  │  │
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

**The critical `box-sizing` property:**

```css
/* Default behavior (box-sizing: content-box) — confusing and counterintuitive */
.box {
  width: 200px;     /* This is the CONTENT width */
  padding: 20px;    /* Adds 40px (20 each side) to total width */
  border: 2px solid; /* Adds 4px (2 each side) to total width */
  /* Total rendered width: 200 + 40 + 4 = 244px !! */
}

/* box-sizing: border-box — intuitive and strongly recommended */
.box {
  box-sizing: border-box;
  width: 200px;     /* This is the TOTAL width (including padding and border) */
  padding: 20px;    /* Included within the 200px */
  border: 2px solid; /* Included within the 200px */
  /* Total rendered width: 200px exactly */
}

/* Apply to everything — use in your reset */
*, *::before, *::after {
  box-sizing: border-box;
}
```

This is why `box-sizing: border-box` is in virtually every CSS reset. With `content-box` (the default), if you want a two-column layout with each column at 50%, you need to mentally calculate padding and border offsets. With `border-box`, you simply say `width: 50%` and it works.

---

### Margin, Padding, Border

```css
.element {
  /* Shorthand: top right bottom left (clockwise) */
  margin: 10px 20px 10px 20px;
  
  /* Shorthand: vertical horizontal */
  margin: 10px 20px;
  
  /* Shorthand: all four sides equal */
  margin: 10px;
  
  /* Longhand */
  margin-top: 10px;
  margin-right: 20px;
  margin-bottom: 10px;
  margin-left: 20px;
  
  /* auto for centering block elements horizontally */
  margin: 0 auto;
  
  /* Padding follows the same shorthand rules */
  padding: 1rem 1.5rem;
  
  /* Border */
  border: 2px solid #ddd;            /* Shorthand: width style color */
  border-bottom: 1px dashed orange;  /* One side only */
  border-radius: 8px;                /* Rounds corners */
  border-radius: 50%;                /* Makes element circular if width === height */
}
```

**Margin Collapse** — a common source of confusion: when two vertical margins meet (e.g., the bottom margin of one paragraph and the top margin of the next), they **collapse** into a single margin equal to the larger of the two, rather than adding together. This does NOT happen with horizontal margins, padding, or margins between flex/grid items.

```css
p { margin-bottom: 20px; }
h2 { margin-top: 30px; }

/* The gap between a <p> followed by <h2> is 30px, NOT 50px */
/* The larger margin (30px) "wins" — the 20px is absorbed into it */
```

---

## 3.2 CSS Selectors & Combinators

Selectors are patterns that target HTML elements. The more precisely you can target elements, the better you can write clean CSS without side effects.

---

### Basic Selectors

```css
/* Type selector: targets all elements of that tag */
h1 { color: navy; }
p { line-height: 1.6; }

/* Class selector: targets elements with that class (.) */
.button { padding: 0.5rem 1rem; }
.button.primary { background-color: blue; }  /* Both classes required */

/* ID selector: targets ONE element with that id (#) */
/* Use sparingly! High specificity makes overriding difficult */
#main-header { position: sticky; top: 0; }

/* Universal selector: targets everything */
* { box-sizing: border-box; }

/* Attribute selector */
[type="text"] { border: 1px solid #ccc; }
[href^="https"] { /* starts with https */ }
[href$=".pdf"] { /* ends with .pdf — show download icon */ }
[href*="angular"] { /* contains "angular" anywhere */ }
[class~="card"] { /* has "card" as one of space-separated values */ }
[lang|="en"] { /* equals "en" or starts with "en-" */ }
```

---

### Combinators: Targeting Elements by Relationship

```css
/* Descendant combinator (space): any level of nesting */
nav a { text-decoration: none; }  /* All <a> inside <nav>, any depth */

/* Child combinator (>): direct children only */
ul > li { list-style: disc; }  /* Only <li> that are direct children of <ul> */

/* Adjacent sibling combinator (+): immediately following sibling */
h2 + p { 
  font-size: 1.1rem; 
  color: #555;
}  /* First <p> directly after an <h2> */

/* General sibling combinator (~): all following siblings */
h2 ~ p { 
  margin-top: 0.5rem; 
}  /* All <p> elements that follow an <h2> at the same level */
```

---

### Pseudo-Classes

Pseudo-classes target elements based on their **state** or **position** in the document:

```css
/* User interaction states */
a:hover { color: blue; text-decoration: underline; }
button:focus { outline: 2px solid #4a9eff; outline-offset: 2px; }
button:active { transform: scale(0.98); }  /* "pressed" feel */
input:focus-visible { outline: 2px solid blue; }  /* Only show outline for keyboard focus */

/* Form states */
input:valid { border-color: green; }
input:invalid { border-color: red; }
input:required { border-left: 3px solid orange; }
input:disabled { opacity: 0.5; cursor: not-allowed; }
input:checked + label { font-weight: bold; }  /* Checked checkbox's adjacent label */
input:placeholder-shown { border-style: dashed; }  /* Input with empty value */

/* Structural pseudo-classes */
li:first-child { font-weight: bold; }
li:last-child { border-bottom: none; }
li:nth-child(2) { background: #f0f0f0; }      /* Exactly 2nd child */
li:nth-child(odd) { background: #f9f9f9; }    /* 1st, 3rd, 5th... */
li:nth-child(even) { background: white; }     /* 2nd, 4th, 6th... */
li:nth-child(3n) { color: blue; }             /* Every 3rd: 3, 6, 9... */
li:nth-child(3n+1) { color: red; }            /* 1, 4, 7, 10... */
li:only-child { border: none; }               /* Only child of its parent */
p:not(.intro) { font-size: 0.9rem; }          /* All <p> without class "intro" */

/* :is() and :where() — grouping selectors */
/* :is() has the specificity of its most specific argument */
:is(h1, h2, h3) { margin-top: 1.5rem; }  /* Equivalent to h1, h2, h3 rule */

/* :where() has ZERO specificity — very useful for resets and base styles */
:where(h1, h2, h3, h4) { line-height: 1.2; }

/* :has() — the "parent selector" — one of CSS's most powerful new features */
/* Targets the element IF it contains the specified child/descendant */
.card:has(img) { padding-top: 0; }           /* Cards that contain an image */
form:has(:invalid) { border: 1px solid red; } /* Form that contains invalid inputs */
li:has(> ul) { list-style-type: none; }      /* List items that have a nested list */
.parent:has(.child:hover) { background: #eee; } /* Parent when child is hovered */
```

---

### Pseudo-Elements

Pseudo-elements let you style a specific **part** of an element, or insert virtual elements without adding HTML:

```css
/* ::before and ::after — insert content before/after element's content */
/* They are inline by default; must have a content property to appear */
.required-field::after {
  content: " *";
  color: red;
}

.external-link::after {
  content: " ↗";  /* Visual indicator for external links */
}

/* Decorative uses (no semantic content) */
.section-divider::before {
  content: "";  /* Empty string — still required */
  display: block;
  width: 60px;
  height: 3px;
  background: linear-gradient(to right, #667eea, #764ba2);
  margin-bottom: 1rem;
}

/* ::first-line and ::first-letter */
article p:first-child::first-letter {
  font-size: 3em;
  float: left;
  line-height: 0.8;
  margin-right: 0.1em;
}

/* ::placeholder */
input::placeholder {
  color: #aaa;
  font-style: italic;
}

/* ::selection — text highlighted by user */
::selection {
  background-color: #264de4;
  color: white;
}

/* ::marker — the bullet/number in list items */
li::marker {
  color: blue;
  font-size: 1.2em;
}
```

---

### Specificity — The Scoring System

When multiple selectors target the same element and property, specificity is the tiebreaker. Think of it as a three-column score `(A, B, C)`:

- **A** = Number of ID selectors (`#id` = 1-0-0)
- **B** = Number of class selectors, attribute selectors, and pseudo-classes (`.class`, `[attr]`, `:hover` = 0-1-0 each)
- **C** = Number of type selectors and pseudo-elements (`div`, `::before` = 0-0-1 each)
- Universal selector (`*`) and combinators (`>`, `+`, `~`) contribute 0.
- Inline styles = `1-0-0-0` (a fourth, higher column)
- `!important` overrides everything (separate mechanism)

```css
/* Specificity examples: */
p                          /* 0-0-1 */
.card                      /* 0-1-0 */
p.card                     /* 0-1-1 */
#header                    /* 1-0-0 */
#header .nav a             /* 1-1-1 */
#header .nav a:hover       /* 1-2-1 */

/* Compare scores left to right — first differing column wins */
/* 1-0-0 beats 0-9-9 (because 1 in A column beats 0) */

/* :is() takes specificity of its most specific argument */
:is(#id, .class) p         /* 1-0-1 (because #id is in :is()) */

/* :where() always has 0 specificity */
:where(#id, .class) p      /* 0-0-1 */
```

---

## 3.3 CSS Layout — Classic

### Display Property

`display` is the single most important CSS property for layout. It determines how an element participates in the flow of the document.

```css
/* Block: full width, starts on new line */
div { display: block; }

/* Inline: flows with text, width determined by content */
span { display: inline; }

/* Inline-block: flows with text BUT accepts width/height/vertical padding */
/* Classic use: navigation links with padding, inline buttons */
.badge { display: inline-block; padding: 2px 8px; border-radius: 12px; }

/* None: removes element completely from layout AND accessibility tree */
/* Use aria-hidden="true" on HTML side if you want visual-only hiding */
.hidden { display: none; }

/* Flex and Grid are covered in their own sections */
.flex-container { display: flex; }
.grid-container { display: grid; }
```

---

### Position

```css
/* static — the default. Element follows normal document flow */
.normal { position: static; }

/* relative — stays in flow but can be nudged. Also creates a
   "positioned ancestor" for absolute children */
.nudged {
  position: relative;
  top: 10px;    /* moves down 10px but original space is preserved */
  left: 5px;
}

/* absolute — removed from flow. Positioned relative to the nearest 
   ancestor with position: relative|absolute|fixed|sticky */
.tooltip {
  position: absolute;
  top: 100%;   /* Below the parent */
  left: 0;
  z-index: 10;
}

/* Pattern: parent relative + child absolute for overlays, badges, tooltips */
.card {
  position: relative;  /* Establishes positioning context */
}
.card__badge {
  position: absolute;
  top: -8px;
  right: -8px;  /* Positioned at top-right corner of card */
}

/* fixed — positioned relative to the viewport. Stays in place on scroll */
.sticky-header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;    /* OR width: 100% */
  z-index: 100;
}

/* sticky — hybrid of relative and fixed. Acts relative until threshold,
   then sticks to specified position within its scrolling container */
.table-header {
  position: sticky;
  top: 60px;   /* Sticks 60px from viewport top when scrolled to it */
  z-index: 1;
}
```

---

### Z-Index and Stacking Contexts

`z-index` controls which element appears "on top" when elements overlap. But here's the crucial subtlety: `z-index` only works within the same **stacking context**.

```css
/* A new stacking context is created by: 
   - position (relative/absolute/fixed/sticky) + any z-index other than auto
   - opacity less than 1
   - transform, filter, will-change, isolation: isolate
   - display: flex/grid (creates stacking context for children) */

.modal-backdrop {
  position: fixed;
  inset: 0;        /* shorthand for top:0 right:0 bottom:0 left:0 */
  background: rgba(0,0,0,0.5);
  z-index: 100;
}

.modal {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 101;    /* Higher than backdrop to appear above it */
}

/* isolation: isolate — create a stacking context WITHOUT changing z-index behavior */
/* Useful in Angular components to contain z-index values */
.component-container {
  isolation: isolate;  /* Children's z-index values won't interfere with siblings */
}
```

---

### Overflow

```css
.box {
  overflow: visible;  /* Default: content can extend beyond the box */
  overflow: hidden;   /* Clip content, hide scrollbars */
  overflow: scroll;   /* Always show scrollbars */
  overflow: auto;     /* Show scrollbars only when needed (recommended) */
  overflow: clip;     /* Like hidden but also prevents scroll via JS */
  
  /* Control each axis independently */
  overflow-x: hidden;
  overflow-y: auto;
}

/* overflow: hidden triggers a Block Formatting Context (BFC) —
   causes the element to contain its floated children (clearfix use case) */
.container { overflow: hidden; }
```

---

## 3.4 CSS Flexbox

Flexbox is a one-dimensional layout system — it arranges items along either a row or a column. It's the best tool for: navigation bars, toolbars, card rows, centering content, aligning items with different sizes, and distributing space.

### The Core Mental Model

With Flexbox, you have a **container** and **items**. Properties on the container control the overall layout. Properties on items control individual item behavior. There are two axes: the **main axis** (the direction of layout) and the **cross axis** (perpendicular).

```css
/* ========= CONTAINER PROPERTIES ========= */

.flex-container {
  display: flex;          /* OR inline-flex for inline-level container */
  
  /* flex-direction: defines the MAIN AXIS */
  flex-direction: row;            /* → left to right (default) */
  flex-direction: row-reverse;    /* ← right to left */
  flex-direction: column;         /* ↓ top to bottom */
  flex-direction: column-reverse; /* ↑ bottom to top */
  
  /* flex-wrap: should items wrap to new lines? */
  flex-wrap: nowrap;   /* Default: all items on one line (may overflow) */
  flex-wrap: wrap;     /* Items wrap to next line if needed */
  
  /* flex-flow: shorthand for flex-direction + flex-wrap */
  flex-flow: row wrap;
  
  /* justify-content: alignment along the MAIN AXIS */
  justify-content: flex-start;    /* Items at start (default) */
  justify-content: flex-end;      /* Items at end */
  justify-content: center;        /* Items centered */
  justify-content: space-between; /* Equal space BETWEEN items, none at edges */
  justify-content: space-around;  /* Equal space around each item */
  justify-content: space-evenly;  /* Equal space between items AND edges */
  
  /* align-items: alignment along the CROSS AXIS */
  align-items: stretch;    /* Items stretch to fill container height (default) */
  align-items: flex-start; /* Items align to start of cross axis */
  align-items: flex-end;   /* Items align to end of cross axis */
  align-items: center;     /* Items centered on cross axis */
  align-items: baseline;   /* Items aligned by their text baselines */
  
  /* align-content: alignment of MULTIPLE LINES (only when flex-wrap is wrap) */
  align-content: flex-start | flex-end | center | space-between | 
                 space-around | space-evenly | stretch;
  
  /* gap: space between items (modern, preferred over margin hacks) */
  gap: 1rem;           /* Equal gap in all directions */
  gap: 1rem 2rem;      /* row-gap column-gap */
  row-gap: 1rem;
  column-gap: 2rem;
}
```

```css
/* ========= ITEM PROPERTIES ========= */

.flex-item {
  /* flex-grow: how much the item grows relative to siblings 
     to fill extra space. 0 = no growth (default) */
  flex-grow: 0;
  flex-grow: 1;   /* Item takes up all available extra space */
  
  /* If you have 3 items with flex-grow: 1, 2, 1 — the second
     item takes twice as much extra space as each of the others */
  
  /* flex-shrink: how much the item shrinks relative to siblings 
     when space is tight. 1 = can shrink (default) */
  flex-shrink: 1;   /* Can shrink */
  flex-shrink: 0;   /* Will NOT shrink (maintain its size) */
  
  /* flex-basis: the initial size BEFORE growing/shrinking */
  flex-basis: auto;    /* Use the item's content size (default) */
  flex-basis: 200px;   /* Start at 200px, then grow/shrink from there */
  flex-basis: 33.333%; /* One-third of container */
  flex-basis: 0;       /* Start from nothing, grow purely from flex-grow ratio */
  
  /* flex: shorthand for flex-grow flex-shrink flex-basis */
  flex: 1;          /* Means 1 1 0 — most common "fill available space" pattern */
  flex: auto;       /* Means 1 1 auto */
  flex: none;       /* Means 0 0 auto — inflexible item */
  flex: 0 0 200px;  /* Fixed 200px item that won't grow or shrink */
  
  /* align-self: override the container's align-items for this item */
  align-self: auto | flex-start | flex-end | center | baseline | stretch;
  
  /* order: change visual order (default: 0, lower = earlier) */
  order: -1;  /* Move this item to the beginning */
  order: 1;   /* Move this item toward the end */
}
```

---

### Flexbox Practical Patterns

```css
/* ---- Perfect centering (most searched CSS question ever) ---- */
.center-everything {
  display: flex;
  justify-content: center;  /* Horizontal center */
  align-items: center;      /* Vertical center */
  min-height: 100vh;        /* Full viewport height */
}

/* ---- Navigation bar with logo left, links right ---- */
.navbar {
  display: flex;
  align-items: center;
  padding: 0 1.5rem;
}
.navbar__logo { margin-right: auto; }  /* Push everything else to the right */
.navbar__links { display: flex; gap: 1.5rem; }

/* ---- Card with footer pinned to bottom ---- */
.card {
  display: flex;
  flex-direction: column;
  height: 100%;             /* Card fills its grid cell */
}
.card__body { flex: 1; }    /* Body expands to fill available space */
.card__footer { /* Naturally at bottom */ }

/* ---- Responsive item row that wraps ---- */
.item-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
}
.item {
  flex: 1 1 250px;  /* Grow, shrink, with a base of 250px */
  /* Items fill the row until they'd be smaller than 250px, then wrap */
}
```

---

## 3.5 CSS Grid

Grid is a two-dimensional layout system — it controls both rows and columns simultaneously. It's the best tool for: page layouts, complex UI regions, photo galleries, dashboard panels, and any layout where you need precise control over both axes at once.

---

### Grid Fundamentals

```css
/* ========= CONTAINER PROPERTIES ========= */

.grid-container {
  display: grid;
  
  /* grid-template-columns: define column sizes */
  grid-template-columns: 200px 200px 200px;   /* Three 200px columns */
  grid-template-columns: 1fr 2fr 1fr;          /* Fractions of available space */
  grid-template-columns: repeat(3, 1fr);       /* Three equal columns */
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));  /* Responsive! */
  
  /* grid-template-rows: define row sizes */
  grid-template-rows: auto auto auto;  /* Rows sized to content (default) */
  grid-template-rows: 80px 1fr 60px;   /* Header, flexible content, footer */
  
  /* gap between cells */
  gap: 1rem;
  column-gap: 2rem;
  row-gap: 0.5rem;
  
  /* Alignment of content within cells */
  justify-items: start | end | center | stretch;  /* Horizontal alignment in cell */
  align-items: start | end | center | stretch;    /* Vertical alignment in cell */
  
  /* Alignment of the entire grid within the container */
  justify-content: start | end | center | space-between | space-around | space-evenly;
  align-content: start | end | center | space-between | space-around | space-evenly;
}
```

---

### The `fr` Unit and `repeat()`

The `fr` (fraction) unit is unique to CSS Grid. It represents a fraction of the **available free space** after fixed-size columns are placed.

```css
/* 3-column layout where the middle takes twice as much space */
grid-template-columns: 1fr 2fr 1fr;
/* Available space: if container is 700px, columns are: 175px 350px 175px */

/* repeat() avoids repetition */
grid-template-columns: repeat(4, 1fr);    /* 4 equal columns */
grid-template-columns: repeat(3, 200px 1fr); /* Alternating 200px and flexible */

/* minmax() — sets a minimum and maximum size */
grid-template-columns: repeat(3, minmax(100px, 1fr));
/* Each column is at least 100px but can grow to fill available space */
```

---

### The Most Important Grid Pattern: Responsive Auto-Fill

This single rule creates a responsive grid that automatically adjusts the number of columns based on available space — no media queries needed:

```css
.auto-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 1.5rem;
}
/* 
  - auto-fill: create as many columns as can fit
  - minmax(250px, 1fr): each column is at least 250px, max 1fr
  - Result: on mobile (~375px) = 1 column; tablet (~768px) = 3 columns; 
    desktop (~1200px) = 4 columns. All automatic, no @media needed!
    
  Difference: auto-fill vs auto-fit:
  - auto-fill: creates empty column tracks even if there aren't enough items
  - auto-fit: collapses empty tracks (items stretch to fill) — often preferred
*/
.auto-grid-fit {
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
}
```

---

### Named Template Areas — Most Readable Grid Pattern

```css
.page-layout {
  display: grid;
  grid-template-areas:
    "header  header  header"
    "sidebar content content"
    "footer  footer  footer";
  grid-template-columns: 250px 1fr 1fr;
  grid-template-rows: 80px 1fr 60px;
  min-height: 100vh;
}

.page-header  { grid-area: header; }
.page-sidebar { grid-area: sidebar; }
.page-content { grid-area: content; }
.page-footer  { grid-area: footer; }

/* Responsive: change to single-column on mobile */
@media (max-width: 768px) {
  .page-layout {
    grid-template-areas:
      "header"
      "content"
      "sidebar"
      "footer";
    grid-template-columns: 1fr;
    grid-template-rows: auto;
  }
}
```

---

### Placing Items: Spanning and Explicit Placement

```css
/* span: make an item cover multiple rows or columns */
.featured-item {
  grid-column: span 2;    /* Occupies 2 column tracks */
  grid-row: span 3;       /* Occupies 3 row tracks */
}

/* Explicit placement with start/end line numbers */
.hero-section {
  grid-column: 1 / 4;     /* From line 1 to line 4 (spans 3 columns) */
  grid-row: 1 / 3;
}

/* Shorthand: grid-area: row-start / col-start / row-end / col-end */
.sidebar {
  grid-area: 1 / 1 / 4 / 2;  /* From row line 1 to 4, col line 1 to 2 */
}

/* Negative line numbers count from the end */
.full-width {
  grid-column: 1 / -1;   /* From first to last line = full width */
}
```

---

### Subgrid (Modern CSS)

Subgrid allows a nested grid to participate in the parent grid's track sizing — solving the long-standing "card alignment" problem:

```css
/* Without subgrid: each card creates its own independent grid,
   so elements inside cards don't align across cards */

/* With subgrid: card's rows inherit the parent grid's row tracks */
.cards-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: auto 1fr auto;  /* title, body, footer */
  gap: 1rem;
}

.card {
  display: grid;
  grid-row: span 3;
  grid-template-rows: subgrid;  /* Inherit parent's row definitions */
}

/* Now .card__title, .card__body, .card__footer all align 
   perfectly across all cards, even with different text lengths */
.card__title { /* row 1 */ }
.card__body   { /* row 2, stretches to equal height across all cards */ }
.card__footer { /* row 3, always at the bottom */ }
```

---

## 3.6 CSS Responsive Design

Responsive design means your layout adapts to different screen sizes, orientations, and device capabilities without requiring separate websites.

---

### Mobile-First Approach

Write CSS for the smallest screen first, then add complexity with `min-width` media queries as the viewport grows. This approach produces cleaner code because it ensures the baseline works on all devices.

```css
/* BASE STYLES — mobile first (no media query needed) */
.container {
  padding: 1rem;
}

.card-grid {
  display: grid;
  grid-template-columns: 1fr;   /* Single column on mobile */
  gap: 1rem;
}

/* TABLET (768px and up) */
@media (min-width: 48em) {  /* 48em = 768px */
  .container {
    padding: 1.5rem;
  }
  
  .card-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

/* DESKTOP (1024px and up) */
@media (min-width: 64em) {  /* 64em = 1024px */
  .container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 2rem;
  }
  
  .card-grid {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

> **Why `em` for breakpoints?** Using `em` for media queries means breakpoints respond to the user's browser font size setting. If a user has increased their browser base font size to 20px for readability, your breakpoints scale with them. Using `px` ignores the user's font size preference.

---

### Media Query Features

```css
/* Width-based (most common) */
@media (min-width: 768px) { }
@media (max-width: 767px) { }   /* max-width overlaps with min — use carefully */
@media (min-width: 768px) and (max-width: 1024px) { }

/* Orientation */
@media (orientation: portrait) { }
@media (orientation: landscape) { }

/* User preference queries — respect system settings */
@media (prefers-color-scheme: dark) {
  body { background: #1a1a1a; color: #e0e0e0; }
}

@media (prefers-reduced-motion: reduce) {
  /* Users who have motion sensitivity enabled in their OS */
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}

@media (prefers-contrast: high) { }
@media (hover: none) { } /* Touch devices (no hover capability) */
@media (pointer: coarse) { } /* Touch devices (imprecise pointer) */
@media print { } /* Print stylesheet */
```

---

### Container Queries — The Future is Now

Container queries let you style an element based on the size of its **parent container** rather than the viewport. This enables truly reusable components that adapt to their context:

```css
/* Step 1: Make the parent a "container" */
.card-wrapper {
  container-type: inline-size;  /* Watch the inline (width) dimension */
  container-name: card;          /* Optional: name for targeting */
}

/* Step 2: Style the card based on its CONTAINER's width */
@container card (min-width: 400px) {
  .card {
    display: grid;
    grid-template-columns: 1fr 2fr;  /* Two-column layout when space allows */
  }
  
  .card__image {
    border-radius: 8px 0 0 8px;
  }
}

@container (min-width: 600px) {
  .card__title {
    font-size: 1.5rem;
  }
}
```

Container queries are transformative because a `.card` component in a narrow sidebar automatically switches to one-column layout, while the same component in a wide main area uses two-column — all without any media queries or JavaScript resizing logic.

---

### Viewport Units

```css
/* Classic viewport units */
.hero { height: 100vh; }    /* 100% of viewport height */
.section { width: 50vw; }   /* 50% of viewport width */
.icon { font-size: 5vmin; } /* 5% of smaller viewport dimension */

/* Problem with 100vh on mobile: includes the browser UI chrome,
   causing content to be hidden behind address bars */

/* Modern dynamic viewport units (solve the mobile browser UI problem) */
.hero {
  height: 100dvh;  /* Dynamic: adjusts when browser UI shows/hides */
  height: 100svh;  /* Small: always smallest viewport (UI visible) */  
  height: 100lvh;  /* Large: always largest viewport (UI hidden) */
}
/* Best practice: use 100dvh for full-height layouts on mobile */
```

---

### Fluid Typography with `clamp()`

`clamp(min, preferred, max)` returns the preferred value if it's between min and max, otherwise it clamps to the boundary. This creates fluid, proportional typography without any media queries:

```css
/* Font that scales with viewport width, between 1rem and 3rem */
h1 {
  font-size: clamp(1.5rem, 4vw + 1rem, 3rem);
  /* On mobile (320px): likely ~1.5rem (min)
     On desktop (1440px): likely ~3rem (max)
     Scales linearly between those breakpoints */
}

/* Fluid spacing */
.section {
  padding-block: clamp(2rem, 5vw, 5rem);
}

/* min() and max() functions */
.container {
  width: min(100%, 1200px);  /* max-width: 1200px equivalent, more explicit */
}

img {
  width: min(100%, 600px);  /* Never wider than container, max 600px */
}
```

---

## 3.7 CSS Typography

Typography makes up the majority of most web pages. Understanding it deeply separates professional designers from amateurs.

---

### Font Properties

```css
body {
  /* Font stack: try each in order, fall back to the next */
  font-family: 'Inter', 'Helvetica Neue', Arial, sans-serif;
  
  /* System font stack: uses the OS's native UI font */
  font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', 
               Roboto, sans-serif;
  
  font-size: 16px;   /* Base size. Use px here, rem elsewhere */
  font-weight: 400;  /* 100-900, or: thin, light, regular, medium, semibold, bold, extrabold, black */
  font-style: normal | italic | oblique;
  line-height: 1.6;  /* Unitless values are relative to font-size. ALWAYS use unitless! */
  /* line-height: 1.6 means 1.6 × font-size. For body text, 1.5-1.7 is ideal for readability. */
}
```

> **Why unitless `line-height`?** If you write `line-height: 24px` and a child element has a larger font, the line-height doesn't scale with it, causing text to overlap. A unitless multiplier like `1.6` always scales proportionally with whatever font-size is inherited.

---

### Web Fonts

```css
/* @font-face: load a custom font file */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/Inter-variable.woff2') format('woff2');
  font-weight: 100 900;         /* Variable font: specify range */
  font-style: normal;
  font-display: swap;           /* CRITICAL for performance (see below) */
}

/* font-display values:
   auto     — browser decides (usually like block)
   block    — invisible text for up to 3s, then fallback, then swap
   swap     — show fallback immediately, swap to web font when loaded (best for body text)
   fallback — invisible for 100ms, then fallback, swap only in first 3s 
   optional — use font only if already cached (best for non-essential decorative fonts) */
```

---

### Text Properties

```css
.text-styles {
  /* Alignment */
  text-align: left | right | center | justify | start | end;
  
  /* Decoration */
  text-decoration: none | underline | overline | line-through;
  text-decoration: underline dotted blue;  /* color and style control */
  text-underline-offset: 3px;             /* Space between text and underline */
  
  /* Transform */
  text-transform: none | uppercase | lowercase | capitalize;
  
  /* Spacing */
  letter-spacing: 0.05em;  /* Tracking — use em to scale with font size */
  word-spacing: 0.1em;
  
  /* Wrapping and overflow */
  white-space: normal | nowrap | pre | pre-wrap | pre-line;
  overflow: hidden;
  text-overflow: ellipsis;  /* Shows "..." when text overflows (requires overflow:hidden) */
  
  /* Multi-line clamp (no JS needed!) */
  display: -webkit-box;
  -webkit-line-clamp: 3;        /* Show max 3 lines */
  -webkit-box-orient: vertical;
  overflow: hidden;
  
  /* New: text-wrap */
  text-wrap: balance;   /* Evenly balance line lengths in headings */
  text-wrap: pretty;    /* Avoid orphaned words at end of paragraphs */
}
```

---

## 3.8 CSS Colors & Backgrounds

### Modern Color Formats

```css
/* Hex */
color: #ff6600;
color: #f60;    /* Shorthand: #rgb → #rrggbb */
color: #ff660080; /* With alpha (#rrggbbaa) */

/* RGB / RGBA */
color: rgb(255, 102, 0);
color: rgba(255, 102, 0, 0.5);  /* 50% transparent */
color: rgb(255 102 0 / 50%);    /* Modern space-separated syntax */

/* HSL — most intuitive for designers */
/* hue (0-360 degrees) saturation (%) lightness (%) */
color: hsl(24, 100%, 50%);              /* Bright orange */
color: hsl(24 100% 50% / 0.5);         /* 50% transparent */

/* OKLCH — the future of color in CSS (perceptually uniform) */
/* lightness (0-1) chroma (0-0.4) hue (0-360) */
color: oklch(0.7 0.15 24);
/* oklch is perceptually uniform: you can change lightness predictably
   without shifting the perceived hue. Makes theming and color scales 
   much easier to work with than HSL. */
```

---

### CSS Custom Properties (Variables)

CSS custom properties are the foundation of modern design systems. They cascade and inherit like regular properties, and can be changed with JavaScript at runtime:

```css
/* Define variables on :root to make them globally available */
:root {
  /* Color tokens */
  --color-primary: oklch(0.55 0.22 264);
  --color-primary-light: oklch(0.75 0.15 264);
  --color-secondary: oklch(0.65 0.18 145);
  --color-text: #1a1a2e;
  --color-text-muted: #666;
  --color-surface: #fff;
  --color-border: #e2e8f0;
  
  /* Spacing scale */
  --space-xs: 0.25rem;
  --space-sm: 0.5rem;
  --space-md: 1rem;
  --space-lg: 1.5rem;
  --space-xl: 2rem;
  --space-2xl: 4rem;
  
  /* Typography */
  --font-sans: 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', monospace;
  --text-sm: 0.875rem;
  --text-base: 1rem;
  --text-lg: 1.125rem;
  
  /* Radius */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 16px;
  --radius-full: 9999px;
  
  /* Shadows */
  --shadow-sm: 0 1px 3px rgba(0,0,0,0.1);
  --shadow-md: 0 4px 12px rgba(0,0,0,0.1);
}

/* Dark mode: override variables — no need to duplicate component styles! */
@media (prefers-color-scheme: dark) {
  :root {
    --color-text: #e2e8f0;
    --color-surface: #1a1a2e;
    --color-border: #2d3748;
  }
}

/* Usage */
.card {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
  padding: var(--space-lg);
  box-shadow: var(--shadow-md);
}

.button {
  background: var(--color-primary);
  color: white;
  padding: var(--space-sm) var(--space-lg);
}

/* Override locally: variables cascade! */
.dark-card {
  --color-surface: #1a1a2e;  /* Overrides for this element and all descendants */
  --color-text: #e0e0e0;
  background: var(--color-surface);
  color: var(--color-text);
}

/* JavaScript can manipulate CSS variables at runtime */
/* document.documentElement.style.setProperty('--color-primary', '#e44') */
```

---

### Gradients

```css
.gradient-examples {
  /* Linear gradient */
  background: linear-gradient(to right, #667eea, #764ba2);
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  background: linear-gradient(to bottom, transparent, rgba(0,0,0,0.8));

  /* Radial gradient */
  background: radial-gradient(circle, #667eea, #764ba2);
  background: radial-gradient(ellipse at top right, #ff6b6b, #764ba2);

  /* Conic gradient (great for pie charts, color wheels) */
  background: conic-gradient(red 0deg, blue 180deg, green 360deg);
  background: conic-gradient(from 0deg, red 25%, blue 25% 50%, 
                             green 50% 75%, yellow 75%);
}

/* Multiple backgrounds: layered from front to back */
.hero {
  background:
    linear-gradient(to bottom, transparent 50%, rgba(0,0,0,0.7)),
    url('hero.jpg') center/cover no-repeat;
}
```

---

### Filters and Blend Modes

```css
/* filter: apply visual effects to elements */
.image-effects {
  filter: blur(4px);
  filter: brightness(1.2);       /* > 1 = brighter */
  filter: contrast(1.5);         /* > 1 = more contrast */
  filter: grayscale(100%);
  filter: sepia(50%);
  filter: saturate(2);
  filter: hue-rotate(90deg);
  filter: invert(100%);
  filter: drop-shadow(0 4px 8px rgba(0,0,0,0.3)); /* Respects transparency */
  
  /* Multiple filters: combined */
  filter: brightness(1.1) contrast(1.2) saturate(1.3);
}

/* backdrop-filter: apply filter to the area BEHIND the element */
/* Used for frosted glass effects */
.glass-panel {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(12px) saturate(1.5);
  border: 1px solid rgba(255, 255, 255, 0.2);
}
```

---

## 3.9 CSS Animations & Transitions

### Transitions

Transitions animate a property change smoothly from one value to another, triggered by state changes (hover, focus, class change):

```css
/* Shorthand: property duration timing-function delay */
.button {
  background-color: blue;
  transform: scale(1);
  
  /* Transition specific properties */
  transition: background-color 0.3s ease, transform 0.15s ease-out;
  
  /* Transition all properties (avoid in production — less performant) */
  transition: all 0.3s ease;
}

.button:hover {
  background-color: darkblue;
  transform: scale(1.05);
  /* The transition plays automatically between blue/scale(1) and these values */
}

/* Timing functions */
.examples {
  transition-timing-function: ease;          /* Slow start, fast middle, slow end (default) */
  transition-timing-function: linear;        /* Constant speed */
  transition-timing-function: ease-in;       /* Slow start */
  transition-timing-function: ease-out;      /* Slow end (natural deceleration) */
  transition-timing-function: ease-in-out;   /* Slow start and end */
  transition-timing-function: cubic-bezier(0.175, 0.885, 0.32, 1.275); /* Custom bounce */
  transition-timing-function: steps(4, end); /* Stepped (good for sprite animation) */
}
```

---

### Keyframe Animations

`@keyframes` animations play automatically without a trigger and give you full control over every step of the animation:

```css
/* Define the animation */
@keyframes fadeInUp {
  from {                        /* 0% */
    opacity: 0;
    transform: translateY(20px);
  }
  to {                          /* 100% */
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50%       { transform: scale(1.05); }
}

@keyframes shimmer {
  0%   { background-position: -200% center; }
  100% { background-position: 200% center; }
}

/* Apply the animation */
.card {
  animation: 
    fadeInUp            /* animation-name */
    0.5s               /* animation-duration */
    ease-out           /* animation-timing-function */
    0.1s               /* animation-delay */
    1                  /* animation-iteration-count (or infinite) */
    normal             /* animation-direction (normal, reverse, alternate, alternate-reverse) */
    forwards           /* animation-fill-mode: forwards = keep final state */
    running;           /* animation-play-state (running or paused) */
}

/* Skeleton loading animation */
.skeleton {
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}
```

---

### Performance-Safe Animation Properties

Only two CSS properties can be animated without causing expensive layout recalculations:

```css
/* SAFE: Only trigger compositing (GPU handles these, main thread stays free) */
.smooth {
  transform: translateX(100px) rotate(45deg) scale(1.5);
  opacity: 0.5;
}

/* EXPENSIVE: Trigger layout reflow (use sparingly or not at all for animations) */
/* Never animate: width, height, margin, padding, top, left, right, bottom,
   font-size, display, border-width, max-height */

/* will-change: hint to browser to prepare for animation */
/* Creates a new compositing layer */
.will-animate {
  will-change: transform, opacity;
  /* Use ONLY when you know animation will happen soon */
  /* Don't set on too many elements — wastes GPU memory */
}

/* Remove will-change after animation completes (via JS) */
```

---

### CSS Transforms

```css
.element {
  /* 2D transforms */
  transform: translateX(50px);           /* Move horizontally */
  transform: translateY(-20px);          /* Move vertically */
  transform: translate(50px, -20px);     /* Move both */
  transform: rotate(45deg);             /* Rotate */
  transform: scale(1.5);                /* Uniform scale */
  transform: scale(2, 0.5);            /* Non-uniform scale */
  transform: skewX(15deg);             /* Shear horizontally */
  
  /* 3D transforms */
  transform: rotateX(45deg);            /* Rotate around X axis */
  transform: rotateY(45deg);            /* Rotate around Y axis */
  transform: perspective(500px) rotateY(30deg);
  
  /* Chaining transforms (applied right to left!) */
  transform: translateX(100px) rotate(45deg) scale(0.8);
  
  /* Transform origin: where transformation is applied from */
  transform-origin: center center;       /* Default */
  transform-origin: top left;
  transform-origin: 50% 100%;           /* Bottom center */
}
```

---

## 3.10 CSS Modern Features (2024–2026)

### CSS Nesting (Native)

CSS nesting is now supported in all modern browsers. No Sass needed for nesting:

```css
/* Native CSS nesting (baseline 2024, all browsers) */
.card {
  background: white;
  border-radius: 8px;
  
  /* Nest rules for child elements */
  & .card__title {
    font-size: 1.25rem;
    font-weight: 700;
  }
  
  /* Pseudo-classes */
  &:hover {
    box-shadow: 0 8px 24px rgba(0,0,0,0.15);
    transform: translateY(-2px);
  }
  
  /* Media queries can be nested too */
  @media (min-width: 768px) {
    padding: 2rem;
  }
  
  /* Nest combinators */
  & + & {
    margin-top: 1rem;  /* Adjacent card siblings */
  }
}
```

---

### CSS @layer (Cascade Layers)

`@layer` solves the specificity wars problem in large codebases by giving you explicit control over the priority order of CSS blocks:

```css
/* Define the order of layers upfront (later layers have higher priority) */
@layer reset, base, theme, layout, components, utilities;

/* Styles in each layer */
@layer reset {
  * { box-sizing: border-box; margin: 0; }
}

@layer base {
  h1 { font-size: 2rem; }    /* Low priority base styles */
  p { line-height: 1.6; }
}

@layer components {
  .button { padding: 0.5rem 1rem; background: blue; }
}

@layer utilities {
  /* Utility classes always win — they're in the highest layer */
  .hidden { display: none; }
  .text-red { color: red; }
}

/* Key insight: a rule in a higher layer with LOW specificity 
   beats a rule in a lower layer with HIGH specificity.
   This means you no longer need !important or selector specificity hacks
   to override third-party library styles — just put your overrides 
   in a higher-priority layer. */

/* Importing third-party CSS into a layer (prevents it from overriding yours) */
@import url('bootstrap.css') layer(bootstrap);
```

---

### CSS Scroll-Driven Animations

Animations that respond to scroll position — no JavaScript IntersectionObserver needed:

```css
/* Scroll progress bar at the top of the page */
@keyframes grow-progress {
  from { width: 0%; }
  to   { width: 100%; }
}

.reading-progress-bar {
  position: fixed;
  top: 0;
  left: 0;
  height: 4px;
  background: linear-gradient(to right, #667eea, #764ba2);
  animation: grow-progress linear;
  animation-timeline: scroll(root);  /* Driven by root element's scroll */
}

/* Fade in elements as they enter the viewport */
@keyframes fade-in-slide {
  from { opacity: 0; transform: translateY(30px); }
  to   { opacity: 1; transform: translateY(0); }
}

.animate-on-scroll {
  animation: fade-in-slide 0.6s ease-out;
  animation-timeline: view();            /* Tied to element's position in viewport */
  animation-range: entry 0% entry 100%; /* Play during entry phase */
}
```

---

### CSS Anchor Positioning

Position a floating element (tooltip, dropdown) relative to any other element on the page — without JavaScript position calculations:

```css
/* The anchor element */
.button {
  anchor-name: --my-button;  /* Register as anchor with a custom name */
}

/* The floating element */
.tooltip {
  position: absolute;
  position-anchor: --my-button;  /* Anchor to the button */
  
  /* inset-area: position relative to anchor using directional keywords */
  inset-area: top;               /* Place above the button */
  margin-bottom: 8px;            /* Gap from the anchor */
  
  /* Or use anchor() function for precise control */
  bottom: anchor(top);           /* Tooltip's bottom = button's top */
  left: anchor(center);          /* Center-align with button */
  translate: -50% 0;
}
```

---

### View Transitions API

Animate transitions between page states with CSS — including multi-page app navigation:

```css
/* CSS for view transitions */
::view-transition-old(root) {
  /* How the leaving page animates out */
  animation: slide-out 0.3s ease-in;
}

::view-transition-new(root) {
  /* How the entering page animates in */
  animation: slide-in 0.3s ease-out;
}

@keyframes slide-out {
  from { transform: translateX(0); }
  to   { transform: translateX(-100%); }
}

@keyframes slide-in {
  from { transform: translateX(100%); }
  to   { transform: translateX(0); }
}

/* Give individual elements their own transition name */
.hero-image {
  view-transition-name: hero;  /* Unique name — this element animates independently */
}

/* In Angular: use withViewTransitions() in provideRouter() to enable automatic
   view transitions for route navigation */
```

---

### @starting-style — Entry Animations

Animate elements when they are first displayed (entering from `display: none`):

```css
.dialog {
  opacity: 1;
  transform: scale(1);
  transition: opacity 0.3s, transform 0.3s;
  
  /* Define styles for when the element FIRST appears */
  @starting-style {
    opacity: 0;
    transform: scale(0.9);
  }
}

/* When display switches from none to block, the transition plays
   from @starting-style values to the normal values */
```

---

### CSS @property — Typed Custom Properties

Allows you to define type information for CSS custom properties, enabling animations of custom properties:

```css
/* Register a typed custom property */
@property --gradient-angle {
  syntax: '<angle>';
  inherits: false;
  initial-value: 0deg;
}

/* Now you can ANIMATE this custom property (not possible with regular custom props) */
@keyframes rotate-gradient {
  to { --gradient-angle: 360deg; }
}

.animated-border {
  border: 3px solid transparent;
  background: conic-gradient(
    from var(--gradient-angle),
    #667eea,
    #764ba2,
    #667eea
  ) border-box;
  animation: rotate-gradient 3s linear infinite;
}
```

---

### light-dark() Function

```css
:root {
  color-scheme: light dark;  /* Tell browser both modes are supported */
}

.element {
  /* light-dark(light-value, dark-value) — no media query needed! */
  background-color: light-dark(#ffffff, #1a1a2e);
  color: light-dark(#1a1a2e, #e0e0e0);
  border-color: light-dark(#e2e8f0, #2d3748);
}
```

---

## 3.11 CSS Architecture & Methodologies

### BEM (Block Element Modifier)

BEM is the most widely used CSS naming convention. It makes the relationship between HTML structure and CSS class names explicit and predictable.

```css
/* Block: the standalone component */
.card { }

/* Element: a part of the block (double underscore) */
.card__header { }
.card__title { }
.card__body { }
.card__footer { }

/* Modifier: a variant of the block or element (double hyphen) */
.card--featured { }          /* Featured card variant */
.card--dark { }              /* Dark theme variant */
.card__title--large { }      /* Large title variant */
```

```html
<article class="card card--featured">
  <div class="card__header">
    <h2 class="card__title card__title--large">Article Title</h2>
  </div>
  <div class="card__body">
    <p>Body content here...</p>
  </div>
  <footer class="card__footer">
    <button class="button button--primary">Read More</button>
  </footer>
</article>
```

**Why BEM works:** Every class name is unique and self-documenting. No specificity problems (all single class selectors). No accidental style leakage between components. You can refactor HTML without hunting for cascading style bugs.

---

### CSS Modules (Most Relevant for Angular)

CSS Modules automatically generate unique class names at build time, preventing all style collisions. Angular's `ViewEncapsulation.Emulated` (the default) achieves a similar effect by adding unique attribute selectors.

```css
/* card.module.css */
.card { background: white; }
.title { font-size: 1.25rem; }

/* Generated output (class names are hashed) */
/* .card_abc123 { background: white; } */
/* .title_abc123 { font-size: 1.25rem; } */
```

---

### Utility-First CSS (Tailwind CSS Concepts)

Utility-first CSS provides single-purpose, composable classes. Each class does exactly one thing. You build UI by composing these atomic classes in HTML.

```html
<!-- Tailwind example: each class is a single CSS declaration -->
<button class="px-4 py-2 bg-blue-600 text-white font-semibold 
               rounded-lg shadow-md hover:bg-blue-700 
               focus:outline-none focus:ring-2 focus:ring-blue-500 
               transition-colors duration-200">
  Submit
</button>
```

The advantage is zero dead CSS, no naming bikeshedding, and extremely fast prototyping. The tradeoff is verbose HTML and a steeper learning curve for the class vocabulary.

---

## 3.12 CSS Preprocessors

### Sass/SCSS

Despite native CSS nesting reducing the need for Sass, SCSS still offers powerful features for complex design systems:

```scss
// Variables (though CSS custom properties are usually better now)
$primary: oklch(0.55 0.22 264);
$radius-md: 8px;

// Mixins: reusable style blocks that can accept arguments
@mixin flex-center {
  display: flex;
  align-items: center;
  justify-content: center;
}

@mixin card($padding: 1rem, $radius: $radius-md) {
  padding: $padding;
  border-radius: $radius;
  background: white;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

// Extend: share styles between selectors
%button-base {
  display: inline-flex;
  align-items: center;
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}

.button-primary {
  @extend %button-base;
  background: $primary;
  color: white;
}

.button-secondary {
  @extend %button-base;
  background: transparent;
  border: 1px solid $primary;
  color: $primary;
}

// Partials and @use (modern Sass — replaces @import)
// _tokens.scss, _reset.scss, _typography.scss, etc.
@use 'tokens' as t;
@use 'mixins' as m;

.card {
  @include m.card(1.5rem);
  color: t.$color-text;
}

// Functions
@function to-rem($px, $base: 16) {
  @return ($px / $base) * 1rem;
}

h1 { font-size: to-rem(32); }  // → 2rem
```

---

## 3.13 CSS in Angular Context

### ViewEncapsulation

Angular's view encapsulation controls how component styles are scoped to prevent unintended style leakage:

```typescript
import { Component, ViewEncapsulation } from '@angular/core';

@Component({
  selector: 'app-card',
  template: `<div class="card"><h2 class="title">{{ title }}</h2></div>`,
  styles: [`
    .card { background: white; border-radius: 8px; }
    .title { font-size: 1.25rem; }
  `],
  
  // THREE OPTIONS:
  
  // Emulated (DEFAULT): Angular adds unique attribute selectors to styles
  // .card[_ngcontent-abc123] { ... }
  // This scopes styles to this component without Shadow DOM
  encapsulation: ViewEncapsulation.Emulated,
  
  // None: Styles are global. Use for global/base styles or
  // when you NEED to style children components
  encapsulation: ViewEncapsulation.None,
  
  // ShadowDom: Uses real Shadow DOM — true encapsulation
  // Prevents all outside styles from penetrating the component
  // Tradeoff: Angular Material's global styles won't apply
  encapsulation: ViewEncapsulation.ShadowDom,
})
export class CardComponent { }
```

---

### Host Selectors in Angular

```scss
// Styles that apply to the component's HOST element (the <app-card> tag itself)
:host {
  display: block;           // Components are inline by default — make them block!
  padding: 1rem;
}

// Style the host based on its attributes/classes
:host(.featured) {
  border: 2px solid gold;
}

:host-context(.dark-theme) {
  // Style the component when it's inside a .dark-theme ancestor
  background: #1a1a2e;
  color: white;
}

// Targeting child components (use sparingly — breaks encapsulation)
::ng-deep .mat-button {
  border-radius: 0;
}
// ⚠️ ::ng-deep is deprecated — prefer ViewEncapsulation.None for shared styles
// or use CSS custom properties to configure child component appearance
```

---

### Angular Material Theming

Angular Material uses a token-based theming system built on CSS custom properties:

```scss
// styles.scss — global theming
@use '@angular/material' as mat;

// Define your theme
$my-theme: mat.define-theme((
  color: (
    theme-type: light,
    primary: mat.$violet-palette,
    tertiary: mat.$orange-palette,
  ),
  typography: (
    brand-family: 'Inter, sans-serif',
  ),
  density: (
    scale: 0,
  )
));

// Apply the theme
html {
  @include mat.all-component-themes($my-theme);
}

// Dark mode
@media (prefers-color-scheme: dark) {
  $dark-theme: mat.define-theme((
    color: (
      theme-type: dark,
      primary: mat.$violet-palette,
    )
  ));
  
  html {
    @include mat.all-component-colors($dark-theme);
  }
}
```

---

### Best Practices for CSS in Angular

**Use CSS custom properties for theming**, not Sass variables. Custom properties can change at runtime, respond to media queries, and work across component encapsulation boundaries. Sass variables are compiled away at build time.

**Always set `:host { display: block; }`** on Angular components that should behave like block elements. Angular components are rendered as custom elements (e.g., `<app-card>`) which are inline by default.

**Prefer component-scoped styles** over global styles wherever possible. Put only truly global concerns (typography scale, CSS reset, design tokens) in `styles.scss`.

**Use `styleUrls` for anything more than a few lines**. This keeps component logic and styles separate, enables syntax highlighting and linting for CSS files, and makes large components manageable.

**Avoid deep nesting** even in SCSS. More than 3 levels deep is almost always a sign of over-specificity. Flat BEM-style selectors perform better and are easier to override.

---

## ✅ CSS Best Practices Summary

**Box model:** Always use `box-sizing: border-box` globally. It makes layout calculations intuitive.

**Specificity management:** Keep specificity low and flat. Use `:where()` for base styles, `@layer` for architecture, and avoid `!important`. Prefer class selectors over element or ID selectors in component CSS.

**Layout:** Use Grid for two-dimensional layouts (page structures, card grids). Use Flexbox for one-dimensional layouts (navigation bars, content within a card). Don't use one when the other is a better fit.

**Responsive design:** Write mobile-first. Use `em` for breakpoints. Explore container queries for component-level responsiveness. Use `clamp()` for fluid type and spacing.

**Performance:** Only animate `transform` and `opacity` for smooth 60fps animations. Use `will-change` sparingly. Use `content-visibility: auto` for long page sections.

**Maintainability:** Use CSS custom properties for design tokens. Follow BEM or a consistent naming convention. Organize styles with `@layer`. Comment non-obvious CSS decisions.

**Accessibility:** Never remove the focus outline without providing an equivalent alternative. Use `prefers-reduced-motion` to disable animations for users who need it. Ensure sufficient color contrast (4.5:1 for normal text, 3:1 for large text).

---

*End of Phase 3 — CSS Mastery*

> **Up next:** Phase 4 — JavaScript Mastery, where you'll learn the programming language that brings your HTML structures and CSS styles to life with behavior, interactivity, and data.
