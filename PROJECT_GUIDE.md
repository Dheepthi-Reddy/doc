# Health Companion — Project Guide

What every file does, in the order data flows through the app, followed by how messaging and the AI work.

```
Browser (React pages)  →  lib/api.ts  →  Next.js /api proxy  →  FastAPI routers  →  SQLAlchemy  →  health.db
                                                                     │
                                                                     ├→ Anthropic API (AI summaries, calorie plan)
                                                                     └→ Google tokeninfo (Google sign-in)
```

---

## 1. Frontend: where a request starts

### `frontend/app/layout.tsx`
The root of every page. Loads the Plus Jakarta Sans font and `globals.css`.
Wraps every page in `AuthGate`, so nothing renders until we know who is signed in.
Sets the page title and theme colour.

### `frontend/components/AuthGate.tsx`
Reads the tab's token and asks the backend who is signed in (`GET /api/auth/me`).
Not signed in → redirects to `/login`. Patients are kept out of `/clinician`; clinicians are kept inside it.
`/profile` and `/settings` are shared by both roles.
Wraps the page in `PatientShell` or `ClinicianShell` and provides `useAuth()` (user, refresh, signOut).

### `frontend/app/login/page.tsx`
Sign in / Create account tabs. Sign in also asks "I am a Patient / Clinician", and that role is sent with every login.
Create account collects name, email, password (with a strength meter), age, height, weight, activity level and a doctor.
On success it stores the token, saves the returned calorie plan for the welcome card, and redirects to `/` or `/clinician`.
Also hosts the Google button and the "Signed in as … Continue" banner.

### `frontend/lib/api.ts`
The single doorway to the backend: prefixes `/api`, attaches `Authorization: Bearer <token>`, turns error replies into readable messages.
Stores the token per tab (sessionStorage), plus a copy in localStorage so new tabs start signed in.
On a 401 it clears the token and goes to `/login`.
`useFetch(path)` loads data for pages; the date helpers (`localDate`, `niceTime`, `ago`…) keep "today" in local time.

### `frontend/lib/types.ts`
TypeScript shapes of everything the API returns: User, Home, Energy, Streak, ChatMessage, Thread, PatientRow and more.
Keeps the frontend in step with the backend's JSON.

### `frontend/next.config.mjs`
Rewrites every `/api/*` call to the FastAPI server (`BACKEND_URL`, default `http://127.0.0.1:8000`).
So the browser only ever talks to Next.js, and no CORS setup is needed.
Also redirects the old `/food`, `/activity` and `/health` pages to `/track`.

### Shells: `PatientShell.tsx`, `ClinicianShell.tsx`, `TopBar.tsx`, `ProfileMenu.tsx`, `Backdrop.tsx`
The frame around every page: sticky header with the section links, and the profile menu on the right (profile, settings, switch account, emergency call, sign out).
The patient shell adds the reminder poller (`Notifier`) and the contact button (`ContactFab`).
The clinician shell wraps pages in the chat dock (`ClinicianChatDock`).
`Backdrop` draws the soft moving colour blobs behind the content.

### Patient pages
- **`app/page.tsx` (Today):** loads `/api/home`, `/api/medications/schedule` and `/api/habits`. Shows the starting-plan card, the streak, the status card, progress rings and tips.
- **`app/track/page.tsx` (Track):** food (live preview while typing), activity, the calorie overview with goals, the "I matched my goal" button with its checklist, blood pressure, sugar and the BMI gauge.
- **`app/medicines/page.tsx`:** today's doses (Taken / Skip), the reminders switch, adding a medicine.
- **`app/habits/page.tsx`:** habit cards with done buttons or − / + counters.
- **`app/profile/page.tsx`, `app/settings/page.tsx`:** personal and health details, my doctor, emergency contact; sound, animations, reminders, goals and emergency number. Clinicians get Available and On call switches.

### Clinician pages
- **`app/clinician/page.tsx`:** KPIs, search and filters. "Your patients" come first, then the rest of the clinic. Each card has Message and AI Summary.
- **`app/clinician/patients/[id]/page.tsx`:** one patient's 30-day records (BP, sugar, food, activity, BMI) and a Message button.

### Feature components
- **`StreakHero.tsx` + `Burst.tsx`:** the flying flame, count-up with ticks, the star burst and the 7-day dots.
- **`PlanWelcome.tsx`:** the one-time "Your starting plan" card after sign-up.
- **`DoctorPicker.tsx`:** doctor cards with Available/Away and On call, plus "Any available doctor".
- **`GoogleSignIn.tsx`:** loads Google's button when `GOOGLE_CLIENT_ID` is set; otherwise shows it disabled.
- **`GoalInput.tsx`:** a number box that saves when you leave it.
- **`SummaryBox.tsx`:** the 7/30/90-day picker and the AI Summary result for clinicians.
- **`Toast.tsx`:** small confirmation pop-ups.
- **`Notifier.tsx`:** polls for medicine reminders and shows them as browser notifications.
- **`ContactFab.tsx`, `ClinicianChatDock.tsx`, `ChatThread.tsx`:** messaging (section 3).

