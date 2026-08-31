# FieldSnap — case study

**Role:** product, design and engineering, solo.
**Timeline:** first commit March 2026; store build last updated April 2026.
**Outcome:** live on Google Play under Koyr Studio. 10+ installs.

---

## 1. The problem

Commissioning, asset surveys and maintenance audits all run on the same task: walk a building, find every piece of equipment, and write down what its nameplate says.

A nameplate is a metal or printed label carrying the data that identifies a unit — manufacturer, model, serial, voltage, phase, rating. It is the only authoritative record of what is physically installed. Drawings go stale; the plate does not.

The capture method has not changed in decades. A technician writes it on a clipboard or types it into a phone note, then transcribes the whole set into a spreadsheet or asset system back at the office. Two transcriptions, one of them at the end of a long day, on data made entirely of near-identical alphanumeric strings. `1LE1001-1AA` and `1LE1OO1-1AA` look the same at 6pm.

The failure mode isn't dramatic. It's a serial number that's wrong by one character, discovered eight months later by someone ordering a replacement part.

---

## 2. Why not "just use AI"

The obvious build is: photograph the label, send it to a vision model, save the text. That works and it is not a product.

Raw OCR of a nameplate returns a wall of strings — manufacturer marks, certification logos, safety warnings, part numbers, a QR code's caption, the actual data — with no indication of which is which. It is not more useful than the photo. To become a *record*, the output needs a shape, and the shape depends entirely on what the equipment is.

A transformer is identified by primary voltage, secondary voltage and KVA. An elevator controller by floors served, car capacity and drive type. A VAV box by minimum and maximum CFM. These sets barely overlap.

So the core product decision was to **make the equipment type a first-class input**, chosen before the photo is taken, and use it to supply the schema the model must fill. Extraction is constrained: the prompt names the exact keys, demands JSON containing only those keys, and instructs the model to return an empty string wherever the plate is not clearly legible.

That decision generated the real design work — not the prompt, the **taxonomy**. 28 equipment types across 6 categories, 179 fields, each with a label and a placeholder drawn from a real plate.

The taxonomy came from desk research: manufacturer nameplate documentation and specification sheets, plus trade forums where electricians, HVAC technicians and facilities engineers discuss what gets recorded on a survey and what gets missed. **It was not validated through field interviews.** It is a researched hypothesis, shipped so that use can correct it, and the honest limitation of the project.

**→ [The full taxonomy](equipment-templates.md)**

---

## 3. Designing for the place the work happens

Three constraints came from the environment rather than the feature list, and each one changed the architecture.

**One bar of signal.** A plant room is a concrete box. Everything is local-first: mutations write to device storage immediately and sync on a 4-second debounce, with timeouts on every network boundary. Nothing in the capture flow blocks on the network. The 15-second fetch timeout on the auth client and the 8-second race on session restore both exist because of a specific bug — a stale token left the app on a grey screen until the user force-stopped and cleared cache, which for a non-developer is an uninstall.

**Gloves and a clipboard already in one hand.** Capture is a video walk-around with snapshots taken during it, rather than a series of single framed photos. A technician records their way around a unit and pulls stills from that, instead of composing shots one at a time.

**No patience for a signup wall.** Guest mode is the entire product, not a preview. The design work is in the seam: guest data migrates into a real account on signup, and the sign-in function *awaits* that migration before switching auth state, so the account's first render already contains the work. Getting this order wrong shows an empty home screen to someone who has just documented a substation.

---

## 4. Two tabs instead of one

The review screen shows extraction results in two tabs: **OCR Text** and **Smart Data**.

The temptation is to show only the structured result — it's cleaner and it's the point of the product. The reason both are there is that structured extraction sometimes misses a field the label plainly shows, and when it does, the technician needs to know whether the problem was the photo or the model. Raw text answers that instantly. A bad photo shows garbled OCR; a good photo with a missed field shows the value sitting right there in the raw text, ready to copy.

