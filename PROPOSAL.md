# Proposal: Road Tripping Master

## The three ideas

1. **Road Tripping Master:** my own road trip planner that I keep using after this class: I describe the trip in plain words, an AI copilot splits it into driving days with real drive times and decides who drives each leg, and my trips stay saved in my account. For me and the friend I drive Texas to Boston with.
2. **Foundry Print Planner:** a live board of the 3D printers at the Babson Weissman Foundry with an AI helper that tells a student which printer to use and when. For students who print at the Foundry.
3. **All-cards spending tracker:** one page that collects my spending from every card, including the cards that only text me, and helps me plan a budget. For me.

## Why not the other two

I picked Road Tripping Master because I need a personal road trip planner that I don't need to pay for, and I still have lots of features that I want to add on.

- **Foundry Print Planner:** the data isn't mine. The printer status lives in a local tool Jake (Foundry staff) runs inside the Foundry, and showing it to students outside needs the Foundry manager and Babson IT to agree. I can't count on that before 12/02, and without real data the board is just a page that staff update by hand.
- **Spending tracker:** some of my cards only send text messages, and no API a browser page can call reads those. Bank APIs like Plaid need real bank logins, and I don't want my card data in a class project.

## What I am building

My own road trip planner, one I will keep using, where an AI copilot turns my plain-words trip idea into a day-by-day plan with real drive times and a driver for each leg, saved to my account and shareable with the people coming along.

## Who it is for

Me and [Name], who drive long trips together (Texas to Boston). Google Maps and Apple Maps give one total time, not "where do we sleep each night if we drive 8 hours a day", nobody tells us who should drive which part, and our plan ends up in a group chat that nobody can find later. I want one tool that is mine, that remembers every trip I've taken, and that I keep using after this class.

Built from my Mini Project 3 (SamYe-git/road-trip-planner); every feature below is one that project does not have.

## What the user does

- Types the trip into **Ask the copilot**, for example "Leave Austin Friday 8am, end in Boston, max 8 hours of driving a day, want BBQ and one national park", and clicks **Plan it**.
- Sees a proposed plan: Day 1, Day 2, Day 3, each with the overnight town, the hours driven that day (worked out by my code from OSRM, not by the model) and a one-line reason from the copilot. A day over the limit is marked in red.
- Searches for any place, not only a town: an address, a restaurant, a park, a trailhead. Suggestions show up as they type, and picking one adds it as a stop. They can also click anywhere on the map to drop a stop there. The map shows every stop and the route line, and the day markers move when the plan changes.
- Clicks **Use this plan** to load it into the planner, or **Try again** with a change ("make it 4 days").
- Signs in with email to **Save** the trip. On another device, signs in and sees the same trips.
- Clicks **Share**, copies a link, and sends it to a friend. The friend opens it without signing in and sees the plan, read-only.
- Enters the drivers' names ("Sam, [Name]") and a longest stint (for example 2.5 hours). Each leg shows who drives it, and no one drives longer than the stint in a row.
- Sees the forecast for each overnight town on the day they arrive, with a copilot note if snow or heavy rain is expected.

## Data

- **Fetched:** OSRM (route and per-leg drive time and distance), Open-Meteo (forecast at arrival). Both free, no key, CORS already checked in MP3.
- **Place search:** Photon (photon.komoot.io), built on OpenStreetMap, free, no key, so it finds addresses, businesses and parks, not just towns. It also turns clicks on the map into place names. I'll wait about 300 ms after the last keystroke before searching, so I don't send a request on every key.
- **Map:** Leaflet (pinned version), with OpenStreetMap tiles and the attribution their tile policy asks for.
- **Model:** OpenRouter, called through my own Vercel function `api/copilot.js`. The key lives only in Vercel's settings. The function sends my instructions plus the user's request; the model answers in JSON (list of overnight towns + a reason each). My code geocodes those towns and checks each day against OSRM, so the drive times are never made up by the model.
- **Stored:** Supabase table `trips` (`id`, `user_id`, `name`, `stops`, `settings`, `share_id`, `is_shared`, `updated_at`). Row Level Security: a signed-in user can read and change only their own trips; anyone can read a trip whose `is_shared` is true, by its `share_id`. Supabase Auth for email sign-in.
- **Hosting:** Vercel (needed for the model function).

## Features

1. **Ask the copilot (AI):** plain-words trip request, turned into a day-by-day plan with overnight towns and a reason for each. My code recomputes each day with OSRM and flags any day over the driving limit I set, so a bad AI plan can't slip through.
2. **Driver rotation:** enter two or more drivers and a longest stint; each leg is assigned a driver so no one drives longer than that in a row, and each driver's total hours are shown.
3. **Accounts and sync:** sign in, save trips to Supabase, see them on phone and laptop.
4. **Share link:** a read-only page of the trip that a friend can open without an account.
5. **Weather at arrival:** forecast for each overnight town on the arrival day, with a warning for snow or heavy rain.
6. **Search anywhere and a map:** search any address or place, or click the map to add a stop; the map shows the stops, the route and the overnight towns.

