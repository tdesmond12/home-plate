# Home Plate Recipe Book: Project Notes

Live: https://tdesmond12.github.io/home-plate/  |  Firebase project: `home-plate-3caaa`
Admin account: tdesmond12@gmail.com
Last version noted here: 2026-11-15.41 (`APP_VERSION` in the versioning script; bump on every upload).

## What it is
Baseball/cooking-themed recipe book. One self-contained `index.html` (vanilla HTML/CSS/JS, no build tools needed to run).
Palette: green / cream / red / navy. Bottom tabs: Shopping List, My Kitchen, Recipe Book, Planner. Profile via top-right icon; search via binoculars icon.
Backend: Firebase Auth, Firestore, Firebase AI Logic (Gemini), App Check. Guests can use the app; data is kept in localStorage.

## File structure (4 script blocks)
1. Main app script: recipes, search, kitchen, planner, shopping, profanity filter
2. Add Recipe script
3. Firebase Auth / Firestore / Gemini / App Check script
4. Versioning script (`APP_VERSION`)
Later scripts call earlier globals at call time. The logos are embedded base64 (that is why the file is large).

## Build and test method (what I used)
- Source: `template.html` with placeholders `__MARK__` and `__LOGO__`; the output `index.html` has the base64 logos substituted in.
- Edits were done with small Python assert-replace patches, then a rebuild, then a syntax check (node `new Function` on each script block).
- Tests: Playwright + Chromium with Firebase stubbed (the sandbox can't reach real Firebase or Gemini).
- If working directly on `index.html`, search for the function names below instead.

## Features and key decisions
**Ingredients and kitchen**
- Pantry and Refrigerator in My Kitchen are built only from ingredients in recipes the user can see (All Recipes plus My Recipes). Duplicates are merged by `canonKey` (singularize + synonym map).
- Chicken thighs and chicken breasts stay SEPARATE (user decision).
- Onions and potatoes are fixed in the pantry (`fixedPlacement`) so users don't move them.
- Other items are placed by keyword rules (`PLACE_RULES`, `placeOf`).
- Gemini (`ingKeysFor`) cleans ingredient names on recipe save and stores `ingKeys`; there is a local fallback if Gemini fails.
- Profile has a one-time "Tidy ingredient names" button that disappears after running.
- Selected kitchen items are saved to `users/{uid}.pantry` (guests: `hp_pantry`).

**Search**
- Category filter (any-of), a collapsible filters panel, and an "only recipes I can make with what's in my kitchen" checkbox (off by default; `canMake`).

**Planner**
- Tabs: This Week / Randomizer.
- Randomizer card packs: name, 1-7 meals, max difficulty, kitchen-only option, art (6 designs); up to 12 saved packs.
- Packs open by swiping or tapping, with a tear and firework animation, then reveal, then Accept or "Rip another pack".
- Pool is All Recipes. Accept saves meals to This Week and adds ALL their ingredients to the shopping list.
- Clearing This Week resets the randomizer and returns to the shelf.

**Shopping list**
- One line per ingredient, with amounts.
- Two tabs: "To Buy" and "Have" (Have = items in the user's selected kitchen).
- Recipe page: "Add to shopping list" button under the ingredients (`addToShopping([r], true)`). Notes added this way carry `d: true`, so taking the meal off This Week doesn't remove them. The button then becomes "View shopping list".
- Every shopping item has a remove (✕) button. To Buy items from recipes also have ✓ "I already have this" (adds the item to the kitchen/pantry, so it moves to Have); Have items have a cart button to move back to To Buy (removes it from the pantry).
- Share icon at the top right of Shopping List opens an export dialog (`openShopExport`): Share… (phone share sheet), Email, Text message, Copy list, Print, and Open in new window (standalone printable page via `shopHtml`). Options: show amounts and recipe names, include checked-off items, include items I already have. Long lists are trimmed for mailto/sms link limits (`fitLink`).
- Removing a meal from This Week removes its recipe-derived ingredients.
- Manual items can be typed in and removed. They are NOT used anywhere else.

**Guest data**
- On sign-in, the user is asked whether to copy guest data (recipes etc.) into their account. Local guest data is kept either way. Asked once per account (`hp_guest_asked_<uid>`).

**Sharing**
- A share icon sits at the top right of each recipe page and opens a small dialog: Share… (phone share sheet, when available), Email, Text message, Copy link (`openShare`).
- Private recipes (owner only): the dialog offers "Make public and share link" or sending the full recipe text.
- Public recipes share a link `?recipe=<id>`; opening it jumps to that recipe once recipes load (`openPendingShare`). Private recipes can't be opened by others, so they can share the full recipe text instead.
- In-app messaging plugs into the share dialog through `SHARE_HOOKS.inApp` (now set to `openSendDialog`).

**Privacy Policy and Terms**
- In-app page (`#legal`, `openLegal`) with Privacy Policy / Terms of Use tabs, reached from the footer link on every screen and the note on the Profile page. Text lives in the `LEGAL` object in the main script; update `LEGAL_UPDATED` when it changes. `CONTACT_EMAIL` (privacy contact) is set in the same place. It describes what the app stores, Gemini use, who can see what, deletion, and house rules; update it whenever data handling changes. Not legal advice.

**Editing and deleting**
- Owners get a pencil icon next to the share icon on their recipe page; it reopens the Add Recipe form in edit mode (`openAddRecipe(editPrefill(r))`, `S.editId`). Everything is editable except the credit (sourceName/sourceUrl/sourceNote/importedFrom are locked by the Firestore `creditUnchanged` rule). Saving uses `setDoc` on the existing doc; ingKeys are regenerated when ingredients change. The profanity check applies.
- Profile > "Delete my account and data" (`openDeleteAccount`): type DELETE, re-sign-in with Google (`reauthenticateWithPopup`), then it deletes the user's recipes, messages (sent and received), directory entry and users doc, and finally the Auth account (`deleteUser`). Reports the user filed are kept for moderation. Saved copies other people made stay with them. If a step fails the Auth account is left intact so it can be retried.

**In-app messaging (mailbox)**
- Menu > Messages (red unread badge; red dot on the profile button). Tabs: Inbox / Sent. Opening a message marks it read.
- Send: share icon on a recipe > "Send in Home Plate" > enter the other person's email + optional note (300 chars). Only signed-up users with a verified email can receive (looked up in `directory/{lowercased email}`, written by each user on sign-in via `ensureDirectory`; get-only, no listing).
- `messages/{id}` stores a SNAPSHOT of the recipe (works for private recipes too), plus fromUid/toUid/names/note/read/savedAs. Receiver can only update `read` and `savedAs`; sender or receiver can delete.
- Receiver options: "Save to My Recipes" (private copy, `importedFrom: 'shared'`, credit = original `sourceName` or the creator/sender's name) or "Edit first, then save" (opens Add Recipe prefilled; credit locked). Message then shows "View my copy".
- Messages refresh on load, when opening Messages, when the tab regains focus (60s throttle) and every 2 minutes. Profane messages are hidden from the inbox; notes are profanity-checked on send.
- Block: "Block sender" on a message (or the checkbox in the Report dialog) adds their uid to `users/{me}.blocked` (+ parallel `blockedNames`), deletes their messages, and the Firestore rule on message create rejects blocked senders (sender just sees "may not be accepting messages"). Unblock from the "Blocked senders" card at the bottom of Messages.
- Report: Report dialog (reason + details, optional "also block") writes `reports/{id}` with a copy of the note and recipe text. Only the admin (tdesmond12@gmail.com) can read/dismiss reports: admins get a third "Reports" tab in Messages.
- Report a public recipe: share icon > Report (non-owners, public recipes only). Reason + details + optional "also hide this recipe from me" (`users/{me}.hiddenRecipes`). One report per person per message/recipe: the report doc id is `<reporterUid>_<messageId or recipeId>` and rules reject a second create. Admin: Reports tab shows open reports (`status` unset); "Remove recipe" deletes the recipe (admin-only rule) and marks the report `removed`; "Dismiss report" marks it `dismissed` (kept so the reporter can't re-file).
- Known gaps: no send rate limit (Firestore rules can't rate-limit), no push/email notifications, admin can only dismiss reports (to ban a user, use the Firebase console).

**Profanity filter**
- Checks the recipe name, description, ingredients, directions, creator name, source, and custom category names on save.
- Message: "Strike called! 3 strikes and you're out! Please keep it family friendly and refrain from profanity. Check <field> (<masked word>)."
- The word list is stored ROT13-encoded in `BAD_RAW`. It handles stretched letters, symbol swaps and masked spellings.
- Mild words (damn, hell, crap, balls) are intentionally allowed. Not caught: spaced-out letters ("S H I T").
- Others' recipes containing blocked words are hidden from viewers; owners still see their own.
- LIMITATION: client-side only; direct database writes bypass it.

## Firestore rules (published in the Firebase console, not in the HTML)
Latest copy: `firestore.rules` in this repo. Highlights: owner-only recipe update/delete, the credit fields can't change, `validRecipe` (ingKeys size <= 150), and the users doc is private to its owner. Publishing the updated rules is optional because the old rules already allow ingKeys.

## Open items / ideas
- Enforce App Check once the metrics look good.
- Possible: always-visible "re-tidy" button; ignore pantry staples (salt, pepper, water, oil) in kitchen-only filters; tighten or loosen the profanity list; server-side profanity check.

## Starting a new chat
Attach `index.html`, this file and `firestore.rules`, and say "continue Home Plate".

## Shopping list by store section (v40)
- Each To Buy item is sorted into a store section (Produce, Bakery, Meat & Seafood, Dairy & Eggs, Frozen, Canned & Jarred, Pasta/Rice/Dry Goods, Baking, Spices, Oils/Sauces/Condiments, Drinks, Household, Other) by keyword rules (`SEC_RULES`, first match wins).
- A per-item dropdown overrides the guess (stored as `sec` on the shop item).
- "Store layout" dialog: user sets the section order and an aisle/area per section (`storeCfg` {name, order, aisle}; saved to users/{uid}.store or localStorage `hp_store`). Real store aisles are not looked up; they are typed in.
- Toggle "By store section" / "A-Z" (`hp_shopgroup`). Exports (text, email, SMS, print, window) follow the grouping via the `group` option.
- Known gaps: keyword guesses are imperfect; guest store layout is not copied on sign-in; no real store aisle data.

## Multiple stores and remembered aisles (v41)
- `stores` = array of {id, name, order, aisle{sectionId: text}} (max 8), `activeStoreId`; saved to users/{uid}.stores and .activeStore (guests: localStorage hp_stores / hp_active_store). The v40 single `store` field is migrated into a store with id 'main'.
- "Stores & aisles" dialog adds, renames, deletes and reorders stores; a store picker shows on the list when there are 2+ stores. The button shows even when the list is empty.
- Ingredient memory `shopMem` (users/{uid}.shopMemory): `sec` = ingredient -> section (all stores), `aisle` = store -> ingredient -> aisle text. Keyed by `canonKey(name)`, so it survives clearing the list. Stored as arrays of objects (`[{k,s}]`, `[{st,k,a}]`), not maps, because setDoc merge would otherwise never delete removed keys.
- Within a section, items sort by their remembered aisle; exports show "(Aisle 9)" per item.
- Known gaps: guest data (stores, memory) is not copied on first sign-in; memory is capped (600 sections, 500 aisles per store).
