# Accessibility

> Accessibility means building interfaces that people with different abilities, devices and situations can perceive, operate and understand, and it is both a quality bar and often a legal requirement.

## Overview

People use the web with screen readers, keyboards only, voice control, magnified text, high-contrast settings, switch devices and small screens. Some have permanent disabilities. Others have a broken arm, a glaring sun or a slow connection. Accessible design serves all of them, and often improves the experience for everyone.

Accessibility is not a final polish step. It is a property of structure, and structure is cheapest to get right at the start. Retrofitting an inaccessible component library is slow and expensive. In many jurisdictions, accessibility is also a legal obligation for certain organizations, so ignoring it carries real risk. Check the rules that apply to your product.

## Core Concepts

### The standard

The Web Content Accessibility Guidelines (WCAG), published by the W3C, are the common reference. They are organized around four principles, often abbreviated POUR:

- **Perceivable.** Information can be presented in ways users can perceive (text alternatives, captions, contrast).
- **Operable.** Interface components can be used by keyboard and other inputs, with enough time and no seizure-inducing content.
- **Understandable.** Content and behavior are predictable and errors are explained.
- **Robust.** Content works with current and future assistive technologies.

Conformance has levels (A, AA, AAA). Level AA is the usual target in policies and regulations.

### Semantic HTML first

Native elements carry meaning and behavior for free. A `<button>` is focusable, activates with Enter and Space, and is announced as a button. A `<div onclick>` has none of that.

```html
<!-- Bad: no role, not focusable, no keyboard support -->
<div class="btn" onclick="save()">Save</div>

<!-- Good -->
<button type="button" onclick="save()">Save</button>
```

Use headings to give a document outline, `<nav>`, `<main>` and other landmarks for navigation, lists for lists, `<table>` with headers for tabular data, and `<label>` for form controls.

### ARIA

ARIA (Accessible Rich Internet Applications) attributes add semantics when native HTML cannot express them. The first rule of ARIA: do not use ARIA if a native element does the job. ARIA changes what assistive technology is told. It does not add behavior. A `role="button"` still needs focus handling and key events written by you. Wrong ARIA is worse than none, because it lies to the user.

Useful pieces:

- `aria-label` and `aria-labelledby` give accessible names.
- `aria-expanded`, `aria-selected`, `aria-current` expose state.
- `aria-live` regions announce dynamic updates, such as form errors or toast messages.
- `aria-hidden` hides content from assistive technology, never use it on focusable elements.

### Keyboard access and focus

Everything operable by mouse must be operable by keyboard. Requirements:

- A logical tab order that follows reading order.
- A visible focus indicator. Never remove the outline without replacing it.
- No keyboard traps. Modals should trap focus while open, but allow escape, and return focus to the trigger on close.
- Skip links to bypass repeated navigation.
- Manage focus when content changes, for example moving focus to a new view's heading after client-side navigation.

### Text alternatives and media

Informative images need meaningful `alt` text. Decorative images use empty `alt=""` so they are skipped. Icon-only buttons need an accessible name. Video needs captions, and audio needs transcripts.

### Color, contrast and layout

- Text needs sufficient contrast with its background. WCAG AA specifies minimum ratios, for example 4.5:1 for normal text. Verify with a tool.
- Do not use color as the only signal (an error shown only in red).
- Support zoom and text resizing without breaking layout or hiding content. Use relative units.
- Respect user preferences such as `prefers-reduced-motion` and `prefers-color-scheme`.

```css
@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; transition: none !important; }
}
```

### Forms and errors

Each input has a programmatically associated label. Group related controls with `<fieldset>` and `<legend>`. Errors should be specific ("Enter a date in DD/MM/YYYY format"), connected to the field (`aria-describedby`), and announced. Do not rely on placeholder text as a label.

### Dynamic applications

Single-page apps break assumptions of page loads. After a route change, announce the new page, update the document title and move focus. Custom widgets (menus, tabs, comboboxes, dialogs) have established keyboard patterns documented in the WAI-ARIA Authoring Practices. Following them matters more than inventing your own.

### Touch targets

Controls should be large enough to tap accurately, with spacing so adjacent targets are not hit by mistake.

## Trade-offs

| Choice | Gain | Cost |
|---|---|---|
| Native elements | Accessibility built in | Limited styling for some controls (selects, date inputs) |
| Custom components | Full visual control | You own keyboard, focus, ARIA and testing |
| Established component library | Faster, often well tested | Verify claims, customization limits, dependencies |
| Automated testing | Cheap, catches many issues in CI | Detects only a portion of problems |
| Manual testing with assistive tech | Finds real usability issues | Time and skill |
| Rich animation | Delight | Harm for vestibular disorders, needs a reduced-motion path |

Automated tools catch a meaningful share of issues, such as missing labels or low contrast, but cannot judge whether alt text is meaningful or a flow is understandable. Do not treat a green automated score as proof of accessibility.

## Failure Modes

- **Div soup.** Clickable divs with no role or keyboard support. Detect by tabbing through the page.
- **Removed focus outline.** Keyboard users cannot see where they are.
- **Modal focus leaks.** Focus moves behind the dialog, or never returns after closing.
- **Unlabeled controls.** Icon buttons announced only as "button". Detect with a screen reader or an accessibility tree inspection.
- **Misused ARIA.** `aria-hidden` on a focused element, redundant or contradictory roles.
- **Silent updates.** Dynamic changes never announced, so screen reader users do not know a save failed.
- **Contrast failures from design tokens.** A brand color fails ratios in a disabled or hover state.
- **Time limits.** Sessions expire without warning or ability to extend.
- **Inaccessible custom select or date picker.** Frequent in generated or hand-rolled components.
- **Regression.** A new feature breaks something fixed last quarter. Add checks to CI.

## In Practice

### A quick audit

1. Unplug the mouse. Complete the main task with the keyboard alone.
2. Zoom to 200 percent and check nothing is lost.
3. Run an automated checker, then fix what it finds.
4. Try a screen reader on the main flow. Does it make sense without the visuals?
5. Check contrast of text, icons and focus indicators.
6. Turn on reduced motion and a high-contrast mode.

### Heuristics

- Use the right element first. Add ARIA last.
- Treat the accessibility tree as a second output of your UI and inspect it.
- Include people with disabilities in research where possible. Their feedback beats any checklist.
- Define accessibility acceptance criteria in tickets, so it is part of "done".

### Developer vs. engineer

The developer ships what looks right in a browser. The engineer checks that it works by keyboard, announces correctly, and keeps working after the next refactor.

### Verifying AI-generated code

Generated markup often favors divs and spans, omits labels and alt text, adds ARIA attributes incorrectly, builds custom dropdowns without keyboard support, and ignores focus management. Run automated checks, then test by keyboard and with a screen reader.

## Further Reading

- W3C, Web Content Accessibility Guidelines (WCAG) 2.x and the "Understanding WCAG" supporting documents.
- W3C, WAI-ARIA Authoring Practices Guide.
- MDN Web Docs, "Accessibility" learning area.
- Heydon Pickering, *Inclusive Components*.
- Sara Soueidan and Léonie Watson, writings on accessible interface design.
- Inclusive Design Principles (inclusivedesignprinciples.org).
