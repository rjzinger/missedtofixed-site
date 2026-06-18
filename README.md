# missedtofixed.com

Marketing site and legal pages for **Missed to Fixed** — missed-call-to-SMS
lead capture for local US service businesses.

## Deployment

Static site hosted on Vercel (project `missedtofixed-site`). `vercel.json`
enables clean URLs so legal pages resolve without the `.html` extension.

| Path | File | Purpose |
|------|------|---------|
| `/` | `index.html` | Marketing landing page |
| `/privacy` | `privacy.html` | Privacy Policy (A2P 10DLC compliant) |
| `/terms` | `terms.html` | Terms of Service + SMS terms |
| `/stories` | `stories/index.html` | Founding 50 pre-sale page (children's bedtime stories) |
| `/stories/privacy` | `stories/privacy.html` | Privacy policy for the pre-sale page |
| `/game` | `game/index.html` | "David & the Giant" — faith-based match-3 game for kids 8-12 |

Production: https://missedtofixed.com

## stories/

Self-contained, single-action pre-sale ("Founding 50") landing page for a
separate, unrelated product — personalized, faith-based children's bedtime
stories. Static HTML/CSS/JS, no build step. Before launch, three things must
be wired up (each marked with a `TODO` comment in `stories/index.html`):

1. **Stripe Payment Link** — replace `REPLACE_WITH_PAYMENT_LINK` with the live
   $9/mo founding link; set its success URL to `/stories/?reserved=1`.
2. **Email capture** — point the waitlist form at ConvertKit / Beehiiv /
   MailerLite (swap the form for an embed, or set `FORM_ENDPOINT`).
3. **Analytics** — drop in the Plausible or GA4 snippet (commented in `<head>`);
   events (`hero_cta_click`, `sample_story_view`, `sample_story_swap`,
   `waitlist_submit`, `reserve_click`, `reserve_complete`) are already wired.

Sample stories live in the `SAMPLE_STORIES` array in the page; add entries to
extend the "Read another" rotation.

## game/

**David & the Giant** — a self-contained, faith-based **match-3 game** for kids
ages 8-12, themed on the story of David & Goliath (1 Samuel 17). Single static
HTML file, no build step, no external assets (sound is synthesized with the Web
Audio API; tiles are emoji). Works in any mobile/desktop browser.

How it plays: swap adjacent tiles to line up 3+; every 🪨 stone you match is
slung at Goliath to drain his health bar. Beat him before you run out of moves.
Five story levels each open with a narrative beat and reward a kid-friendly Bible
verse on completion. High score and level progress are saved in `localStorage`
(`davidGiant_v1`); a mute toggle is included.

To extend: edit the `LEVELS` array (story, HP, move limit, verse) and the `TILE`
array (emoji + colors) near the top of the `<script>` in `game/index.html`.

## source/

Original long-form documents the deployed pages were built from — privacy
policy, terms of service, the A2P consent description, and landing page copy.
Kept for provenance; not served.
