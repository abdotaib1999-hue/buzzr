# ⚡ Buzzr Live — Trivia Buzzer

A real-time multiplayer trivia buzzer that runs on phones. A **Trivia Master** hosts a room, **players** buzz in from their own phones, and the app decides who was first, keeps a line of buzzers, tracks scores and crowns a champion.

It is a static site (one HTML file) that runs **100% free** on GitHub Pages + Firebase's free Spark plan.

**Live app:** https://abdotaib1999-hue.github.io/buzzr/

---

## Features

**Game**
- Two roles: **Player** (giant buzzer button) and **Trivia Master** (host controls).
- **Atomic first buzz** — the winner is decided by a Firestore transaction, so two simultaneous buzzes can never both win. Order is by arrival at the server, not by phone clocks.
- **Buzz line** — after the first buzz, other players can still tap to join the line. The host sees who is *answering* and who is *next*.
- **Wrong answer → next in line.** The host taps "WRONG — NEXT: name" and the next buzzer takes over. If nobody is left in line, the buzzer reopens for everyone who hasn't already missed.
- **3-2-1 countdown** at the start of every round, plus an optional **answer timer** (10 / 15 / 20 / 30 s). When it runs out the round shows "TIME'S UP".
- **Scoring** — host awards points (negative values allowed) to any player; live scoreboard for everyone.
- **Game over** — host crowns the winner; the champion's phone gets a trophy card with confetti and sound, everyone else sees the final standings. Ties are supported. The host can start a new game (scores reset).

**Hosting tools**
- **Invite players** — QR code + share/copy link (`?room=CODE`), so players only type their name.
- **Kick player** (✕ on the scoreboard) — removed players can't rejoin that room.
- **Close & delete room** for everyone.
- **Auto-cleanup** — whenever a host creates a room, their own rooms older than 24 hours are deleted.
- Players who tap **Leave** remove themselves from the scoreboard.

**Phone experience**
- Instant local sound on tap (Web Audio, no files) and vibration on Android.
- Buzzer fires on `pointerdown` (no waiting for finger lift).
- Screen stays awake while in a room (Wake Lock), "Offline — reconnecting…" indicator, no pull-to-refresh bounce, `dvh` height and safe-area padding.
- Refresh-proof: name, role and room are kept in the URL, so reloading rejoins automatically.
- Installable (manifest, icons, service worker): Android → Install app; iPhone → Share → Add to Home Screen.

---

## Files

| File | Purpose |
|---|---|
| `index.html` | The whole app (HTML, CSS, JS) |
| `manifest.json` | Install info (name, icons, colors) |
| `sw.js` | Minimal service worker so Android treats the app as installable |
| `icon-192.png`, `icon-512.png`, `icon-180.png` | App icons (Android / iOS) |
| `firestore.rules` | Firestore security rules — paste into the Firebase console |
| `README.md` | This file |

**Tech:** HTML + Tailwind (CDN), Firebase JS SDK 11.6.1 (Firestore + Anonymous Auth), canvas-confetti 1.9.3, qrcode-generator 1.4.4, Outfit font (Google Fonts).

---

## Setup (free)

### 1. Firebase — https://console.firebase.google.com/
1. **Add project** (turn Google Analytics off) and stay on the **Spark** plan.
2. **Build → Authentication → Get started → Sign-in method → Anonymous → Enable → Save.**
3. **Firestore Database → Create database** → *Standard edition* → pick the closest location (it cannot be changed later) → **Production mode** → Create.
4. Firestore **Rules** tab → delete everything, paste the contents of `firestore.rules` → **Publish**.
5. ⚙ → **Project settings → General → Your apps → `</>`** → register a web app (leave Firebase Hosting unchecked) → copy the 6 values of `firebaseConfig`.

### 2. Put your keys in the app
Open `index.html`, find `manualFirebaseConfig`, and replace each `PASTE_YOUR_...` value (keep the quotes).

### 3. GitHub Pages
1. Create a **public** repo (e.g. `buzzr`) and upload all the files to the root.
2. **Settings → Pages → Deploy from a branch → `main` / `(root)` → Save.**
3. After 1–2 minutes the app is live at `https://YOUR-USERNAME.github.io/REPO-NAME/`.

