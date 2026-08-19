# StudyBuddy

An AI-powered flashcard generator with real spaced repetition. Paste your notes (or upload a `.txt`/`.md` file), and Gemini turns them into question-and-answer flashcards. Review them in an interactive 3D card interface, self-rate how well you knew each one, and a proper SM-2 algorithm schedules when you'll see it again — so you spend less time on what you already know, and more on what you don't.

**Live demo:** https://studybuddy-flashcards.netlify.app/

## Features

- **AI flashcard generation** from pasted notes or an uploaded `.txt`/`.md` file, via the Gemini API using structured JSON output (not just prompted text)
- **Interactive 3D review UI** — a genuinely three-dimensional flip card (CSS 3D transforms + perspective), with a cursor-tracking tilt and specular highlight, and an explicit "Reveal Answer" button
- **Real SM-2 spaced repetition** — every self-rating (Again / Hard / Good / Easy) runs the actual SM-2 formula to compute the next review interval and update the card's ease factor
- **Difficulty-aware session ordering** — cards you struggle with resurface again within the same session; cards you know well drop out until their real due date
- **Local persistence** — every generated deck, plus each card's live SM-2 state, is saved to the browser's `localStorage`. Past decks are listed and can be resumed (only showing cards actually due) without calling the API again
- **"Review Again"** — a manual option to replay a full deck outside of the automatic due-date schedule

## Tech stack

- Plain HTML, CSS, and JavaScript — no frameworks, no build tools
- [Gemini API](https://ai.google.dev/gemini-api/docs) (`gemini-3.5-flash-lite`) for flashcard generation, using `responseSchema` for structured JSON output
- [Three.js](https://threejs.org/) with a custom GLSL fragment shader for the animated background
- Browser `localStorage` for all persistence — no external database
- Browser `FileReader` API for `.txt`/`.md` file uploads

## Getting started

### 1. Get a Gemini API key
1. Go to [Google AI Studio](https://aistudio.google.com/apikey)
2. Sign in and click **Create API key**
3. Copy the key

### 2. Run the app
- **Option A (locally):** download `studybuddy.html` and open it directly in any modern browser (a real browser tab — some sandboxed preview environments block the outbound network calls this app needs).
- **Option B (hosted):** visit the [live demo](https://studybuddy-flashcards.netlify.app/).

### 3. Use it
1. On the **Create** tab, paste your Gemini API key and click **Save**.
2. Paste your notes into the text box, or click **"Upload a .txt or .md file instead"**.
3. Click **Generate Flashcards** — you'll be taken straight to the **Review** tab with your new deck.
4. Read the question, click **Reveal Answer** (or click the card itself), then rate how well you knew it.
5. Keep going until you see "You're all caught up!" — or click **Review deck again** to replay the whole deck voluntarily.
6. Past decks appear under **Previously generated decks** on the Create tab — click one to resume it, showing only the cards currently due.

## Notes and limitations

- **API key storage:** the Gemini API key is stored in browser `localStorage` and used directly in client-side requests. This is appropriate for a personal, single-user tool — see `ARCHITECTURE.md` for the full trade-off discussion and what a multi-user deployment would need instead.
- **File upload** currently supports plain text formats (`.txt`, `.md`) only, capped at 2MB. PDF/DOCX would require a dedicated parsing library and isn't implemented yet.
- **Gemini free-tier rate limits** apply and reset daily (midnight Pacific Time), not per-minute. If generation fails with a quota error, this is the usual cause.
- **3D background performance** depends on the device's GPU; it should run smoothly on most modern laptops/phones but may be heavier on older integrated graphics.

## Project structure

```
studybuddy.html   # entire application: markup, styles, and logic
README.md         # this file
ARCHITECTURE.md    # design decisions, challenges, and trade-offs
```
