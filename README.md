# Echo Schedule

A team shift-scheduling web app built for Echo (a bar near Rotterdam Centraal). Single-file HTML/CSS/JavaScript, no build step, hosted for free on GitHub Pages, with Firebase Firestore as the real-time shared database.

**Live app:** https://echo687.github.io

## What it does

- **Fill in** — team members set their availability (Available/Unavailable) per day, plus an optional short note (e.g. "can work till 1am").
- **My Schedule** — team members see the shifts they've actually been assigned, by week/month/year.
- **Hours** — after a shift, team members log the hours they actually worked (native time pickers), separate from what was scheduled.
- **Owner** (password-protected) — assign shifts, see the full team overview table, mark days as Closed, manage the team list, export logged hours as CSV, download/print the schedule as a PDF, undo the last change, and view a recent-activity log.

Data syncs in real time across everyone's phones via Firebase Firestore — no polling, no rate limits.

## Customizing

Everything lives in one file: `index.html`. All the settings you're likely to change are grouped in one place, inside a `CONFIG` object — search the file for these names:

| Setting | What it does |
|---|---|
| `SITE_PASSWORD` | Password to open the app. **Also appears in a small separate script near the top of the file** — if you change it, update both copies, or the lock screen and the "switch name" flow will disagree. |
| `OWNER_PASSWORD` | Password for the Owner tab. |
| `DEFAULT_STAFF` | Starting team member list (only used before the team list is first saved). |
| `NIGHT_OPTIONS` / `DAY_OPTIONS` | The quick-assign shift time presets shown in the Owner tab's assign dialog. |
| `FIREBASE` | Your Firebase project config (see setup below). |

Line numbers aren't listed here on purpose — they shift every time the file is edited. Searching for the setting name is reliable; a specific line number won't stay accurate for long.

## Setup (from scratch)

1. **Firebase (data storage):**
   - Create a free project at [console.firebase.google.com](https://console.firebase.google.com)
   - Add a Web app (`</>` icon) to get your config values
   - Enable **Firestore Database** (Build → Firestore Database → Create database)
   - Set permanent security rules restricting access to just this app's document:
     ```
     rules_version = '2';
     service cloud.firestore {
       match /databases/{database}/documents {
         match /rooster/state {
           allow read, write: if true;
         }
       }
     }
     ```
   - Paste your config values into the `FIREBASE` object in `index.html`

2. **Hosting (GitHub Pages):**
   - Create a public GitHub repository
   - Upload `index.html` (must be named exactly that, lowercase)
   - Settings → Pages → your site goes live at `https://<username>.github.io/<repo>/`
   - For a shorter link with no username in it, name the repo `<username>.github.io` exactly — GitHub then serves it at the account root

3. **First run:**
   - Open the site, unlock with `SITE_PASSWORD`
   - Add your team via Owner → Manage team
   - Change both default passwords before sharing the link with your team

## Architecture notes

- **No build step, no dependencies to install.** The whole app is one HTML file. Firebase's SDK is loaded via a CDN `import` in a `<script type="module">`.
- **The password lock screen runs in a separate, plain `<script>`** placed *before* the module script, so it works instantly even if the Firebase SDK is slow to load over the network.
- **Real-time sync** uses Firestore's `onSnapshot` listener — changes from other people appear automatically, no polling.
- **Local-edit protection:** while you're actively editing something, incoming remote updates are held back (`AppState.isTyping` / `AppState.dirty`) so a slow network response can't overwrite what you just typed. `AppState.locallyTouched` tracks exactly which fields you edited this session so a merge always prefers your own recent edit over a stale remote value for those specific fields.
- **Offline fallback:** if Firestore is unreachable, the app loads from a local backup copy (`localStorage`) so it's still usable, and clearly flags that it's showing offline/cached data.
- **Dates are handled in local time throughout** (`fmtISO`/`parseISODateLocal`), not `toISOString()`/UTC — this matters for anyone in a timezone ahead of UTC, where UTC-based date formatting silently shifts every date back by one day.

## Known limitations

- Passwords are plain shared strings checked in client-side JavaScript, suitable for a small trusted team, not real access control. Anyone who can view the page source can read them.
- The Firebase `apiKey` in `index.html` is not a secret — Firebase web apps are designed to have this key be public. Actual protection comes from the Firestore security rules above, not from hiding this value.
- No automated tests. Changes should be tested manually across the four tabs (Fill in, My Schedule, Hours, Owner) before relying on them.
