# Frontend Audit and Feature-to-API Mapping

## Scope and status

This is a source-code audit of the frontend routes, visible controls, data sources, and network/database operations. It is not a click-by-click browser QA report. The Express backend has been removed; the Vite frontend build succeeds, but API calls still present in the frontend do not have a server implementation.

“Works” below means the control has a client-side handler or a configured external Firebase/Cloudinary operation. Firebase operations still require valid Firebase configuration and permissive, correctly scoped Firebase security rules.

## Pages and route inventory

Routes are registered in `src/App.jsx`.

| Page / route | Main controls and current status | Data / important limitations |
|---|---|---|
| Home `/` | Search, Explore, Post Work, and signup navigation work. Project cards open the client-side detail/hire modal. | Static project cards, freelancer profiles, category tiles, and headline statistics. |
| Explore `/explore` | Text search, sort, AI toggle, and project detail opening work locally. AI Search calls a missing API. Notification control has no handler. Like, comment, and message actions in project detail depend on removed API/Firebase. | Loads `staticProjects`; attempts a missing projects API first and falls back to static data. |
| Browse Jobs `/jobs` | Search, sort, pagination, and job-detail navigation work locally. | Hard-coded job list from `src/lib/jobData.js`. |
| Job Details `/jobs/:id` | Back navigation works. Follow and Apply controls are display-only. | Hard-coded job record; unknown IDs silently display the first job. |
| Services `/services` | Search, category/price filters, clear filters, and filter-modal controls work locally. Create Service has no handler. | Static services are returned when the removed services API fails. |
| Skill Battles `/battles` | Filters and cards have UI handlers, but listing depends on a missing battle API; the page displays an API error/empty state without it. | No local battle seed data found. |
| Battle Detail `/battles/:id` | Back navigation works. Vote and vote-state checks call missing APIs; detail cannot load without the battle API. | No local battle data fallback. |
| Create Battle `/battles/create` | Form fields, image previews, validation, and navigation code exist. Submit calls a missing API, so creation cannot complete. | Images are held as browser data URLs; no durable battle storage. |
| Messages `/messages` | Search, conversation selection, typing indicator, read receipts, and sending messages are wired directly to Firebase Realtime Database. Header phone/video buttons have no handlers. | Requires signed-in user, populated RTDB conversations, and database rules. No `initialConversations` seed is used by this page. |
| Post Work `/post-work` | Wizard, fields, price/skill/delivery selections, local preview, and Cloudinary upload are implemented. Publish calls the removed projects API and therefore cannot persist. | Upload goes directly to Cloudinary; project record is not saved after backend removal. |
| Edit Listing `/edit/:type/:id` | Form editing, file selection/preview, and Cloudinary upload work in the browser. Save calls missing update APIs. | Existing listings are only loaded from static fallback data or missing API responses; static entries are not editable records. |
| Project Matches `/project/:id/matches` | Live project/match listeners and user lookup read Firestore directly. Recalculate calls a missing scoring API. | Requires Firestore data/rules. The matching/scoring engine is not client-side. |
| Profile `/profile`, `/profile/:uid` | Tabs and form/modal UI work. Profile loading and profile/avatar/banner/experience edits use missing `/users/:uid` API calls; message/profile navigation may work separately. | Own profile can fall back to Firebase Auth/localStorage user data; other profile details are not reliably available without API/Firestore migration. |
| Dashboard `/dashboard` | New Post and navigation links work. Delete calls missing APIs; Withdraw has no handler. Matches links reach the matches page but that page needs Firestore and the missing scoring API. | Listings use API fetch with static fallback. Earnings, satisfaction, and message metrics are hard-coded; listings are only real if returned by the API. |
| Support `/support` | FAQ accordion works. Contact fields accept input; Send Message is inert (`type="button"` with no handler). | FAQ text is embedded in the page. No support-ticket submission exists. |
| How It Works `/how-it-works` | Client/Freelancer tabs, FAQ expand/collapse, and navigation CTAs work locally. | Static explanatory content; claims about hiring, escrow, messaging, and publishing are not implemented end-to-end. |
| Login `/login` | Email/password submit, Google sign-in, and signup navigation use Firebase Auth. | Requires valid Firebase config and enabled providers. Password recovery control, if displayed, has no reset flow. |
| Signup `/signup` | Email/password signup and Google sign-in use Firebase Auth. Role selector only changes local UI state; the selected role is not persisted by signup. | Auth profile sync writes a basic user document to Firestore. |

## Button and interaction status by shared component