## States and their tests

| State | What it looks like | Test: what I do, and what I should see |
|---|---|---|
| Empty | Signed in with no trips: "No trips yet. Tell the copilot where you want to go." and the Ask box. | Sign in with a new email, open the link. I should see that message and no trip cards. |
| In use | Day cards with overnight town, hours driven (from OSRM), the copilot's reason and the arrival forecast. | Ask "Austin to Boston, 8 hours a day". I should see 3 or 4 day cards, each under 8 hours, each with a forecast. |
| Invalid input | Empty request or over 400 characters: "Tell the copilot where you start and end." A daily limit under 1 or over 14 hours, a stint under 0.5 hours, or no driver name is refused. Nothing is sent. | Click Plan it with the box empty. I should see the message and no copilot request in DevTools Network. |
| Search | Typing 3 or more letters shows up to 5 suggestions with the place name and its city and state. Picking one adds a stop and a pin on the map. | Type "Franklin Barbecue". I should see it in the list with "Austin, TX" under it; clicking it adds a stop and a pin in Austin. |
| No search result | Search with no match: "No places found for '…'. Try a city or address." Nothing is added. | Type "zzqxv". I should see that message and no new stop or pin. |
| Map | Pins for every stop, the route line, overnight towns numbered by day. Clicking the map adds a stop with its nearest place name. | Plan Austin to Boston, then click a spot in Tennessee. I should see a new numbered pin there, its name in the stop list, and new drive times. |
| Error (search/map) | Search failing: "Search isn't working right now. Try again." Map tiles failing: grey map, and the stop list and day cards still work. | Block photon.komoot.io in DevTools and type a place. I should see that message, and the planner should still work. |
| Driver rotation | Each leg shows a driver name; totals per driver at the bottom. A leg longer than the stint is split at a rest stop or marked "swap halfway". | Enter "Sam, Alex" with a 2-hour stint, plan Austin to Dallas (about 3 h). I should see the drive split between both names, neither over 2 h. |
| Over the limit | If an AI day is longer than the limit, that day turns red: "9 h 40 m, over your 8 h limit." | Set the limit to 3 hours and ask "Austin to Boston in 2 days". I should see red days, not a plan that pretends to fit. |
| Error (model) | "The copilot isn't answering. You can still add stops by hand." The manual planner still works. | Block the copilot request in DevTools, click Plan it. I should see that message and can still add a stop. |
| Error (route/weather) | "Couldn't get drive times. Check your connection and Retry." with a Retry button; weather failing shows "Forecast unavailable" on the card only. | Turn on Offline in DevTools, click Plan it. I should see the message and a Retry button. |
| Loading | Plan it button turns into "Planning…" and is disabled; day cards show a loading line. | Set Network to Slow 4G, click Plan it. I should see "Planning…" and can't click twice. |
| Long answer | Each reason is cut to 2 lines with "more"; the plan list scrolls inside its panel. | Ask "Explain every possible route in detail". The page shouldn't stretch past the panel. |
| Signed out | Can use the copilot and see the plan, but Save says "Sign in to save this trip". No trip list. | Open the link in a private window, plan a trip, click Save. I should see the sign-in prompt and no saved trips. |
| Shared link | Read-only plan, no Edit, Save or Delete buttons. A link to an unshared or deleted trip: "This trip isn't shared." | Share a trip, open the link in a private window. I should see the plan and no edit buttons. Turn sharing off, reload: "This trip isn't shared." |

### AI test inputs

| Input | A good reply must |
|---|---|
| "Austin to Boston, max 8 hours a day." | Give 3 or 4 overnight towns that are real places along I-30/I-40/I-81 or similar, in order. |
| "Austin to Boston in one day." | Say it can't be done in one day within a safe limit and suggest a number of days. |
| "Dallas to Chicago, want BBQ and a national park." | Mention a BBQ town or stop and a real national park near the route, not one 500 miles off. |
| "What's the capital of France?" | Say it only helps plan road trips. |
| Any input | Be valid JSON in the format the instructions ask for, so my code can read it. |

## What I do not know yet

- **Can the model reliably answer in JSON with real town names?** My code needs to read its reply. If it adds extra text or invents towns, the geocoding will fail.
  - What I tried: I sent the model my instructions and "Austin to Boston, max 8 hours a day". It [gave clean JSON / added extra text / named a town that doesn't exist], so [what I'll change].
- **How to make the share link private enough:** a random `share_id` in the URL, and Row Level Security that only allows reading rows where `is_shared` is true.
- **Whether Supabase email sign-in links work smoothly on my phone.**
- **Keeping it running after the course:** the Babson key can't be deployed, so I need my own OpenRouter key with a credit limit, and a free Supabase project pauses after a week without use.
- **Whether the public OSRM server is fast enough for 4 or 5 extra route checks per plan.**
- **How good Photon is at small places:** whether it finds a specific restaurant or trailhead, or only bigger places, and whether the public server is fast enough for search-as-you-type.
- **Leaflet on a phone:** whether the map and the plan fit on one small screen, or the map needs to be a tab you switch to.