If you see an *unauthorized domain* error: Firebase → Authentication → Settings → Authorized domains → add `YOUR-USERNAME.github.io`.

---

## How to play

**Host (Trivia Master)**
1. Enter a name, choose **Trivia Master**, type a room code, tap **CREATE / HOST ROOM**.
2. Tap **Invite players** and show the QR code or share the link.
3. Pick a timer, then tap **OPEN BUZZER** — a 3-2-1 countdown starts, then the buzzer goes live.
4. When someone buzzes, read their answer:
   - Correct → use **Award Round Points**, then **NEXT ROUND**.
   - Wrong → tap **WRONG — NEXT** to pass it to the next buzzer in line.
5. When done, tap **Finish All Rounds & Crown Winner**.

**Players**
1. Open the invite link (or scan the QR), enter your name, tap **JOIN ROOM**.
2. Wait for the countdown, then tap the big red **BUZZ!** button.
3. If someone else is already answering, you can still tap to join the line.

---

## Data model

Each room is one Firestore document:

`artifacts/{appId}/public/data/rooms/{roomId}`

| Field | Meaning |
|---|---|
| `hostId` | UID of the Trivia Master who created the room |
| `status` | `waiting` · `countdown` · `active` · `locked` · `timeup` · `gameover` |
| `roundNum` | Current round number |
| `winner` | `{ userId, name }` — the player currently answering |
| `buzzQueue` | Ordered list of buzzers (index 0 = answering) |
| `missed` | UIDs who answered wrong this question |
| `kicked` | UIDs removed by the host |
| `timerSecs`, `opened` | Answer timer length, and a nonce for each opening |
| `players` | Map `uid → { userId, name, role, score, joinedAt }` |
| `createdAt` | Server timestamp (used for cleanup) |

---

## Security

- The Firebase web API key in `index.html` is **not a secret** — it only identifies your project. GitHub's "secret detected" email about it can be dismissed. Your data is protected by `firestore.rules`, so make sure they are published.
- `firestore.rules` enforces: only the host controls the room; a player can only add/rename/remove **their own** entry (never set a score or become host); only one first buzz per open round; kicked players and players who missed can't buzz; hosts can list/delete only their own rooms.
- Optional hardening: Google Cloud Console → APIs & Services → Credentials → the key "(auto created by Firebase)" → Application restrictions → **Websites** → add `https://YOUR-USERNAME.github.io/*`.
- Never commit a Firebase **service-account** JSON or private key.

---

## Free-tier limits

- **GitHub Pages** (free plan): the repo must be public.
- **Firestore (Spark):** 50,000 reads, 20,000 writes and 20,000 deletes per day, 1 GiB storage; the daily quota resets around midnight Pacific. Each update counts as one read per listening phone. Rough estimate: a 20-player, 15-round game ≈ 3,000 reads, so about 15 full games a day.
- **Auth (Spark):** 3,000 daily active users and 100 new accounts per hour per IP address.

---

## Known limitations

- Closing a tab (instead of tapping **Leave**) leaves that player on the scoreboard until the host kicks them.
- If the host clears their browser data they lose control of that room — start a new room code.
- The countdown and answer timer are driven by the **host's phone** — keep the host's screen on and the app open.
- The promoted answerer (after "WRONG — NEXT") is not timed; the timer only runs while the buzzer is open.
- iPhone Safari has no Vibration API, so iPhones get sound but no vibration.
- Sync speed depends on each player's connection.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Status stuck on "Connecting" / "Connection Error" | Anonymous sign-in not enabled, or a config value is wrong |
| "Failed to connect to room" | Firestore database not created, or rules not published |
| "Room Not Found" when joining | Wrong code — only the host creates rooms |
| A button does nothing | Probably a rules rejection — open the browser console and read the error |
| Unauthorized-domain error | Add your `github.io` domain under Authentication → Settings → Authorized domains |

## Ideas for later

- **Question mode:** host pastes AI-generated questions (JSON) and each round shows the question on every phone, with the answer visible only to the host.
- Optional ready-made questions from the free Open Trivia DB API.
- Optional "Generate with AI" using the host's own free Gemini API key (stored only in their browser).
- Timer for the promoted answerer; presence tracking to remove closed-tab ghosts automatically.
