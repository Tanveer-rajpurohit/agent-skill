---
name: gsap
description: >
  Build smooth, performant web animations with GSAP and its plugins. Trigger for:
  scroll-driven animations (ScrollTrigger), timelines and sequencing, entrance/
  reveal animations, parallax, pinned sections, scrub animations, text/SplitText
  effects, hover/micro-interactions, page transitions, SVG/path animation, and
  integrating GSAP with React/Next.js (useGSAP, cleanup, refs). Also trigger for
  "animate this", "scroll animation", "parallax", "stagger", "timeline",
  "smooth scroll", "reveal on scroll", "make it feel alive", or fixing janky/
  laggy animations. Covers React integration and 60fps performance.
---

# GSAP Skill — Smooth, Performant Web Animation

You animate like a motion designer who profiles. Default to **GSAP + ScrollTrigger**.
Animate only `transform` and `opacity` (GPU-composited) — never `width`, `height`,
`top`, `left`, `margin` (they trigger layout/paint → jank). Target a steady 60fps.

## Core mental model

```
gsap.to(target, vars)        animate TO these values from current
gsap.from(target, vars)      animate FROM these values to current (great for reveals)
gsap.fromTo(target, from, to) explicit both ends (most predictable for reveals)
gsap.set(target, vars)       instant set, no animation (initial state)

timeline()  → sequence/overlap multiple tweens with precise control
ScrollTrigger → tie any animation to scroll position
```

## React / Next.js integration — the ONLY correct pattern

Use `@gsap/react`'s `useGSAP` — it scopes selectors and **auto-cleans up** on
unmount (no leftover ScrollTriggers, no memory leaks, no double-fire in StrictMode).

```tsx
"use client"
import { useRef } from "react"
import gsap from "gsap"
import { ScrollTrigger } from "gsap/ScrollTrigger"
import { useGSAP } from "@gsap/react"

gsap.registerPlugin(ScrollTrigger, useGSAP)

export function Reveal() {
  const root = useRef<HTMLDivElement>(null)

  useGSAP(() => {
    // selectors are scoped to `root` — won't leak to other components
    gsap.from(".item", {
      y: 40, opacity: 0, duration: 0.6, ease: "power3.out", stagger: 0.08,
      scrollTrigger: { trigger: root.current, start: "top 80%" },
    })
  }, { scope: root })           // cleanup is automatic on unmount

  return (
    <div ref={root}>
      <div className="item">A</div>
      <div className="item">B</div>
      <div className="item">C</div>
    </div>
  )
}
```

**Rules for React:**
- Always `"use client"` — GSAP touches the DOM.
- Always `useGSAP(..., { scope: ref })` — never bare `useEffect` + manual cleanup
  unless you have a reason; `useGSAP` handles `revert()` for you.
- Use `gsap.context`/scope, not global selectors, so animations don't collide.
- Set initial hidden state with `gsap.set` or in `from`, not CSS opacity:0 alone
  (avoids a flash if JS is slow — or use `autoAlpha` which combines opacity+visibility).

## Timelines — sequence and overlap

```ts
const tl = gsap.timeline({ defaults: { ease: "power3.out", duration: 0.6 } })
tl.from(".title", { y: 30, opacity: 0 })
  .from(".subtitle", { y: 20, opacity: 0 }, "-=0.4")   // overlap: start 0.4s before prev ends
  .from(".cta", { scale: 0.9, opacity: 0 }, "<")       // "<" = start WITH previous
  .from(".image", { opacity: 0 }, 0.2)                 // absolute time 0.2s

// Position parameter cheatsheet:
//   "+=0.5" after prev ends + 0.5   "-=0.3" overlap by 0.3
//   "<" with prev start             ">" at prev end
//   "<0.2" 0.2 after prev start      1.5 absolute time
```
Timelines beat chaining `delay`s — you can retime the whole sequence by editing
overlaps, and `tl.timeScale(2)` speeds up everything.

## ScrollTrigger — the workhorse

```ts
gsap.to(".panel", {
  xPercent: -100,
  ease: "none",
  scrollTrigger: {
    trigger: ".container",
    start: "top top",      // [trigger position] [viewport position]
    end: "+=2000",         // 2000px of scroll
    scrub: 1,              // tie progress to scrollbar (1 = 1s catch-up smoothing)
    pin: true,             // pin trigger in place while animating
    // markers: true,      // DEV ONLY — visualize start/end, remove before ship
  },
})
```