### Design system and helpers
- **`components/ui.tsx`:** shared building blocks (Card, Btn, Chips, Pill, Ring, Progress, Toggle, Stepper, Avatar, Skeleton, CountUp).
- **`components/icons.tsx`:** the SVG icon set.
- **`tailwind.config.js` + `app/globals.css`:** colours, shadows, animation keyframes, input styles; animations turn off when motion is reduced.
- **`lib/sound.ts`:** sounds generated in the browser (tick, chime, pop, fanfare, send).
- **`lib/prefs.ts`:** sound and animation switches, stored per device.
- **`lib/password.ts`:** password strength score.
- **`lib/emergency.ts`:** guesses the emergency number from the browser's region.
- **`public/sw.js`:** the service worker that displays notifications.

---

## 2. Backend: where the request is handled

### `backend/app/main.py`
Creates the FastAPI app and plugs in the five routers.
On startup it: creates missing tables, adds new columns to old tables, assigns old messages to a conversation, and seeds demo data (only if there are no users yet).
Also serves `GET /api/health`.

### `backend/app/database.py`
Connects to SQLite (`DATABASE_URL`, default `health.db`) and switches on foreign keys, so deleting a user deletes their rows.
`get_db()` hands each request a database session and closes it afterwards.
`add_missing_columns()` is a small migration: it adds any new nullable column to an existing table.

### `backend/app/schemas.py`
Pydantic models that check every request body before any code runs (lengths, ranges, required fields, allowed values).
Shapes every response (UserOut, MessageOut, ThreadOut…).
Lists which profile fields each role may change.

### `backend/app/security.py`
Hashes passwords (PBKDF2-SHA256 with a salt) and checks them.
Makes and verifies login tokens: signed with `SECRET_KEY`, valid for 12 hours.
`current_user`, `require_patient` and `require_clinician` guard every protected route; the wrong role gets 403.

### `backend/app/models.py`
The database tables as Python classes: users, bp, glucose, bmi, medications, medication_logs, food_entries, workouts, habits, habit_logs, checkins, messages, notification_log, summary_audit.
Patients and clinicians share `users`. `clinician_id` is a patient's doctor; `available` and `on_call` are clinician settings.
Every patient record carries `user_id`, and every query filters on it.

### Routers: the API endpoints
- **`routers/auth.py`:** login (role required and enforced), register (doctor choice + AI calorie plan + first BMI + starter habits), Google sign-in, the public doctor list, and `GET`/`PATCH /me` for profile changes, limited by role.
- **`routers/patient.py`:** home summary, check-in, BP, sugar, BMI, food (preview, add, matched-goal), energy, activity, medicines and dose logs, habits. Patient-only, own data only.
- **`routers/clinician.py`:** patient list with attention level, a patient's records, AI summary, chat threads, and reading/sending messages in the clinician's own conversation. Clinician-only.
- **`routers/messages.py`:** the patient side of chat, the rule for who a patient may message, and the backfill for old messages.
- **`routers/notifications.py`:** which medicine reminders are due now (repeat every 30 minutes), and the once-a-day login nudge.

### Logic modules: the "brains" the routers call
- **`calories.py`:** turns "2 idli and a cup of dal" into foods and calories; calories burned from MET values; BMI and category; the energy balance (eaten − burned vs goal, with tips).
- **`care_team.py`:** who is available or on call, picking the least-busy available doctor, checking a patient's doctor choice, on-call cover.
- **`calorie_plan.py`:** the starting calorie goal at sign-up (section 4).
- **`llm.py`:** clinician AI summaries with guardrails (section 4).
- **`google_auth.py`:** asks Google to verify a sign-in token and checks it was issued for this app.
- **`stats.py`:** sums and averages: day totals for food and activity, check-in days, dose adherence, period statistics for AI summaries.
- **`streaks.py`:** current streak, best streak, every 5th day a milestone.
- **`alerts.py`:** labels BP and sugar readings (Normal, Elevated, High, Crisis…) with a severity from 0 to 3.
- **`suggestions.py`:** the personal sentence used in the login nudge.

