# MBTI Personality Test Website — Product Requirements Document

> Version: 1.1.0
> Date: 2026-04-03
> Status: Confirmed

---

## 1. Product Overview

### 1.1 Product Vision

A modern MBTI personality test website that delivers a fluid, visually polished testing experience across web and mobile platforms. Users complete a standardized personality questionnaire and receive detailed 16-personality-type results with dimension breakdowns, trait descriptions, and shareable reports.

### 1.2 Target Users

| Segment | Description |
|---|---|
| Casual testers | Users curious about their personality type, accessing via mobile or web |
| Returning users | Users re-testing after a period, wanting to compare results |
| Social sharers | Users who want to share and compare personality types with friends |

### 1.3 Core Value Proposition

- **Zero-friction testing**: No login required, start testing immediately
- **Rich result presentation**: Detailed personality profiles with visual dimension breakdowns
- **Fluid experience**: Smooth animations and responsive design across all devices
- **Up-to-date questions**: Periodic synchronization with 16personalities question bank

---

## 2. Technical Architecture

### 2.1 Tech Stack

| Layer | Technology | Rationale |
|---|---|---|
| Frontend | React 18 + TypeScript | User-specified; mature ecosystem |
| UI Library | MUI v7 (Material UI) | Standard React Material Design library; built-in responsive Grid system (xs/sm/md/lg/xl breakpoints), comprehensive component set |
| Animation | Framer Motion | Deep React integration; page transitions, card animations, progress indicators |
| Routing | React Router v7 | Mature SPA routing |
| State Management | Zustand | Lightweight; ideal for quiz progress, score tracking |
| Backend | Fastify (Node.js) | Faster than Express, native TypeScript, plugin ecosystem |
| API Data Source | [16personalities-api](https://github.com/SwapnilSoni1999/16personalities-api) (Unofficial) | Pre-built question bank + scoring engine; MIT License |
| Database | SQLite (dev) / PostgreSQL (prod) | Question cache, test results, sync metadata |
| ORM | Prisma | Type-safe database access, migration support |
| Cron Scheduler | node-cron | Periodic question bank synchronization |
| Build Tool | Vite | Fast HMR, React ecosystem standard |
| HTTP Client | Axios / fetch | API communication |

### 2.2 System Architecture

```
┌─────────────────────────────────────────────────┐
│                    Client                        │
│  React 18 + MUI v7 + Framer Motion + Zustand    │
│  ┌───────┐ ┌───────┐ ┌───────┐ ┌─────────────┐ │
│  │ Home  │ │ Test  │ │Result │ │  History    │ │
│  └───────┘ └───────┘ └───────┘ └─────────────┘ │
└────────────────────┬────────────────────────────┘
                     │ REST API
┌────────────────────▼────────────────────────────┐
│                   Server                         │
│            Fastify + Prisma ORM                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │Questions │ │ Results  │ │  Sync Service    │ │
│  │  API     │ │   API    │ │ (node-cron)      │ │
│  └──────────┘ └──────────┘ └────────┬─────────┘ │
└─────────────────────────────────────┼───────────┘
                                      │ HTTP
                        ┌─────────────▼────────────┐
                        │  16personalities-api.com  │
                        │  (External Data Source)   │
                        │  GET /questions           │
                        │  POST /personality        │
                        └──────────────────────────┘
```

### 2.3 Data Flow

```
[Scheduled Sync Job]
    │
    ▼
GET 16personalities-api.com/api/personality/questions
    │
    ▼
Parse & Store in DB (questions table)
    │
    ▼
Client requests questions ───► Server returns from DB cache
    │
    ▼
User submits answers ───► Server forwards to 16p API
    │                         │
    │                         ▼
    │               POST 16personalities-api.com/api/personality
    │                         │
    │                         ▼
    │               Receive: type code, traits, scores
    │
    ▼
Server stores result + returns to client
    │
    ▼
Client renders Result Page with animations
```

---

## 3. Data Models

### 3.1 Question

```typescript
interface Question {
  id: string;          // Base64-encoded question identifier (from 16p API)
  text: string;        // Question text (e.g., "You regularly make new friends.")
  category: string;    // Dimension mapping hint (derived from position/grouping)
  options: Option[];   // Answer options
}

interface Option {
  label: string;       // e.g., "Agree strongly"
  value: number;       // -3, -2, -1, 0, 1, 2, 3
}
```

### 3.2 Answer (Client Submission)

```typescript
interface Answer {
  id: string;          // Question ID
  value: number;       // Selected option value (-3 to 3)
}
```

### 3.3 TestResult

```typescript
interface TestResult {
  id: string;                  // Unique result ID (UUID)
  sessionId: string;           // Anonymous session identifier
  niceName: string;            // e.g., "Architect"
  fullCode: string;            // e.g., "INTJ-A"
  snippet: string;             // Short personality description
  traits: TraitScore[];        // Dimension breakdown
  createdAt: string;           // ISO timestamp
  answers: Answer[];           // Stored answers for re-calculation
}

interface TraitScore {
  key: string;                 // e.g., "introverted", "intuitive"
  label: string;               // e.g., "Energy", "Mind"
  trait: string;               // e.g., "Introverted", "Intuitive"
  score: number;               // Raw score
  pct: number;                 // Percentage (0-100)
  description: string;         // Trait description
  color: string;               // Display color
}
```

### 3.4 SyncLog

```typescript
interface SyncLog {
  id: string;
  source: string;              // "16personalities-api"
  status: "success" | "failed";
  questionsFetched: number;
  errorMessage?: string;
  syncedAt: string;            // ISO timestamp
}
```

---

## 4. API Specification

### 4.1 Question Bank Endpoints

#### GET /api/questions

Fetch all cached questions.

**Response 200:**

```json
{
  "total": 120,
  "lastSynced": "2026-04-03T10:00:00Z",
  "questions": [
    {
      "id": "WW91IHJlZ3VsYXJseSBtYWtlIG5ldyBmcmllbmRzLg",
      "text": "You regularly make new friends.",
      "options": [
        { "label": "Agree strongly", "value": 3 },
        { "label": "Agree moderately", "value": 2 },
        { "label": "Agree a little", "value": 1 },
        { "label": "Neither agree nor disagree", "value": 0 },
        { "label": "Disagree a little", "value": -1 },
        { "label": "Disagree moderately", "value": -2 },
        { "label": "Disagree strongly", "value": -3 }
      ]
    }
  ]
}
```

#### POST /api/sync

Manually trigger question bank synchronization (admin use).

**Response 200:**

```json
{
  "status": "success",
  "questionsFetched": 120,
  "syncedAt": "2026-04-03T10:00:00Z"
}
```

### 4.2 Test Endpoints

#### POST /api/test/submit

Submit test answers and receive personality result.

**Request:**

```json
{
  "answers": [
    { "id": "WW91IHJlZ3VsYXJseSBtYWtlIG5ldyBmcmllbmRzLg", "value": 3 },
    { "id": "...", "value": -1 }
  ],
  "gender": "Male"
}
```

`gender` accepts: `"Male"` | `"Female"` | `"Other"`

**Response 200:**

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "niceName": "Architect",
  "fullCode": "INTJ-A",
  "snippet": "Imaginative and strategic thinkers, with a plan for everything.",
  "scales": ["Energy", "Mind", "Nature", "Tactics", "Identity"],
  "traits": [
    {
      "key": "introverted",
      "label": "Energy",
      "color": "blue",
      "score": 32,
      "pct": 66,
      "trait": "Introverted",
      "description": "Introverted individuals tend to prefer fewer, yet deep and meaningful, social interactions."
    },
    {
      "key": "intuitive",
      "label": "Mind",
      "color": "yellow",
      "score": 37,
      "pct": 69,
      "trait": "Intuitive",
      "description": "Intuitive individuals are imaginative, curious, and interested in possibilities."
    },
    {
      "key": "thinking",
      "label": "Nature",
      "color": "green",
      "score": 35,
      "pct": 68,
      "trait": "Thinking",
      "description": "Thinking individuals focus on objectivity and rationality."
    },
    {
      "key": "judging",
      "label": "Tactics",
      "color": "purple",
      "score": 22,
      "pct": 61,
      "trait": "Judging",
      "description": "Judging individuals are decisive, thorough, and highly organized."
    },
    {
      "key": "assertive",
      "label": "Identity",
      "color": "red",
      "score": 38,
      "pct": 69,
      "trait": "Assertive",
      "description": "Assertive individuals are self-assured, even-tempered, and resistant to stress."
    }
  ],
  "avatarSrc": "https://www.16personalities.com/static/animations/avatars/all/architect-male.json",
  "avatarSrcStatic": "https://www.16personalities.com/static/images/personality-types/avatars/intj-architect-male.svg",
  "createdAt": "2026-04-03T12:00:00Z"
}
```

#### GET /api/test/result/:id

Retrieve a previously stored test result.

### 4.3 System Endpoints

#### GET /api/health

Health check endpoint.

#### GET /api/sync/status

Get last sync status and metadata.

---

## 5. Page Specifications

### 5.1 Home Page (`/`) — *Deferred to Phase 6*

**Purpose**: Landing page with product introduction and call-to-action.

| Element | Specification |
|---|---|
| Hero Section | Full-width gradient background; animated personality type icons floating; title + tagline |
| CTA Button | Material Design elevated button; "Start Test"; ripple effect; navigates to `/test` |
| Features Section | 3-column grid (desktop) / stacked (mobile); icons + text describing the test |
| Footer | Brief credits, privacy note |

**Animations**:
- Hero text: Fade-in + slide-up on mount (Framer Motion `initial={{ opacity: 0, y: 20 }}`)
- Floating icons: Infinite gentle bob (Framer Motion `animate={{ y: [0, -10, 0] }}`, `repeat: Infinity`)
- CTA Button: Pulse animation on hover

### 5.2 Test Page (`/test`) — *MVP Core*

**Purpose**: Core quiz interface. One question per screen with smooth transitions.

| Element | Specification |
|---|---|
| Progress Bar | Linear, sticky top; shows current question / total (e.g., "12 / 120"); smooth width transition |
| Question Card | Centered card; question text in `h6` typography; 7 answer buttons in 2-column grid (desktop) / stacked (mobile) |
| Navigation | "Previous" and "Next" buttons at card bottom; disabled states when at boundaries |
| Answer State | Selected button shows filled variant + primary color; unselected show outlined variant |
| Auto-advance | After selecting an answer, auto-advance to next question after 400ms delay with animation |

**Answer Option Display** (7-point Likert Scale):

```
┌──────────────────────────────────────────────────────────┐
│  [Agree strongly]  [Agree moderately]  [Agree a little]  │
│               [Neither agree nor disagree]                │
│  [Disagree strongly] [Disagree moderately] [Disagree a little] │
└──────────────────────────────────────────────────────────┘
```

**Mobile Layout**:

```
┌────────────────────┐
│   ════════════ 12% │  ← Progress bar
│                    │
│  "You regularly    │
│   make new         │
│   friends."        │
│                    │
│  ● Agree strongly  │
│  ○ Agree moderately│
│  ○ Agree a little  │
│  ○ Neither         │
│  ○ Disagree a      │
│    little          │
│  ○ Disagree mod.   │
│  ○ Disagree strongly│
│                    │
│  [← Prev] [Next →]│
└────────────────────┘
```

**Animations**:
- Question transition: Slide left (next) / slide right (prev) with `AnimatePresence` + `motion.div`
- Answer selection: Scale pop (`scale: [1, 1.05, 1]`) + color transition
- Progress bar: Smooth width animation (`transition={{ duration: 0.3 }}`)

**State Management** (Zustand):

```typescript
interface TestStore {
  questions: Question[];
  currentIndex: number;
  answers: Map<string, number>;   // questionId → selected value
  isLoading: boolean;
  