The two are handled differently on failure, on purpose:

- **Raw OCR throws.** The error is shown, because raw text is the diagnostic and silently empty diagnostics are worthless.
- **Structured extraction degrades to empty fields.** A blank form the technician fills by hand is a working outcome; an error dialog blocking them from a form they could have filled is not.

The same conservatism governs custom fields. Users can add their own fields to any template, but those fields are filled by hand and never sent to the model — a vague user-authored label ("Notes", "Check") invites hallucinated values into a record whose whole value is being a faithful transcription of a plate.

---

## 5. Shipping was the design constraint

The most instructive part of the project had nothing to do with the interface.

Google Play's Photo & Video Permissions policy forbids `READ_MEDIA_IMAGES` and `READ_MEDIA_VIDEO` unless persistent media-library access is an app's **core purpose**. FieldSnap picks one photo at a time and saves snapshots to the gallery — neither needs those permissions, and the app config never asks for them.

The media-library dependency injects them into the generated manifest regardless. The policy violation is in the build output, and there is no fix at the app-config layer.

The resolution was a custom Expo config plugin that filters those permissions out of `AndroidManifest.xml` during the build, while keeping `READ_MEDIA_VISUAL_USER_SELECTED` — Android 14's partial-access model, which is policy-safe. It ships with the policy reasoning written into the file, because in six months the reasoning matters more than the twelve lines of code.

This is the part of solo product work that doesn't photograph well and matters most: reading a platform policy, understanding why a dependency violates it, and meeting it at the build layer. A designer who stops at the Figma file never encounters this problem. A designer who ships owns it.

The same instinct shows up in pricing. Subscription prices are pulled from the store per country; if the store can't be reached the price cell stays **blank** and the buy button disables to *Currently unavailable*. There is a USD fallback in the service layer and the screen refuses to render it, because showing `$19` to someone whose price is not $19 is a small lie that the payment sheet corrects too late.

---

## 6. The accessibility trade

Android permits 200% system text scaling. Nameplate data is dense and tabular — inherited blindly, that scaling breaks the layouts the product exists to display.

FieldSnap caps it globally with `maxFontSizeMultiplier: 1`, overriding the OS setting, and replaces it with its own appearance suite: light/dark/follow-system, three text sizes, bold text, high contrast, reduce motion.

The implementation is the part worth defending. Text scale and bold-weight are folded into the resolved theme object that every screen already consumes, so they invalidate the same style memos dark mode does. One mechanism, applied at the token layer, referenced 321 times across 38 files without becoming 321 places to forget.

And the limitation is real: **there are no screen-reader labels.** The product is designed for low-vision legibility and motion sensitivity, not yet for non-visual use. Overriding the OS setting is a defensible trade only while the replacement keeps improving, and labelling the icon buttons is the next milestone.

**→ [Design system detail](design-system.md)**

---

## 7. Where it stands

Live on Google Play under Koyr Studio, in-app purchases configured, last updated April 2026, **10+ installs**.

That number is the honest state of it. FieldSnap is a finished product taken through a platform's full review and policy process by one person — design, build, backend, store listing, marketing site — and it is not a business. Presenting it as anything else would be the same small lie as the hardcoded price.

**What I would do differently**

- **Put the schema in version control.** The database was built through a console and its shape is documented only as TypeScript row types in the client. That is a weaker guarantee than a migration history and it is the first thing I would fix.
- **Validate the taxonomy with technicians.** Desk research produced a defensible starting point. It cannot tell me which of the 179 fields are actually filled in and which are dead weight, and that is exactly the kind of question a few site visits answer in an afternoon.
- **Move extraction server-side.** The Gemini call is made from the client, so the API key ships in the bundle. Fine for a free-tier key on a solo product; wrong at any scale.
- **Write the accessibility labels first, not last.** Retrofitting them across 32 screens is now a project. Doing it as each screen was built would have been free.
