# From SPA to SSR: How I Took Google PageSpeed from 20 to 100

### Executive Summary

I turned a legacy SPA-hybrid (Rails + AngularJS) into fast, server-rendered pages that ship only page-scoped CSS/JS. I automated per-page CSS extraction, eliminated the monolithic JS from first paint, and lazy-loaded interactive widgets. The result: mobile PageSpeed 20 → 97–100, sub-second desktop main-content render, and green Core Web Vitals across ~4.5k pages.  

### Context & Constraints

* Legacy state: one 1.5 MB CSS bundle and one 2.2 MB JS bundle (3.7 MB total; 713 KB gz) blocking first render; CSS class duplication caused widespread collisions. Past attempts (hand-picked above-the-fold CSS) proved unmaintainable and worsened collisions.
* Risk profile: I was the only full-time engineer on the effort, so I pursued incremental change over a risky big-bang rewrite. 

### Initial State (why the score was ~20)

A SPA-ish front end forced heavy client-side work before showing content; global CSS/JS bundles created render-blocking weight. On landing pages, desktop “main content” appeared after ~3–4 s (mobile ~10+ s); mobile PageSpeed was < 20.  

### Diagnosis

The **monolithic CSS/JS** and **hydration-first path** were the core bottlenecks. Past “above-the-fold CSS” hacks increased conflicts and had no maintenance path—so I needed a systematic way to ship **only what a page actually uses**, and to **decouple interactivity from first paint**. 

### Strategy: Why SSR (and why this flavor)

I chose **incremental SSR**: render full HTML on the server, inline the minimal CSS/JS needed per page, and keep the rest async. Where an area initially depended on AJAX, I inlined the JSON first, then refactored it to render server-side. Complex, mid-page interactivity became **self-contained widgets** that lazy-load after the first paint. 

### Implementation

**A tiny “component system” with page-scoped assets.**
I introduced a ViewComponent-inspired approach using ERB partials and two helpers—`include_component_css` / `include_component_js`—to inline each component’s minified assets once per page. This made page assembly declarative and safe: add a component → its CSS/JS appears; remove it → they vanish. 

**Automated per-page CSS extraction for legacy styles.**
To stop shipping 1.5 MB of legacy CSS to every page, I created **profiles** per controller#action and used **MinimalCSS** to crawl each page and store only the rules actually used. If a profile existed, I inlined that CSS and skipped `application.css`. I rolled this across the site—**32 profiles covering ~4.5k pages**. 

**Widget system: interactivity without first-paint tax.**
The 2.2 MB (551 KB gz) `application.js` was removed from initial render. I extracted simple UI (e.g., top nav) to small PlainJS and shipped it inline. Heavier, mid-page functionality moved into a **widget service** with a **declarative placeholder** and a **small loader** that discovers placeholders and lazy-loads the widget (initially via iframes with iFrameResizer). Embedding a widget is literally two lines (script + div).  
This replaced a 551 KB gz first-paint tax with a tiny (10 KB gz) async loader and deferred the heavy code until it was needed. 

**Widget refactors.**
I then converted legacy AngularJS widgets into **lean Vue bundles** that ship just their HTML/CSS/JS (about **60–65 KB gz** each). Later I added a **direct-HTML injection path** (faster than iframes) and solved style collisions with randomized/prefixed classes plus build-time HTML post-processing. 

### Results

**Key outcomes**

| Metric                        |    Before |              After |          Δ |
| ----------------------------- | ---------:| ------------------:| ----------:|
| Google PageSpeed (mobile)     |      < 20 |             97–100 |     +77–80 |
| Desktop: time to main content |     3–4 s |              < 1 s |     −2–3 s |
| Mobile: time to main content  |    ~10+ s |              < 2 s |      −8+ s |
| Initial JS at first paint     | 551 KB gz |    10 KB gz loader | −541 KB gz |
| Global CSS at first paint     | 162 KB gz | per-page extracted |          — |

**Coverage & status**

All ~4.5k pages achieved green Core Web Vitals in Google Search Console; the **new stack serves 100% of traffic**. 

### Business Impact

- The widget service enabled **partner syndication** using embeddable widgets under revenue-sharing agreements.
- PPC landing pages built on this architecture became **the fastest** and **the highest-converting** across the company’s websites based on internal tracking.

### Risks & Trade-offs

* The direct-HTML widget path is the fastest, but needed class randomization/post-processing to avoid style collisions. Some elements (like `input` tags) can still cause style collisions.

### My Role & Collaboration

I led the migration end-to-end as the sole full-time engineer: defined the plan, built the component/inline-asset pattern, automated CSS extraction, created the widget service, and refactored AngularJS widgets to Vue. 

### Reusable Playbook / Checklist

1. Introduce a simple component pattern with page-scoped assets added declaratively. 
2. Automate per-page CSS extraction (MinimalCSS) to skip the global CSS. 
3. Refactor monolithic JS into:
   - Small PlainJS components included on a page declaratively (see #1)
   - Heavy widgets with a tiny loader + placeholder discovery, lazy-load the widgets. 
4. Refactor heavy widgets into ~60–65 KB gz bundles (only used CSS + JS); add direct-HTML injection with collision-proofing for speed.

### What I’d bring to your team

- A **repeatable pattern for high-performance pages**—server-rendered HTML, declarative page-scoped delivery with components, automated per-page CSS extraction for legacy styles, and lazy widgets—cleanly integrated into Rails.
- **Measurable wins**: PageSpeed mobile 20→97–100; desktop main content 3–4s→<1s; mobile ~10+s→<2s; removed a 551 KB gz first-paint tax with a 10 KB loader.
- **Business impact**: PPC landers became the fastest and highest-converting in the company; embeddable widgets opened a rev-share partner channel.
- **End-to-end ownership**: I led diagnosis, design, rollout, and AngularJS→Vue widget refactors through safe, incremental releases.

If you need pages to feel instant on mobile—without shipping megabytes of JS—I’ve already shipped that playbook, and the numbers above are why it sticks.

### Appendix (artifacts)

#### Lighthouse screenshot, Article (score 97)

![Article](https://i.imgur.com/YjHHU8k.jpeg)

#### Lighthouse screenshot, PPC landing page (score 100)

![Landing page](https://i.imgur.com/MKcNtKH.jpeg)

#### ActiveAdmin view for "CSS Profiles"

![Admin](https://i.imgur.com/IL1OyQ6.jpeg)