  // Actions
  setQuestions: (questions: Question[]) => void;
  answerQuestion: (questionId: string, value: number) => void;
  goToNext: () => void;
  goToPrev: () => void;
  getCurrentQuestion: () => Question | null;
  getProgress: () => { current: number; total: number; pct: number };
  isComplete: () => boolean;
  reset: () => void;
}
```

**Persistence**: Answers saved to `localStorage` on each selection. On page reload, resume from last answered question.

### 5.3 Result Page (`/result/:id`) — *MVP Core*

**Purpose**: Display personality test results with rich visualizations.

| Element | Specification |
|---|---|
| Personality Type Code | Large display (e.g., "INTJ-A") with typewriter animation; below: nice name ("Architect") |
| Avatar | Animated avatar from 16p CDN (Lottie JSON), static SVG fallback |
| Trait Radar/Bar Chart | 5-trait horizontal bar chart; each bar animated from 0 to pct%; color-coded per trait |
| Description Cards | Expandable cards for each trait with description text; staggered fade-in animation |
| Dimension Summary | 5-dimension comparison: e.g., "66% Introverted — 34% Extraverted"; sliding scale visual |
| Share Button | Generate shareable image (html2canvas) + copy link button |
| Retake CTA | "Test Again" button navigating to `/test` |

**Trait Dimension Colors** (from 16p API):

| Scale | Color |
|---|---|
| Energy (E/I) | Blue |
| Mind (S/N) | Yellow/Green |
| Nature (T/F) | Green |
| Tactics (J/P) | Purple |
| Identity (A/T) | Red |

**Animations**:
- Page enter: Staggered reveal (type code → avatar → bars → descriptions)
- Trait bars: `motion.div` with `initial={{ width: 0 }}` → `animate={{ width: "${pct}%" }}`
- Share: Material Design `Snackbar` with "Link copied!" feedback

### 5.4 History Page (`/history`) — *Deferred to Phase 6*

**Purpose**: Show past test results for comparison.

| Element | Specification |
|---|---|
| Result List | Chronological list; each entry shows date, type code, and dimension mini-bars |
| Compare | Side-by-side comparison of two results (optional v2 feature) |
| Clear | Button to clear all local history |

**Storage**: `localStorage` keyed by anonymous session ID.

---

## 6. Question Bank Synchronization

### 6.1 Sync Strategy

| Parameter | Value |
|---|---|
| Source API | `https://16personalities-api.com/api/personality/questions` |
| Sync Frequency | Every 24 hours (node-cron `0 0 * * *`) |
| Fallback | Serve cached questions from DB if API is unreachable |
| Retry | 3 attempts with exponential backoff (1s, 5s, 15s) |
| Manual Trigger | `POST /api/sync` endpoint for admin-initiated sync |

