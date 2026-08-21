# Fitness Tracker — Generic Build (v1)

This is a third, standalone build — separate from `Josh_Tracker` and `lauren_tracker`, which are untouched. It's a generic version of the same tracker, designed so anyone (starting with your sister) can set it up with their own details rather than yours or Lauren's being baked in.

It's a single self-contained HTML file: `generic_tracker_v1.html`. Open it in a browser and it runs — no build step, no server required for the core app.

## What's new vs. the personal builds

**1. Personalised login.** First screen asks for a name. It's used throughout — header title, AI chat greeting, the welcome-back banner — so it feels like *their* app, not a copy of yours.

**2. Sex (male/female).** Selected during setup, feeds directly into the macro calculation (see below) and can be changed later in Settings.

**3. Age, height, current weight.** Collected during setup, used for the BMR calculation and stored on the profile. All editable later in Settings.

**4. Goal weight.** Still there, now paired with a goal *type* (see #7).

**5. Streak, made more rewarding.** Two things changed:
   - A **welcome-back banner** appears once per day, the first time the app is opened that day — e.g. *"Welcome back, Sam! You've hit your targets 7 days running. Keep it going today. One full week — awesome consistency!"* It reads the actual streak data (calorie deficit + protein target, same logic as before), not a guess.
   - The **streak card** itself is more visual now — bigger flame icon, escalating emoji at 7/14/30-day milestones, plus a "Best streak" line so a broken streak doesn't erase the achievement.
   - Important caveat: this is an **in-app** greeting, shown when the app is opened. It is not a push notification — a real "ping on your phone at 7am" notification needs a backend and native app wrapper (or web push + a server), which is a separate, bigger project. Worth doing later if this goes toward a real app store release.

**6. Two colour schemes.** "Ocean Blue" (your original scheme) and "Teal" (Lauren's original scheme) are both built in as a picker during setup and a swatch-picker in Settings — switching is instant, no reload needed.

**7. Goal type — Lose weight / Maintain / Build muscle.** Selecting one changes the macro targets. The calculation:
   - **BMR** via the Mifflin-St Jeor equation (the most accurate widely-used predictive formula, and it's sex-specific by design).
   - **× activity level** (a new field — Sedentary through Very Active) to get maintenance calories.
   - **Lose weight:** ~20% below maintenance, protein at 2.2g/kg bodyweight (higher to protect muscle in a deficit).
   - **Maintain:** at maintenance, protein at 1.6g/kg.
   - **Build muscle:** ~10% above maintenance, protein at 2.0g/kg.
   - **Fat:** 25% of calories (male) / 30% (female) — women get a higher floor, standard sports-nutrition guidance for hormonal health.
   - **Carbs:** whatever calories are left over, floor of 50g.
   - There's a safety floor on total calories (1500 male / 1200 female) so an aggressive goal-weight input can't produce a starvation-level target.

**8. Settings tab — full profile + macro breakdown.** Tap the ⚙️ in the header. It now shows: editable name/sex/age/height, goal type + goal weight + activity level with a "Recalculate targets" button, a plain-English explanation of *why* your macros are what they are (see the card under "Your Macro Targets" — this is generated from the same formula, not a separate AI call, so it's instant and always consistent with your actual numbers), the colour-scheme picker, and the existing tab visibility toggles.

## v2 refinements (from Lauren's first-look feedback)

**9. Quick Add is now personal.** It used to be a fixed list of generic staples. Now it's an editable list, seeded with those same staples on first run — tap "➕ Add food" to save your own regularly-eaten items (name, calories, protein, fat, carbs, fiber, a serving description and an emoji), and any item you added yourself gets a small ✕ to remove it. The seeded defaults can't be removed, only your own additions — keeps the list from being accidentally emptied out.

**10. Fiber tracking, and tracked nutrients are now optional.** Fiber is calculated the same way as the other macros (~14g per 1000 calories, a standard dietary-guideline benchmark) and shown everywhere macros are shown — Today's ring, the meal log, manual entry, food lookup, photo lookup, and logging via the Ask chat. New in Settings under **"Tracked Nutrients"**: Fat, Carbs and Fiber can each be turned on/off independently to declutter Today and the log form down to just what you personally want to watch. **Calories and Protein always stay on** — the streak and daily status logic is built around hitting those two, so making them optional would need a bigger rework of that engine; flagging this since it's narrower than "make all of them optional." Defaults are unchanged from before (Fat on, Carbs on, Fiber off) so nothing changes for anyone who doesn't touch the new setting.

**11. Fasting — both styles, fully personalised.** New optional **Fasting** tab (off by default, turn it on in Settings same as Peptides), with two independent tools:
   - **Daily eating window** — set any window length and start time (not locked to 18:6 or 16:8), and it shows a live "Eating window open, closes at X" / "Fasting, eating opens at X" status that updates itself.
   - **Extended / periodic fast** — 24h/48h/72h/5-day presets or any custom length. Start it, watch a live progress ring with elapsed/remaining time, and end it whenever — completed vs. stopped-early both get logged to a simple history list underneath.
   These are independent of each other and independent of daily meal logging, so someone running a 3-day fast isn't fighting the calorie/protein targets in the meantime.

**12. The Ask chat can now log things for you.** This was already promised in the quick-start guide but hadn't actually been wired up in this build — fixed. Tell it "log 2 eggs and toast" or "log my weight at 68.4" or "I did 8000 steps today" and it'll estimate/parse and add it directly, with a ✅ confirmation, instead of just talking about it. It only acts when you've actually asked it to log something — mentioning food in passing won't trigger it.

## The Fat-tracking bug — found and fixed

Your description ("fat 28g, Fat 30g = 2830g") nailed it down: that's textbook JavaScript string concatenation instead of numeric addition — `"28" + "30"` gives `"2830"`, not `58`. Anywhere a macro value ends up stored as a string rather than a number (an old schema, a rough sync merge, or — the most likely path in a generic AI-driven app like this — an AI response that quotes a number as `"28"` instead of `28`) and then gets added with a plain `+`, JavaScript silently concatenates instead of throwing an error. That's exactly the runaway-number pattern you saw.

I checked this build's own totals code and it had the identical vulnerable pattern (`a.fat + (m.fat||0)`, `a.fat+m.fat` — this code was ported over from your build, so if it had crept in on your side, it was sitting here too, just not yet triggered). I didn't find a literal duplicate `fat`/`Fat` field anywhere in this build's data model — only one `fat` key is ever used — so I can't confirm that was the exact mechanism here specifically, but the underlying JS pitfall you described is real and was present. I've fixed it three ways so it can't recur, from any entry point (manual, Quick Add, photo/text AI lookup, or the new chat logging):
- **At the source** — `addMeal()` and `addWater()` now force every macro value to a real number before it's ever saved, regardless of what type arrives.
- **At read time** — the totals calculations (`totalsForDate`, `waterTotalForDate`) now coerce on every read too, so even data that's already sitting in storage with the wrong type gets summed correctly instead of concatenated.
- **On load** — a one-time cleanup pass normalises any already-saved meals/water entries the same way, so if you'd been using this build for a while before this update, any bad values already in your local data get quietly corrected the next time it opens.

Verified with a Playwright test that deliberately seeds meals with string-typed macros (`"28"`, `"30"`) the way a quoted-number AI response would — totals now come back as `58`, not `2830`.

On the **data disappearing when scrolling back**: I tested this directly — seeded a meal on "yesterday," navigated back a day with the actual ← control, and it displayed correctly (totals, ring, meal list, all present). I couldn't reproduce that symptom in this build under a normal scenario, so it may be tied to whatever specific sync/data state triggered the Fat issue on your personal build rather than a general bug in the date-scrolling logic itself. If it happens again in this build, the most useful thing to grab is: was Drive sync connected at the time, and did it happen right after opening the app on a different device — that'd point at a sync-merge conflict rather than local logic.

If you get a chance to push your personal build's fix to the repo, I'm happy to diff it against what I did here and port over anything I missed.

## Other changes worth knowing about

- **Peptides tab is now generic and off by default.** No preset compounds, no personal dosing schedule — it's a blank tracker you turn on in Settings and add your own entries to, same lookup-on-add behaviour as before.
- **All personal health content was stripped**, per your call: no FIFO/shift-work references, no specific health conditions in the AI chat prompt, no hardcoded allergy banner. The AI chat and food-lookup prompts now build a short profile description (age/sex/height/weight) from whatever the user actually entered, instead of anything hardcoded.
- **Quick Add and Supplements seed lists are generic** (common everyday foods; Multivitamin/Vitamin D/Omega-3 as starter supplements) rather than either of your personal stacks.
- Everything else — meal logging (photo/text lookup/manual), water, movement, weight, sleep, the AI "Ask" chat with its log-via-chat actions, and Google Drive sync — carried over unchanged in behaviour.

## What is NOT ready for a store or wider release

You already flagged the two big ones — flagging the specifics here so they don't get lost:

1. **Google OAuth client** — the app still uses your personal Google Cloud OAuth client ID for Drive sync. Fine for your sister using your account; not fine for strangers. Before wider release this needs its own verified OAuth consent screen and client.
2. **AI proxy** — every AI feature (food lookup, photo analysis, chat, peptide/supplement lookups) calls `josh-tracker-proxy.joe-shwa-smith.workers.dev`, which is your personal Cloudflare Worker holding an Anthropic API key. Fine for family use; a real release needs its own backend with proper rate-limiting, auth, and cost controls — otherwise anyone with the URL can spend your API credits.
3. **Push notifications** — the streak greeting is in-app only (see #5 above). True notifications need a backend + service worker (web push) or a native wrapper (Capacitor/similar) with push entitlements.
4. **No real user accounts / no server-side data** — data lives in the browser's local storage plus (optionally) the user's own Google Drive appData folder. That's fine for personal use across your own devices, but there's no multi-device login system, no password recovery, nothing a typical app-store app would have.
5. **Icons are placeholders** — the file references `fitness-tracker-icon-192.png` / `-512.png`, which don't exist yet. Generate or design real icons before treating this as installable.
6. **No app name/branding decided** — I used "Fitness Tracker" as a plain placeholder title. Worth deciding on a real name before going further.

## Suggested next steps, if this goes toward commercialisation

- Stand up a small backend (even a lightweight one) to own the Anthropic API key and add per-user rate limits.
- Move to a proper OAuth setup (or drop Google Drive in favour of the backend's own accounts + database).
- Wrap as a PWA with a manifest + service worker (gets you installability and a path to real push notifications) or use something like Capacitor to ship to app stores.
- Add real icons and a settled app name.
- Consider wearable integration (Terra or similar) now that Movement is a first-class tab — that's flagged in-app already as a future item.

## Testing performed

The build was syntax-checked and smoke-tested end-to-end with a headless browser: full setup flow (both "lose weight" and "build muscle" paths, male and female), macro calculation verified by hand against the Mifflin-St Jeor formula, theme switching, tab visibility toggling (including turning Peptides on), meal logging updating the protein ring, and the welcome-back banner rendering correctly for both a fresh-start and a 7-day-streak scenario.

**v2 additions were tested the same way:** Fiber ring hidden by default and appearing after enabling it in Settings > Tracked Nutrients; adding and removing a personal Quick Add item and confirming its calories/fiber land in the day's totals; turning on the Fasting tab, setting a custom eating window and confirming the live status label updates; starting a 24h extended fast, confirming the progress ring and target label, then ending it and confirming it lands in history; navigating back to a day with real logged data and confirming it displays correctly; and — for the Fat bug specifically — seeding meals with string-typed macro values (mimicking a quoted-number AI response) and confirming totals now add numerically instead of concatenating. No console/JS errors in any of this — the only console noise is expected (the placeholder icon files and Google's sign-in script, both already called out above/below as not-yet-ready items, blocked by this sandboxed test environment rather than anything wrong in the app).
