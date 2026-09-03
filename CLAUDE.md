# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file static web app (`index.html`) for a Thai high-school class (ม.4/8, โรงเรียนสรรพวิทยาคม) with two features behind a tab switcher:

- **กระดานของหาย (Lost & Found board)** — public, no login required. Anyone can post a lost/found item report with an optional photo, and mark it resolved.
- **เช็คชื่อยืนยันตัวตน (Attendance check-in)** — gated behind Google Sign-In restricted to the school email domain. Students take a live webcam photo (or file a self-service leave request with a reason) to mark themselves present for the day; admins review the roster, override statuses, and export history.

There is no build system, package manager, or test suite — it's one HTML file with inline `<style>` and `<script>` blocks, deployed as-is.

## Commands

There is no build/lint/test tooling in this repo. The only useful sanity check when editing the inline `<script>` block is verifying it's still syntactically valid JS, since a typo silently breaks the whole page:

```bash
sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' > /tmp/check.js && node --check /tmp/check.js
```

Also worth checking that `<style>...</style>` braces balance after a CSS edit (a stray `{`/`}` won't error, it'll just silently mis-style the page):

```bash
python3 -c "
import re
style = re.search(r'<style>(.*)</style>', open('index.html').read(), re.S).group(1)
print(style.count('{'), style.count('}'))
"
```

There's no local dev server needed — `index.html` can be opened directly, though Google Sign-In / Firestore reads will behave the same as production since they hit the live Firebase project either way.

## Deployment

Push to `main` → GitHub Pages auto-builds and deploys (workflow "pages build and deployment", runs in ~30–60s). Live at:

**https://jakapongdiwaa-hash.github.io/pun-chakapong/**

There is no staging environment — every push to `main` goes live immediately to the students using this.

## Architecture

### Backend: Firebase

Everything is backed by a single Firebase project (config is hardcoded in `index.html`, project id `pun-chakapong248`), using the `firebase-app/auth/firestore-compat` v10 SDKs loaded from a CDN (no npm). Three Firestore collections:

- **`reports`** — lost & found posts. Public read/write per Firestore rules (no auth required by the UI). Fields: `name, itemName, status ("ของหาย"|"เจอของ"), location, time, contact, imageUrl, resolved, createdAt`.
- **`students`** — the class roster. Fields: `name, email, no (seat number, stored as string), studentCode, classRoom`. **This collection is meant to grow organically from self-registration** (see below) — there is no seed/import step that runs automatically anymore.
- **`attendance`** — one doc per student per day, doc ID is `${date}_${studentId}` (date as `YYYY-MM-DD`). Fields: `studentId, studentName, date, status ("ส่งแล้ว"|"ลา"|"ขาด"|"ยังไม่ส่ง"), photoUrl?, reason?, submittedAt`. Keying by date means every day's attendance is independent and browsable via the date picker — nothing resets or gets deleted automatically.

Firestore security rules are wide open (`allow read, write: if true`) for all three collections — there is no server-side enforcement of the admin password or student identity; everything is a client-side gate only.

Photos (both lost & found posts and attendance check-ins) are uploaded to **imgbb** (an external image host, API key hardcoded as `IMGBB_API_KEY`) rather than Firebase Storage — `uploadImageToImgbb()` is the shared helper for both features.

### Auth and the roster identity problem

Login uses `signInWithPopup` (not redirect — redirect was tried first and failed silently on GitHub Pages due to cross-origin storage issues between the `github.io` origin and the `firebaseapp.com` authDomain). Only emails ending in `SCHOOL_EMAIL_DOMAIN` (`@sappha.ac.th`) are accepted; anything else is signed out immediately.

On login, the app looks up the `students` collection by exact `email` match. If found, the student goes straight to check-in; if not, they see a self-registration form (เลขที่ + ชื่อ-นามสกุล) that creates their `students` doc with their **real** login email. This replaced an earlier design that pre-seeded 40 students with a *guessed* `studentCode@domain` email — that guess didn't always match a student's real school email, silently routing them into self-registration and creating duplicate rows for the same person. There is no seeding step anymore for this reason; the roster is meant to be built up entirely by students logging in for the first time.

Because there's no server-side identity check, an admin "แก้ไข" (edit) action lets a name/email be corrected directly if a duplicate or mismatch happens anyway, and a "ลบรายชื่อ" / "ลบรายชื่อนักเรียนทั้งหมด" pair of actions exist for cleaning up one row or wiping the whole roster (double-confirmed).

### Client-side admin gate

"โหมดแอดมิน" is a single hardcoded password (`ADMIN_PASSWORD`) checked in JS — it toggles `isAdmin` and shows/hides admin-only buttons (mark leave/absent/pending, edit, delete, bulk import, CSV export). This is UI-only; it is not backed by Firestore rules or Firebase Auth roles, so treat it as convenience, not real access control.

### Attendance capture flow

The webcam capture (`#cameraModal`) uses `getUserMedia` + a `<canvas>` frame grab (not a file `<input capture>`, which only forces a live camera on mobile and just opens a normal file picker on desktop) — shoot → preview → retake/confirm → upload. The "แจ้งลาวันนี้แทน" link is the alternate path: a text reason instead of a photo, writing `status: "ลา"` + `reason` directly.

### CSV export

The admin "ส่งออกประวัติการเช็คชื่อ" feature queries `attendance` by a `date` range for a chosen month and builds a CSV client-side (with a UTF-8 BOM so Thai text renders correctly in Excel) — no server/Cloud Function involved.