### 6.2 Sync Flow

```
1. Cron triggers at 00:00 UTC
2. GET 16personalities-api.com/api/personality/questions
3. Validate response (expect ~120 questions, each with id + text + options)
4. Diff against existing DB questions (by id)
5. INSERT new questions, UPDATE changed text, soft-delete removed questions
6. Log result to sync_logs table
7. On failure: log error, continue serving cached data
```

### 6.3 Resilience

- **API Down**: Serve last successfully synced questions from DB
- **Partial Response**: Reject and retry; never serve incomplete question sets
- **Data Validation**: Each question must have `id`, `text`, and 7 options with values -3 to 3
- **Circuit Breaker**: If 3 consecutive syncs fail, alert and increase retry interval to 6 hours

---

## 7. Responsive Design Specification

### 7.1 Breakpoints (MUI Default)

| Breakpoint | Width | Target |
|---|---|---|
| `xs` | 0–599px | Mobile phones |
| `sm` | 600–899px | Tablets (portrait) |
| `md` | 900–1199px | Tablets (landscape) / small laptops |
| `lg` | 1200–1535px | Desktops |
| `xl` | 1536px+ | Large screens |

### 7.2 Layout Adaptation

| Component | Mobile (xs/sm) | Desktop (md+) |
|---|---|---|
| Navigation | Bottom app bar | Top app bar |
| Question Card | Full-width, no horizontal margin | Centered, max-width 720px |
| Answer Options | Stacked vertical list | 2-column grid (3+3+1) |
| Progress Bar | Sticky top | Sticky top |
| Result Trait Bars | Full-width bars | Wider bars with labels beside |
| Hero Section | Stacked, smaller text | Side-by-side, larger typography |