| Component | Working controls | UI-only or dependent controls |
|---|---|---|
| Navbar, Sidebar, MobileTabBar | Route navigation, menu open/close, collapse, login/logout. Logout uses Firebase Auth. | Any route not registered in `App.jsx` leads to an unmatched route. |
| ProjectCard / HireModal | Opening a project card and closing the modal work locally. Edit/Matches links navigate. | “Save for Later” and “Send Message” in HireModal have no click handlers. Card ratings/badges are deterministic visual values, not verified review data. |
| ProjectDetailView | Close, message-conversation setup (RTDB), and media display have handlers. | Like and comment use missing APIs. Message setup requires Firebase RTDB access. |
| BattleCard | Opens battle detail through its supplied callback. | Vote data shown is only as real as the unavailable battle API response. |
| Footer | Internal links navigate where routes exist; social links open external URLs. | Collections, About, Blog, Careers, Privacy, Terms, and Cookies routes are not registered in `App.jsx`; those links have no matching page. |

## Source-level issues found

- **Messages conversation shape mismatch:** project detail creates `participants` as an object keyed by UID, and the conversation list supports that shape. The active chat header calls `activeConv.participants.find(...)`, which assumes an array; opening a conversation created from a project can throw at render time. Normalize participants to one documented shape or handle both.
- **Explore price sorting labels are reversed:** “Price ↑” sorts high-to-low and “Price ↓” sorts low-to-high. Browse Jobs uses the opposite (ascending/descending) behavior for those labels.
- **Service category mismatch:** seeded service records use values such as `Tech And Development`, while the filter category is `Tech & Development`; exact category filtering can hide matching seeded services.
- **No catch-all route:** unregistered routes render no not-found page, including several Footer links listed above.
- **Stale development proxy:** `vite.config.js` still proxies `/api` to `localhost:5000`, although the backend was removed. API-dependent screens can show proxy/network errors during development.
- **Success claims are not supported by persistence:** several UI labels and informational pages describe real-time, escrow, ratings, and marketplace activity, while related content is hard-coded or no longer has a backend.

## APIs called by the frontend but missing after backend removal

There is no Express server or API route implementation in this repository now. These are required contracts inferred from the frontend call sites; **none should be treated as currently available**.

| Feature | Required backend logic | API contract currently expected / recommended |
|---|---|---|
| Discover projects | Fetch persisted, publishable projects | `GET /api/projects` |
| Publish project | Validate owner, persist project and media metadata | `POST /api/projects` |
| Edit/delete project | Verify ownership and mutate/remove project | `PUT /api/projects/:id`, `DELETE /api/projects/:id` |
| Search projects with AI | Securely call AI provider; rank project results | `POST /api/search/smart` |
| Like project | Authenticate user; prevent duplicate likes; return count/state | `POST /api/projects/:id/like` |
| Project comments | Fetch and persist validated comments | `GET /api/projects/:id/comments`, `POST /api/projects/:id/comments` |
| Discover/create/edit/delete services | Fetch and persist service listings with owner checks | `GET /api/services`, `POST /api/services`, `PUT /api/services/:id`, `DELETE /api/services/:id` |
| Public profile and profile edits | Fetch profile; validate and persist profile fields/media | `GET /api/users/:uid`, `PUT /api/users/:uid` |
| Skill battles | Filter/list battles; fetch one battle; create battle; record unique vote and return counts/vote state | `GET /api/battles`, `GET /api/battles/:id`, `POST /api/battles`, `GET /api/battles/:id/user-vote`, `POST /api/battles/:id/vote` |
| Project matching | Securely score eligible freelancers and write ranked results | `POST /api/matches/score-all` |
| Jobs and applications | Fetch real jobs and details; validate eligibility and persist applications | `GET /api/jobs`, `GET /api/jobs/:id`, `POST /api/jobs/:id/applications` |
| Hire / contact flow | Create or retrieve conversation and send an inquiry/message | `POST /api/conversations`, `POST /api/conversations/:id/messages` (or retain secured RTDB implementation) |
| Dashboard analytics | Aggregate earnings, listings, satisfaction, and unread messages for the authenticated user | `GET /api/dashboard` |
| Withdraw / payments | Validate balance, initiate payment-provider transfer, record status and audit trail | `POST /api/withdrawals` |
| Support contact form | Validate and store ticket; notify support; return ticket status | `POST /api/support/tickets` |

The exact URI naming can be changed, but frontend and server contracts must be kept consistent. All writes and privileged operations need authentication, authorization/ownership checks, validation, abuse controls, and server-side secrets. Do not put AI or payment-provider secrets in `VITE_*` variables.

## Firebase and external service calls made directly by frontend

