# Design audit checklist

Used by Step 0 of the `staging-redesign` skill. The checklist says what to look for, not
what style to use. Every concrete value in a fix (font, color, spacing, radius, timing)
comes from the site's own tokens: the Tailwind config and global CSS in
`get_global_code`. If the site has no value that fits, ask the user.

## Fix order

1. States (hover, pressed, focus)
2. Layout and spacing
3. Generic components
4. Missing states and pages
5. Copy
6. Motion

## Generic AI patterns

- Purple/blue "AI gradient" hero.
- Every section centered and symmetrical.
- Three equal cards as the feature row. The content is fine; try a two-column zig-zag or
  an asymmetric grid.
- Border + shadow + white background on every card. Use cards only where elevation
  shows hierarchy.
- A border on one side of a card only.
- One filled + one ghost button everywhere.
- Pill "New"/"Beta" badges.
- Three-card testimonial carousel with dots.
- A 4-column footer link farm. Trim to the main paths plus legal links.
- Rocket/shield icon clichés.
- More than one accent color, or oversaturated accents.
- A random dark section in a light page, or the reverse.

## Layout and code

- `100vh` on full-screen sections. Use `min-height: 100dvh`.
- No max-width container.
- Body text wider than about 65ch.
- Headings without `text-wrap: balance`, body without `text-wrap: pretty`.
- Numbers in tables and prices without `tabular-nums`.
- CTAs not pinned to the bottom of card groups; pricing feature lists starting at
  different heights.
- Arbitrary values (`text-[19px]`, `z-[9999]`). Snap them to the site's Tailwind scale.
- Div soup. Use `nav`, `main`, `section`, `article`.
- Inline styles mixed with classes.
- Meaningful images without a real `alt`.

## States

- Hover on every button and link.
- Pressed feedback (a slight scale-down, about 0.97).
- A visible focus ring.
- A current-page indicator in the navigation builtin.
- No `#` dead links. Link them or visually disable them.
- Forms: required-field and email validation, inline specific errors, and a plain
  success message.

## Missing pages and ship items

- A branded 404 (`list_error_pages`).
- Privacy and terms links in the footer builtin.
- A skip-to-content link in the layout.
- A way back from every page.
- A favicon.
- Cookie consent where required.

Check these against the `client-handoff-checklist` skill rather than repeating its steps.

## Copy

Audit against the Copy rules in the `landing-page-from-brief` skill: vague claims, weak
CTAs, filler words, placeholder names and numbers, Title Case headings.

## Motion

Follows Emil Kowalski's guidance on UI motion; it wins where other sources disagree.

- `window` scroll listeners for reveals. Use IntersectionObserver.
- Animated `top`/`left`/`width`/`height`. Use transform and opacity.
- Long or decorative animations. Keep UI transitions under about 300ms, ease-out for
  entering elements, no animation on high-frequency interactions, and respect
  `prefers-reduced-motion`.
- Don't add blur scroll reveals, word-by-word text reveals or long easing.