### 7.3 Touch Optimization

- Minimum touch target: 48x48px (Material Design spec)
- Answer buttons: Full-width on mobile, minimum height 48px
- Swipe gesture support: Swipe left/right to navigate between questions (optional)
- Disable hover-only interactions on touch devices

---

## 8. Animation Specification

### 8.1 Page Transitions

```typescript
// React Router + Framer Motion page transition
const pageVariants = {
  initial: { opacity: 0, x: 50 },
  animate: { opacity: 1, x: 0, transition: { duration: 0.3, ease: "easeOut" } },
  exit: { opacity: 0, x: -50, transition: { duration: 0.2 } },
};
```

### 8.2 Question Card Transition

```typescript
// Direction-aware slide
const slideVariants = {
  enter: (direction: number) => ({ x: direction > 0 ? 300 : -300, opacity: 0 }),
  center: { x: 0, opacity: 1 },
  exit: (direction: number) => ({ x: direction < 0 ? 300 : -300, opacity: 0 }),
};
```

### 8.3 Answer Selection

```typescript
// Pop + color transition
const selectedVariant = {
  scale: [1, 1.05, 1],
  transition: { duration: 0.2 },
};
```

### 8.4 Result Reveal

```typescript
// Staggered reveal sequence
const containerVariants = {
  animate: { transition: { staggerChildren: 0.15 } },
};
const itemVariants = {
  initial: { opacity: 0, y: 20 },
  animate: { opacity: 1, y: 0, transition: { duration: 0.4 } },
};
```