| Operation | Client-side implementation | Notes |
|---|---|---|
| Email/password login and signup | Firebase Auth SDK in `src/lib/auth.js` | No custom Express auth endpoint. |
| Google login | Firebase Auth popup provider | Requires provider enabled in Firebase Console and allowed domains configured. |
| Auth profile sync | Firestore `setDoc(users/{uid})` from `src/lib/auth.js` | Writes a limited user document from the browser; Firestore rules must restrict writes to that user. Signup role is not included. |
| Realtime messaging | Firebase RTDB reads/writes for `conversations`, `messages`, `typing`, and `readReceipts` in `src/pages/Messages.jsx` | Includes message send, typing status and read receipt writes. Rules must restrict conversation participants. |
| Start conversation from project | RTDB `get`/`update` in `src/components/ProjectDetailView.jsx` | Direct client write; no server-side conversation validation. |
| Match display | Firestore listeners/reads of `projects`, `users`, and `matches/{projectId}/results` in `src/pages/ProjectMatchPage.jsx` | Scoring itself still requires the missing API. |
| Media upload | Direct unsigned Cloudinary upload in `src/lib/cloudinary.js` | Uses Vite-exposed cloud name and upload preset. Presets must be constrained; a browser variable is not a secret. |
| Analytics | Firebase Analytics SDK in `src/lib/firebase.js` | Client-side telemetry; not app/business data persistence. |

No Firestore client operations were found for projects, services, likes, comments, profile editing, battles, or job applications. Those currently depend on Express APIs or local/static UI data.

## Dummy, seeded, and fallback data

- `src/lib/data.js`: a fixed project catalogue with generated avatars, preset like counts, and `source: "static"`. `fetchProjects()` attempts the removed API, then returns this catalogue.
- `src/lib/serviceData.js`: six hard-coded services. `fetchServices()` attempts the removed API, then returns these entries.
- `src/lib/jobData.js`: six hard-coded jobs used by job list/detail pages.
- `src/pages/Home.jsx`: hard-coded freelancer profiles, service categories, and headline statistics. Project cards also use the static project catalogue.
- `src/lib/messageData.js`: sample conversations exist in source, but the current Messages page reads RTDB and does not import this seed data.
- `src/pages/Dashboard.jsx`: earnings, satisfaction, and message count are hard-coded. Listing count is calculated from fetched/fallback arrays.
- `src/components/ProjectCard.jsx` and `src/components/HireModal.jsx`: generated rating/badges and fixed rating/delivery/satisfaction text are presentation data, not verified marketplace metrics.
- `src/pages/JobDetails.jsx`: hard-coded budget/detail values and a first-job fallback for an unknown route ID.
- Battles have no client-side seed fallback; the UI expects API responses.

## What works without the removed Express backend?

### Works as frontend-only interactions

- Navigation, responsive menus, animations, form field editing, local sorting/filtering, pagination, FAQ accordions, modal opening/closing, and client-side validation.
- Browsing the seeded project/service/job catalogues.
- Cloudinary media upload if its account, preset, network access, and file settings are valid.
- Login/signup/logout through Firebase Auth if Firebase is configured.
- Messaging and starting conversations through Firebase RTDB if data and security rules are configured.
- Reading match/project/user data directly from Firestore if the relevant data and rules exist.

### Requires the missing custom backend (or a deliberate Firebase migration)

- Persisting/editing/deleting projects and services; real project/service discovery.
- AI search and match scoring.
- Likes/comments and server-validated battle listing, creation, and voting.
- Profile read/update operations currently using `/api/users`.
- Real job data, job application submission, support ticket handling, hiring/workflow/payment/withdrawal.
- Real dashboard financial, satisfaction, and messaging aggregates.

The pages may render when API calls fail, but that does not mean the corresponding write/action works. Static fallbacks are read-only sample content unless new persistence is implemented.

## Recommended implementation order

1. Decide the data authority: Firebase client SDK with strict Firestore/RTDB rules, or a rebuilt authenticated API. Avoid mixing two write paths for the same entity.
2. Restore core project and service persistence/listing; wire create, edit, delete, owner checks, and profile attribution.
3. Implement profile reads/updates and real dashboard aggregates.
4. Implement job data/application flow and support-ticket submission.
5. Implement likes/comments, conversation creation from hiring, battles/voting, and AI search/match scoring.
6. Add loading/error/empty states for every operation and test authenticated/unauthenticated, owner/non-owner, and denied-rule cases.
7. Either remove the stale Vite `/api` proxy (`vite.config.js`) or point it to the API host when a backend is restored.
