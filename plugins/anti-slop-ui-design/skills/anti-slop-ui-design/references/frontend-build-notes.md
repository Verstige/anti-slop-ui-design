# Frontend Build Reference Notes

## Backtick Corruption in Template Literals

When patching TypeScript/TSX files that contain **backtick template literals**, `sed` and sometimes `patch` corrupt the backtick character during write operations — especially when the backtick appears inside a JSX `style={}` attribute or a conditional expression.

**Symptom**: TypeScript error `Unterminated string literal` at a line that visually contains a valid backtick in the source file. The error persists even when the source looks correct in `read_file`.

**Root cause**: write_file/patch tool can write a literal backtick (`) into a file without corruption. However, when a sed command or shell heredoc used to inspect the file shows `backtick` as a visible character in the error message, it means the file actually contains a normal single-quote `'` or the template literal was misformed.

**Safe pattern**:
```tsx
// GOOD — avoid backticks inside style={} template literals when patching
borderBottom: scrolled ? `1px solid ${SOLAR.border}` : '1px solid transparent',

// When patching, rewrite the whole expression to avoid nested backticks
style={{
  borderBottom: scrolled ? '1px solid ' + SOLAR.border : '1px solid transparent',
}}
```

## Next.js Landing Page: Framer-Motion Only (No GSAP)

For standard SaaS marketing landing pages, framer-motion is sufficient and preferred over GSAP:
- `AnimatePresence` for mount/unmount transitions
- `useScroll` + `useTransform` for parallax
- `useInView` for scroll-triggered reveals
- `motion.div` with `whileHover`, `whileTap` for micro-interactions
- `staggerChildren` variants for orchestrated list reveals
- Infinite loops via `animate` with `repeat: Infinity` and `ease: 'easeInOut'`

GSAP/ThreeJS: only when the page requires canvas/WebGL backgrounds, complex scrolltelling sequences, or 3D object rendering. For everything else, framer-motion is faster to ship and easier to debug.

## Railway: Single-Service Next.js Architecture

Railway deploys Next.js apps as a single `web` service by default. There is no separate `api` or `backend` service unless explicitly created. If an app has API routes (`/app/api/...`), those are handled by the same Next.js server process in the `web` service.

When deploying:
- `railway up` deploys the current directory to the `web` service
- `railway up --service <name>` targets a named service
- If only one service exists, `railway up` is sufficient
- Next.js API routes at `/api/...` are served by the same `web` container — no CNAME or routing needed for internal API calls

## the broker: Diagnosing "Not authenticated"

When `POST /api/the broker/connect` returns `{"error":"Not authenticated"}`:
1. **Check if the user is actually logged in** — does the browser have a Supabase auth session cookie?
2. **Check sessionStorage vs localStorage** — sessionStorage survives page reloads but not navigation to a different origin; localStorage persists across sessions
3. **Check `autoLogin` behavior** — the autoLogin param fills the login form with a hardcoded email but does NOT automatically submit. The user must physically click the login button to set the session cookie.
4. **The Railway deploy is NOT the problem** if you can `curl` the endpoint and it returns JSON (even `{"error":"Not authenticated"}`). A missing environment variable would return a 500. A 200 with an auth error = route is running, user just isn't authenticated.
5. **Resolution**: Have the user log in through the UI normally (not autoLogin), then retry the API call.

## SolarOS Design System (2026-05-27)

Used for SolarOS redesign — a dark SaaS landing page for a solar company OS.

**Color palette**:
- Background: `#050507`
- Surface: `#0D0D14`
- Surface2: `#111120`
- Primary (orange): `#FF6B00`
- Secondary (teal): `#00D4AA`
- Accent (yellow): `#FFD60A`
- Border: `rgba(255,255,255,0.06)`
- BorderMid: `rgba(255,255,255,0.1)`
- Muted: `#6B7280`

**Font**: Default Next.js font ( Geist-like ), tracking-tight on headlines

**Motion**: 3 blob mesh gradient background, count-up stats on scroll, framer-motion stagger reveals, floating badge animations