### 8.5 Performance Budget

| Metric | Target |
|---|---|
| FCP (First Contentful Paint) | < 1.5s |
| LCP (Largest Contentful Paint) | < 2.5s |
| CLS (Cumulative Layout Shift) | < 0.1 |
| Animation FPS | ≥ 60fps on mid-range devices |
| JS Bundle (gzipped) | < 200KB initial load |

---

## 9. Error Handling

### 9.1 Client-Side

| Scenario | Behavior |
|---|---|
| Questions fail to load | Show retry dialog with "Reload" button; cache last successful load in localStorage |
| Network error during submit | Show error Snackbar; save answers locally; offer "Retry" button |
| Incomplete answers on submit | Disable submit button until all questions answered; highlight unanswered questions |
| 16p API returns error | Server returns fallback message; client shows "Results temporarily unavailable" |

### 9.2 Server-Side

| Scenario | Behavior |
|---|---|
| 16p API unreachable | Return cached questions; log error to sync_logs |
| Invalid answer format | 400 Bad Request with validation error details |
| Rate limiting | 429 Too Many Requests; client shows "Please wait" message |
| Internal server error | 500 with generic message; log full error to server logs |

---

## 10. Security Considerations

| Concern | Mitigation |
|---|---|
| XSS | React auto-escapes; sanitize question text before rendering |
| CSRF | Use SameSite cookies + CSRF tokens for POST endpoints |
| Rate Limiting | Apply per-IP rate limiting on `/api/test/submit` (max 10/hour) |
| Input Validation | Validate all answer payloads with Zod/Joi schemas |
| HTTPS | Enforce HTTPS in production |
| Dependency Audit | Run `npm audit` in CI pipeline |
| 16p API Key | If authentication is added later, store in environment variable, never in client code |

---

## 11. Project Structure