```
start/end syntax: "top 80%"  → when trigger's top hits 80% down the viewport
scrub: true   → animation progress = scroll progress (no duration)
scrub: 1      → same, but eases/lags by 1s (smoother, premium feel)
toggleActions: "play none none reverse"  → onEnter onLeave onEnterBack onLeaveBack
pin: true     → freezes element while its scroll range plays (carousels, story sections)
```

Common patterns:
```ts
// Reveal on scroll (play once)
ScrollTrigger.batch(".card", {
  start: "top 85%",
  onEnter: (els) => gsap.from(els, { y: 40, opacity: 0, stagger: 0.1, ease: "power3.out" }),
})

// Parallax
gsap.to(".bg", { yPercent: 30, ease: "none",
  scrollTrigger: { trigger: ".section", start: "top bottom", end: "bottom top", scrub: true } })
```

## Easing — the difference between cheap and premium

```
power2/power3.out   → decelerate into place. THE default for entrances. Natural.
power2.in           → accelerate away. For exits.
power2.inOut        → smooth both ends. For loops/state changes.
back.out(1.7)       → slight overshoot. Playful, for pop-in (use sparingly).
elastic.out(1, 0.3) → bouncy. Rarely — looks toy-ish if overused.
expo.out            → dramatic, very fast settle. Hero moments.
"none" (linear)     → ONLY for scrub/continuous (parallax, marquee). Never for entrances.

Durations: micro-interactions 0.15–0.3s | entrances 0.4–0.8s | hero 0.8–1.2s.
Faster than you think feels better. > 1s for a UI transition feels sluggish.
```

## Stagger

```ts
gsap.from(".grid-item", {
  opacity: 0, y: 30, duration: 0.5,
  stagger: { each: 0.06, from: "start" },   // "start" | "center" | "end" | "random" | [x,y] grid
})
```
Small staggers (0.04–0.1s) read as "one fluid motion." Big ones (0.3s+) read as a
slow list — usually not what you want.

## Performance — hit 60fps

```
✅ Animate transform (x, y, scale, rotation, xPercent/yPercent) + opacity/autoAlpha
✅ will-change via gsap.set(el, { willChange: "transform" })  — then clear it after
✅ Use xPercent/yPercent for responsive movement (relative to element size)
✅ Batch DOM reads/writes — GSAP already does this; don't read layout in onUpdate
✅ ScrollTrigger.batch() for many similar elements (one observer vs N)

❌ Animating width/height/top/left/margin/padding → layout thrash, jank
❌ Animating box-shadow/filter on many elements → expensive paint
❌ Huge number of simultaneous tweens → throttle/stagger or use a single timeline
❌ Forgetting to kill ScrollTriggers on route change → leaks (useGSAP fixes this)
```

After a transform animation finishes, clear `will-change` (it costs memory if left on).

## Accessibility — respect reduced motion

```ts
gsap.matchMedia().add("(prefers-reduced-motion: no-preference)", () => {
  // full animations here — only run for users who haven't opted out
  gsap.from(".hero", { y: 50, opacity: 0, duration: 1 })
})
// Users with reduced-motion get the final state instantly. Never animate through
// large motion for them. Essential content must be visible without JS/animation.
```

## Smooth scrolling

Pair with **Lenis** for smooth scroll, and sync it to ScrollTrigger:
```ts
const lenis = new Lenis()
lenis.on("scroll", ScrollTrigger.update)
gsap.ticker.add((t) => lenis.raf(t * 1000))
gsap.ticker.lagSmoothing(0)
```
Don't smooth-scroll inputs/forms-heavy apps — it can hurt usability. Reserve it for
marketing/portfolio/story pages.

## Anti-patterns

```
❌ Bare useEffect + manual gsap cleanup in React   → use useGSAP({ scope })
❌ Global selectors in a component                  → scope to a ref
❌ Animating layout properties                      → use transforms
❌ ease: "none" on entrances                        → power2/3.out
❌ markers: true shipped to production              → dev only
❌ 1.5s+ UI transitions                             → 0.3–0.8s
❌ Animations that ignore prefers-reduced-motion    → wrap in matchMedia
```

## Response format

1. Pick the technique (timeline / ScrollTrigger / stagger) and say why in a line.
2. Give the full `useGSAP`-scoped React component (or vanilla if non-React).
3. State the easing + duration choice and the reduced-motion handling.
4. Flag any property that would have caused jank and what you used instead.