### Data and seeding
- **`app/data/*.json`:** static data kept out of code: `foods.json` (42 foods + the words the parser understands), `activities.json` (MET values), `starter_habits.json`, `demo_data.json` (demo clinicians, patients, logins, medicines, habits, each patient's story).
- **`app/data/__init__.py`:** loads those files once and caches them.
- **`app/seed.py`:** turns `demo_data.json` into 90 days of realistic history per patient (same result every time). Runs only on an empty database; `python -m app.seed --reset` rebuilds it.

### `backend/health.db`
The SQLite file holding all the data. It is ignored by git, so health data never goes into the repository.

### Tests
- **`backend/tests/`:** 67 pytest tests on a temporary database, with the AI and Google replaced by stand-ins.
- **`frontend/lib/*.test.ts`:** 10 vitest tests (dates, emergency number, password strength).

---

## 3. How messaging works

**The rule:** every conversation is private between **one patient and one doctor**. Each row in the `messages` table stores `patient_id`, `clinician_id` (whose conversation it is), `sender_id`, the text and the time.

### A doctor sends a message
1. In `ClinicianChatDock.tsx` the doctor picks a patient (from the dock list, or the Message button on the patient list or records page).
2. The browser sends `POST /api/clinician/patients/{id}/messages`.
3. `routers/clinician.py` checks the token is a clinician and saves the row with `clinician_id` = that doctor.
4. Doctors can message any patient. Each one starts their own private conversation.

### A doctor reads messages
- `GET /api/clinician/threads` lists only conversations this doctor is part of, plus their own patients (so they can start a chat).
- `GET /api/clinician/patients/{id}/messages` returns only this doctor's conversation with that patient.
- Another doctor's conversation with the same patient never appears.

### A patient reads and replies
1. `ContactFab.tsx` (the round button, bottom-right) loads `GET /api/messages`: all the patient's messages, each tagged with its doctor.
2. The chat panel shows one tab per doctor: their own doctor, on-call doctors while their doctor is away, and any doctor who has written to them.
3. A reply sends `POST /api/messages` with `clinician_id` = the open tab, so it reaches only that doctor.
4. The server refuses a doctor the patient has no connection with (403): not theirs, not on call, and never wrote to them.

### Delivery, unread counts and alerts
- There is no live connection; the app polls. Patient: every 20 s (8 s while chat is open). Doctor list: every 20 s. Open thread: every 10 s.
- **Patient unread:** the browser remembers the newest message seen from each doctor (`hc_seen_conv_<userId>` in localStorage). Newer ones count as unread and show as a red badge. A new message plays a chime and a toast "New message from …".
- **Doctor badge:** counts conversations whose last message came from the patient ("awaiting reply").
- **Doctor away:** the contact button says "Dr. X is away" and offers to message or call the on-call doctor.
- **Old messages:** messages from before chats were private were given their doctor on startup by `messages.backfill()`.

---

## 4. How the AI works

Two features use **Claude Opus 5.5** through the official `anthropic` Python SDK, called **only from the backend**.

The same safety pattern applies to both:
- The app computes the numbers itself; the AI only writes words or picks a value inside limits.
- The AI never sees names, emails, phones or anything the patient typed.
- Every answer is checked. If it fails, or there is no API key, a plain non-AI version is used, and the screen says which one you're seeing.

### A. AI summary for clinicians (`backend/app/llm.py`)
1. The clinician clicks **Summary** (7, 30 or 90 days) → `POST /api/clinician/patients/{id}/summary`.
2. `stats.py` builds numbers only: BP and sugar averages and counts, latest BMI, doses taken vs expected, food and activity averages, check-ins.
3. Claude gets those numbers with strict rules: summarise only these figures; no diagnosis, advice or risk; don't invent numbers; 4–6 short bullets, at most 150 words.
4. `validate()` rejects the answer if it is empty, uses diagnostic or advice words ("diagnose", "you should", "recommend", "risk", condition names…), or contains a number not in the input.
5. One retry. If it fails again, or no API key is set, a template summary is built from the same numbers.
6. The clinician sees the text, "Written by AI" or "Automatic (AI not used)", and a fixed disclaimer. Each request is logged in `summary_audit`.

### B. Starting calorie goal at sign-up (`backend/app/calorie_plan.py`)
1. **The app computes a safe range.** Energy needs (Mifflin-St Jeor) from weight, height, age and sex, times the activity factor (1.2 to 1.725). Then adjusted by BMI: about 15–20% less above the healthy range, 10% more below it. Kept between 1,200 and 4,000 kcal.
2. **Claude picks inside that range.** It receives only age, sex, height, weight, BMI, activity level, the estimated needs and the allowed range. It must reply as JSON `{calorie_goal, reason}` (structured output), rounded to 50, with a short friendly reason and no medical wording.
3. **The app checks the answer.** A goal outside the range is pulled back to the nearest edge. A reason with banned words, or that is too long, is rejected. Bad JSON, a refusal or a network error falls back to the formula's own goal.
4. **The result:** the goal is saved as the patient's `calorie_goal`, a first BMI reading is saved, and the Today page shows "Your starting plan" with the reason and "Suggested by AI" or "Calculated with a standard formula". The patient can change the goal at any time.

### Settings that control the AI
- `ANTHROPIC_API_KEY` (or `ANTHROPIC_AUTH_TOKEN`) turns AI on; without it, both features use their non-AI versions.
- `LLM_MODEL` changes the model (default `claude-opus-5-5`).
- In tests, the AI is replaced by a stand-in, so no real calls are made.