```
mbti-app/
├── client/                          # React Frontend
│   ├── public/
│   │   └── index.html
│   ├── src/
│   │   ├── animations/              # Framer Motion variant configs
│   │   │   ├── pageTransitions.ts
│   │   │   ├── cardTransitions.ts
│   │   │   └── resultAnimations.ts
│   │   ├── components/              # Reusable UI components
│   │   │   ├── AnswerButton.tsx
│   │   │   ├── QuestionCard.tsx
│   │   │   ├── ProgressBar.tsx
│   │   │   ├── TraitBar.tsx
│   │   │   ├── ShareButton.tsx
│   │   │   └── Layout.tsx
│   │   ├── hooks/                   # Custom React hooks
│   │   │   ├── useQuiz.ts
│   │   │   └── useResponsive.ts
│   │   ├── lib/                     # Utilities
│   │   │   ├── api.ts               # Axios instance + API calls
│   │   │   ├── localStorage.ts      # Persistence helpers
│   │   │   └── types.ts             # Shared TypeScript types
│   │   ├── pages/                   # Route pages
│   │   │   ├── HomePage.tsx
│   │   │   ├── TestPage.tsx
│   │   │   ├── ResultPage.tsx
│   │   │   └── HistoryPage.tsx
│   │   ├── stores/                  # Zustand stores
│   │   │   └── testStore.ts
│   │   ├── theme/                   # MUI theme customization
│   │   │   ├── theme.ts             # Palette, typography, breakpoints
│   │   │   └── components.ts        # Component-level style overrides
│   │   ├── App.tsx                  # Root component with router
│   │   └── main.tsx                 # Entry point
│   ├── index.html
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── package.json
│
├── server/                          # Node.js Backend
│   ├── prisma/
│   │   ├── schema.prisma            # DB schema
│   │   └── seed.ts                  # Initial data seeder
│   ├── src/
│   │   ├── routes/
│   │   │   ├── questions.ts         # GET /api/questions
│   │   │   ├── test.ts              # POST /api/test/submit, GET /api/test/result/:id
│   │   │   ├── sync.ts              # POST /api/sync, GET /api/sync/status
│   │   │   └── health.ts            # GET /api/health
│   │   ├── services/
│   │   │   ├── questionService.ts   # Question CRUD + caching
│   │   │   ├── scoringService.ts    # Proxy to 16p API + result storage
│   │   │   └── syncService.ts       # Sync logic + retry + logging
│   │   ├── jobs/
│   │   │   └── syncJob.ts           # node-cron scheduled job
│   │   ├── middleware/
│   │   │   ├── rateLimiter.ts
│   │   │   └── errorHandler.ts
│   │   ├── config/
│   │   │   └── index.ts             # Environment config
│   │   └── index.ts                 # Fastify server entry
│   ├── tsconfig.json
│   └── package.json
│
├── shared/                          # Shared types (client + server)
│   └── types.ts
│
├── doc/
│   └── PRD.md                       # This document
│
├── .gitignore
├── .env.example
└── README.md
```

---

## 12. Milestone Plan

> **Confirmed Scope (v1.1)**: MVP 只包含测试页 + 结果页。首页、历史页后续迭代。
> **Deployment**: 本地开发为主，SQLite 作为唯一数据库。
> **Design**: 自由发挥，基于 MUI 默认风格优化。
> **Question Sync**: 定时自动同步（node-cron 24h）+ 手动触发 `POST /api/sync`。

### Phase 1: Foundation

- [ ] Initialize monorepo structure (client + server + shared)
- [ ] Set up Vite + React 18 + TypeScript + MUI v7
- [ ] Set up Fastify + Prisma + SQLite
- [ ] Implement MUI custom theme (colors, typography, breakpoints)
- [ ] Define shared TypeScript types

### Phase 2: Question Sync & API

- [ ] Implement question sync service (node-cron 24h, retry + circuit breaker)
- [ ] Build `GET /api/questions` endpoint (serve from DB cache)
- [ ] Build `POST /api/sync` endpoint (manual trigger)
- [ ] Build `GET /api/health` endpoint
- [ ] Seed initial questions from 16p API on first run

### Phase 3: Test Page (`/test`)

- [ ] Build TestPage with QuestionCard component
- [ ] Implement Zustand testStore (progress, answers, navigation)
- [ ] 7-point Likert scale answer buttons (responsive: 2-col desktop / stacked mobile)
- [ ] Add localStorage persistence for answer state (resume on reload)
- [ ] Implement question card slide animations (Framer Motion AnimatePresence)
- [ ] Add auto-advance after answer selection (400ms delay)
- [ ] Sticky progress bar with smooth animation

