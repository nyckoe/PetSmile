# PetSmile

A swipe-based, AI-assisted pet adoption platform connecting shelters with adopters.

- **[petsmile-project-plan.md](./petsmile-project-plan.md)** — full product plan, feature set, system architecture, data model, and project roadmap.
- **[index.html](./index.html)** — a self-contained, interactive front-end demo of the core product experience.

## Viewing the demo

No build step or server required — just open `index.html` in a modern browser (Chrome, Edge, Firefox, or Safari).

The demo is a single static HTML file (vanilla HTML/CSS/JS) so it can also be published directly with any static host (GitHub Pages, Netlify, Vercel, etc.) without a build pipeline.

## What the demo shows

- A hero section introducing PetSmile's core value proposition, followed by a scroll-snapped, hover-to-focus split view of the two sides of the product:
  - **Shelter side** — AI-assisted photo upload, pet profile drafting, plain-language requirement extraction into structured rules, and availability scheduling.
  - **Adopter side** — attribute/location filtering, natural-language search parsed into filters, a swipeable match deck, and video-chat → in-person-visit booking.

This is a demo/prototype for product and design review — it is not connected to a backend and all data shown is illustrative.

## Publishing with GitHub Pages

1. Push this repository to GitHub.
2. In the repo settings, enable **Pages** for the `main` branch (root directory).
3. GitHub will serve `index.html` directly at the generated Pages URL.
