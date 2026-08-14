# FoodLog AI v2 — Apps Script + Google Sheets Edition

A conversational calorie tracker that treats Google Sheets as its database. Ask a question about what you ate, get a receipt-style calorie + macro estimate back, and browse the results in a dashboard with charts — all served from a Google Apps Script web app.

## What changed from v1

The original was a static HTML/CSS/JS page using `localStorage`. This version is a **full overhaul**:

| | v1 | v2 |
|---|---|---|
| Data storage | Browser `localStorage` (device-only) | **Google Sheet** (durable, inspectable, shareable) |
| Backend | None — everything ran in the browser | **Google Apps Script** (`doGet`, `google.script.run`) |
| Estimates | Calories only | Calories **+ protein/carbs/fat** |
| Views | Single scrolling page | **Log tab** (chat + history) and **Insights tab** (dashboard) |
| Analytics | Today's total only | 7-day trend chart, macro donut chart, streak, average daily calories, most-logged food |
| AI path | Local table only | Local table by default, with an **optional real AI (Gemini) hook** that runs server-side so no key is ever exposed to the browser |

## Concept

**FoodLog AI** is a chat-first calorie tracker: instead of a form with dropdowns and search boxes, you just type what you ate in plain English. It parses quantities and food names, estimates calories and macros from a nutrition table, and logs the result to a Google Sheet that acts as your personal food diary. The Insights tab turns that diary into a small dashboard — enough to demonstrate parsing, persistence, and data visualization in one project.

## Project structure

```
Code.gs          Apps Script backend — food table, NLU parser, Sheet read/write, dashboard aggregation
Index.html       Page shell — header, tab switcher, Log panel, Insights panel
Stylesheet.html  Design system (light + dark mode), included into Index.html
JavaScript.html  Client logic — google.script.run calls, chat rendering, Chart.js dashboards
README.md        This file
```

Apps Script splits HTML into separate files by convention (`Stylesheet.html`, `JavaScript.html`) and stitches them together with `include()` — that's why CSS/JS live in their own `.html` files instead of `.css`/`.js`.

## Deploying it

1. **Create a Google Sheet** (blank, any name — e.g. "FoodLog AI Data").
2. In the Sheet, go to **Extensions → Apps Script**. This opens a script bound to that Sheet.
3. Delete the default `Code.gs` contents and paste in this project's `Code.gs`.
4. Create three HTML files in the script editor (**File → New → HTML file**), named exactly `Index`, `Stylesheet`, and `JavaScript` (Apps Script adds the `.html` extension automatically) — paste in the matching file's contents.
5. Click **Deploy → New deployment**.
   - Select type: **Web app**.
   - Execute as: **Me**.
   - Who has access: **Anyone** (or **Anyone with Google account**, or **Only myself** — pick based on whether classmates/instructors need to open it directly).
   - Click **Deploy**, authorize the requested permissions (it needs to read/write the Sheet).
6. Open the web app URL it gives you. The first request auto-creates a `FoodLog` sheet tab with headers — nothing to set up manually.

That's the whole deployment — no separate hosting, no API keys required to run.

## How the data model works

Every logged food item becomes one row in the `FoodLog` sheet: timestamp, date (ISO + display label), food name, emoji, quantity label, calorie range, protein/carbs/fat in grams, and whether it was recognized by the food table. `getDashboardData()` in `Code.gs` reads the whole sheet on each request and aggregates it into the log table, today's total, the 7-day trend, today's macro split, and the quick-stat cards — so the Sheet is the single source of truth and the dashboard is just a view over it.

Because the "AI" logic runs in Apps Script rather than the browser, you can open the Sheet directly to see (and grade) the raw data behind every chart.

## Estimating calories: local table vs. real AI

By default, `estimateClause_()` in `Code.gs` matches food names against a ~75-item local table (calories + macros per serving) and scales by the parsed quantity — no external API, so it works immediately with zero configuration.

If you want to demonstrate a real AI call for extra credit, Apps Script is actually a good place to do it safely: `UrlFetchApp` runs server-side, so an API key stored in **Project Settings → Script Properties** never reaches the browser. To wire it up:

1. Get a Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey).
2. In the Apps Script editor: **Project Settings → Script Properties → Add script property** — key `GEMINI_API_KEY`, value your key.
3. Add a function like this to `Code.gs` and call it from `submitFoodEntry()` in place of (or as a fallback chain before) the local parser:

```javascript
function callGeminiAI_(text) {
  const key = PropertiesService.getScriptProperties().getProperty("GEMINI_API_KEY");
  if (!key) return null; // no key configured — caller should fall back to the local table

  const prompt = `Estimate calories and macros (protein/carbs/fat in grams) for each food ` +
    `item in this sentence: "${text}". Reply with ONLY a JSON array like ` +
    `[{"label":"...", "emoji":"...", "qtyLabel":"...", "min":0, "max":0, "protein":0, "carbs":0, "fat":0}].`;

  const res = UrlFetchApp.fetch(
    "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=" + key,
    {
      method: "post",
      contentType: "application/json",
      payload: JSON.stringify({ contents: [{ parts: [{ text: prompt }] }] }),
      muteHttpExceptions: true,
    }
  );

  const data = JSON.parse(res.getContentText());
  const raw = data.candidates[0].content.parts[0].text.replace(/```json|```/g, "").trim();
  return JSON.parse(raw).map((it) => ({ ...it, recognized: true }));
}
```

Wrap the call in `try/catch` and fall back to `clauses.map(estimateClause_)` on any error (missing key, quota, malformed response) so the app keeps working even if the AI call fails during a live demo.

## Notes for grading / presentation

- The parser (`splitClauses_`, `parseClause_`, `findFood_`, `estimateClause_`) is pure logic with no Apps Script–specific globals, so it can be unit-tested outside the Apps Script editor (e.g. in Node) if you want to show test coverage.
- `getDashboardData()` is the one function doing all aggregation — a good place to point to if asked "where does the analytics come from?"
- Estimates are clearly labeled as approximate throughout the UI; this isn't medical or nutritional advice.