### Phase 4: Result Page (`/result/:id`)

- [ ] Build `POST /api/test/submit` endpoint (proxy to 16p API, store result)
- [ ] Build `GET /api/test/result/:id` endpoint
- [ ] Build ResultPage with staggered reveal animation
- [ ] Personality type code display (large typewriter animation)
- [ ] Implement 5-trait bar chart visualization (color-coded, animated)
- [ ] Implement dimension summary (sliding scale visual, e.g., "66% Introverted — 34% Extraverted")
- [ ] Description cards for each trait (expandable, staggered fade-in)
- [ ] Static avatar display (SVG from 16p CDN)
- [ ] Share button (copy result URL)
- [ ] "Test Again" CTA → navigate to `/test`

### Phase 5: Polish & Responsive

- [ ] Mobile-specific layout optimizations
- [ ] Touch optimization (48px min targets)
- [ ] Error handling and loading states
- [ ] Performance optimization (lazy loading, bundle analysis)

### Phase 6 (Deferred)

- [ ] Home Page (`/`)
- [ ] History Page (`/history`) with localStorage-backed results
- [ ] Animated avatar (Lottie) on result page
- [ ] Dark mode toggle
- [ ] Swipe gesture navigation on mobile
- [ ] Page transition animations across all routes
- [ ] PWA support / i18n / auth

---

## 13. Acceptance Criteria

### Must Have (MVP)

- [ ] User can complete all 120 questions and receive a personality result
- [ ] Result displays correct 16-personality type code (e.g., INTJ-A)
- [ ] Result shows 5-trait dimension breakdown with percentages
- [ ] Questions auto-sync from 16p API on 24h schedule and stored in SQLite
- [ ] Client fetches questions from our backend only (never calls 16p API directly)
- [ ] Responsive layout works on mobile (375px) and desktop (1440px)
- [ ] Smooth question transition animations (no jank)
- [ ] Answer progress persists across page refreshes (localStorage)
- [ ] Test results can be shared via URL (`/result/:id`)

### Should Have (Deferred to Phase 6)

- [ ] Home Page (`/`) with hero section and CTA
- [ ] History page showing past test results
- [ ] Animated avatar on result page (Lottie)
- [ ] Dark mode toggle
- [ ] Swipe gesture navigation on mobile

### Nice to Have (v2)

- [ ] User authentication for persistent result tracking
- [ ] Compare results with friends
- [ ] PWA support (offline testing)
- [ ] Internationalization (i18n)

---

## 14. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| 16p API goes offline or changes structure | Medium | High | Cache all questions in DB; implement fallback scoring algorithm |
| 16p API gets cease-and-desist | Low | High | Have migration plan to self-hosted question bank (Plan A fallback) |
| MBTI trademark issues | Low | Medium | Avoid "MBTI" in branding; use "Personality Type Test"; reference Jung theory |
| Slow API response degrades UX | Medium | Medium | Load questions on app init; show skeleton during fetch; submit answers in background |
| Animation performance on low-end devices | Medium | Low | Use `will-change` and GPU-accelerated transforms; detect low FPS and reduce animations |

---

## Appendix A: 16personalities API Reference

Source: [github.com/SwapnilSoni1999/16personalities-api](https://github.com/SwapnilSoni1999/16personalities-api)

**License**: MIT

**Base URL**: `https://16personalities-api.com`

| Endpoint | Method | Description |
|---|---|---|
| `/api/personality/questions` | GET | Fetch all 120 test questions |
| `/api/personality` | POST | Submit answers, receive personality result |

**Question Format**: 7-point Likert scale (-3 to +3)

**Result Dimensions**: Mind (E/I), Energy (S/N), Nature (T/F), Tactics (J/P), Identity (A/T)

**Note**: This is an unofficial, community-maintained API. It is not affiliated with 16Personalities. The API may break if 16personalities.com changes its structure. A fallback strategy is mandatory.
