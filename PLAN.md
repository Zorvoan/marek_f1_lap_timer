# Lap Timer: Project Plan

A mobile app for timing drives on routes you define yourself, racing against your
own earlier times, and competing with other drivers on routes they publish.

**Only approved users can create routes.** An admin grants that permission from an admin account. Everyone else can still drive public routes and appear on their leaderboards.

---

## 1. Feature summary

| Area | What it does |
|---|---|
| **Roles and permissions** | Three roles. A **driver** (every new account) can drive public routes, race their own times and appear on leaderboards. A **creator** can also create routes; an admin grants this. An **admin** grants and revokes creator access and can moderate public routes. |
| **Route builder** | *Creators only.* Draw a route on a map, snapped to roads. It can be point-to-point or a **lap** (loop, where start = finish). Choose a **sector length** (50–1000 m) and a **standing** or **rolling** start. |
| **My routes** | Each creator's routes are private. Only the owner can see them, except for the one public route described below. Drivers see the public routes they have driven. |
| **Drive** | Pick a route and a previous run to race against (the reference), then drive. Times are taken from GPS crossings of the start, sector and finish gates. |
| **Drive screens** | **Full HUD**: running time, delta, sector times, etc. **Delta-only**: the delta fills the screen, and the background colour shows how the last sector went (yellow/green/purple). |
| **Delta logic** | The delta updates only when you **cross the end of a sector**: your elapsed time at that sector boundary minus the reference run's elapsed time at the same boundary. |
| **Social** | Each creator can make **one** route public, and it gets a leaderboard. The feed is a map showing every public route. Tapping a route opens its leaderboard. |

---

## 2. Recommended tech stack

| Layer | Choice | Why |
|---|---|---|
| Mobile app | **React Native + Expo** (TypeScript) | One codebase for iOS and Android. Expo dev builds still allow native modules for GPS and maps. |
| Maps | **Mapbox** (`@rnmapbox/maps`) | Good custom styling for a dark "racing" look. Includes the Directions and Map Matching APIs, which snap routes to roads, and a generous free tier. |
| Location | `expo-location` (foreground plus background while a run is active), `expo-sensors` (accelerometer) | GPS fixes for timing. On standing starts, the accelerometer catches a jump start and measures reaction time. |
| Local storage | SQLite (`expo-sqlite`) | Works **offline**: routes, references and finished runs are stored on the device, so driving works without signal (mountain roads). |
| Backend | **Supabase**: Postgres + **PostGIS**, Auth, Storage, Edge Functions | Row-level security covers "only I see my routes". PostGIS handles map viewport queries. Edge Functions run TypeScript (Deno). |
| Shared code | `packages/timing`: a pure TypeScript timing engine | The **same code** runs on the phone during the drive and on the server to verify leaderboard runs. |

### Repo layout (monorepo)

```
apps/mobile/            Expo app (screens, map, drive HUD)
packages/timing/        Pure TS timing engine + unit tests + recorded GPS fixtures
supabase/migrations/    SQL schema, RLS policies, PostGIS indexes
supabase/functions/     Edge functions (verify-run, leaderboard refresh)
```

---

## 3. Core concepts and data model

### Route
- `geometry`: an ordered polyline (LineString). **Direction matters**, so it runs from start to finish.
- `is_loop`: if true, the finish gate is the start gate and laps can run back to back.
- `sector_length_m`: 50–1000 m, in steps of 50 m.
- `sector_boundaries_m[]`: the distances along the route where each sector ends. These are **computed once and frozen** when the route is saved. Example: a 2,300 m route with 500 m sectors has boundaries `[500, 1000, 1500, 2000, 2300]`. If the last sector would be shorter than half a sector length, it is merged into the previous one.
- `start_type`: `standing` | `rolling`.
- Once a route has any runs, its geometry and sectors **cannot be edited**, because edits would make old times incomparable. Users can duplicate the route instead.

### Run
- The time at each gate crossing, stored as cumulative times: `split_times_ms[]` (elapsed time at each sector end). Sector times are the differences between consecutive splits.
- `total_ms`, a `valid` flag, GPS quality stats, and the raw GPS trace (kept in Storage and used for verification and replay).
- On loop routes, each lap is its own run, and runs from the same session share a `session_id`.

### Roles and permissions
| Role | How you get it | What it allows |
|---|---|---|
| `driver` | Every new account | Browse the feed map. Drive **public** routes, see your own runs and appear on leaderboards. |
| `creator` | An **admin grants it** in the admin screen | Everything a driver can do, plus create private routes and publish **one** of them. |
| `admin` | Set once **directly in the database** for your own account; it can't be granted from the app | Grant and revoke `creator`, review access requests, and hide or unpublish any public route. |

- **The server enforces everything**, not just the app UI.
  - Database policies (RLS) refuse a route insert from anyone who isn't a creator, even if someone calls the API directly.
  - Hiding the route builder button is only a convenience.
- **Users can never change their own role.** Roles live in a separate `user_roles` table that only admins can write to, not in `profiles`, where users can edit their own row.
- **Revoking creator access**:
  - The user can no longer create new routes or publish.
  - Their existing routes and runs stay, so other people's leaderboard times are not lost.
  - The admin can separately unpublish or hide their public route.
- **Requesting access**: a driver can tap "Request creator access" and add an optional note. The admin sees pending requests in the admin screen and approves or rejects each one. Every grant and revoke is logged.
- **More admins**: only the bootstrap admin (you) exists at first. Making someone else an admin is also a database operation, which keeps it deliberate.

### Schema sketch

```sql
profiles (id uuid pk = auth.uid, username text unique, avatar_url, created_at)

-- roles: writable ONLY by admins (RLS); a user may read their own row
user_roles (user_id uuid pk fk profiles, role text check (role in ('creator','admin')),
            granted_by uuid fk profiles, granted_at timestamptz)   -- no row = plain driver
creator_requests (id uuid pk, user_id uuid fk, note text, status text check (status in ('pending','approved','rejected')),
                  decided_by uuid, decided_at timestamptz, created_at timestamptz)
role_audit_log (id bigserial pk, actor_id uuid, target_user_id uuid, action text, created_at timestamptz)

-- helpers used by RLS policies (security definer, stable):
--   is_admin()   := exists(select 1 from user_roles where user_id = auth.uid() and role = 'admin')
--   can_create() := exists(select 1 from user_roles where user_id = auth.uid() and role in ('creator','admin'))

routes (
  id uuid pk, owner_id uuid fk profiles,
  name text, geometry geography(LineString), length_m numeric,
  is_loop bool, sector_length_m int check (sector_length_m between 50 and 1000),
  sector_boundaries_m numeric[], start_type text check (start_type in ('standing','rolling')),
  is_public bool default false, published_at timestamptz, created_at timestamptz,
  hidden_by_admin bool default false   -- moderation: hidden routes vanish from feed/leaderboards
)
-- only ONE public route per user:
create unique index one_public_route_per_user on routes(owner_id) where is_public;
create index routes_geom_idx on routes using gist(geometry);

runs (
  id uuid pk, route_id uuid fk, user_id uuid fk, session_id uuid,
  started_at timestamptz, total_ms int, split_times_ms int[],
  valid bool, verified bool default false, trace_path text,
  countdown_s smallint, reaction_ms int,                    -- standing starts only
  jump_start bool default false, jump_distance_m real, penalty_ms int default 0,  -- +5000 on jump start
  -- split_times_ms already includes penalty_ms; raw times = splits - penalty_ms
  gps_avg_accuracy_m real, gps_rate_hz real, created_at timestamptz
)

-- derived (view or table refreshed by trigger/edge function):
route_sector_records (route_id, sector_idx, best_sector_ms, run_id, user_id)  -- for "purple"
leaderboard          (route_id, user_id, best_run_id, best_total_ms, rank)     -- best run per user
```

### Who can see what (RLS)
- **routes**:
  - The owner can see their own routes.
  - Anyone signed in can **read** routes where `is_public and not hidden_by_admin`.
  - **Insert** requires `owner_id = auth.uid() and can_create()`.
  - Setting `is_public = true` also requires `can_create()`.
  - Admins can update `is_public` and `hidden_by_admin` on any route.
- **user_roles / creator_requests / role_audit_log**:
  - Admins can read and write everything.
  - Users can read their own role and their own requests, and can insert a request for themselves.
  - Grants go through a single `set_user_role(target, role | null)` database function that checks `is_admin()` and writes the audit log in the same transaction.
- **runs**: a user can see all of their own runs. On public routes, other users can see only the **times** (leaderboard and sector records), never the raw traces. A trace can reveal where someone lives.
- Only the owner (and only while they are a creator) can insert or update their routes. Runs on public routes only reach the leaderboard after the server has **verified** them (section 6).

---

## 4. Timing engine (`packages/timing`)

The engine is a pure function with no UI. It takes a stream of GPS fixes `{lat, lon, t, accuracy, speed, heading}` and emits events: `armed`, `started`, `sectorComplete(k, splitMs, sectorMs, deltaMs, colour)`, `finished`, `lapComplete`, `invalidated(reason)`.

### 4.1 Where am I on the route?
1. Before the drive, resample the route polyline every ~2 m and record the cumulative distance at each point.
2. Project each GPS fix onto the polyline to get **`s`** (distance along the route) and **`e`** (lateral offset from the line).
3. Only search a **window** around the last known `s` (from slightly behind to a little ahead, scaled by speed). This handles routes that cross themselves and loops where start and finish are the same point.
4. If `e` stays above ~30 m for more than a few seconds, you are off route and the run is invalidated. Fixes with poor accuracy (more than ~20 m) are flagged and lower the run's quality score.

### 4.2 Precise gate crossings
Phone GPS usually updates at **about 1 Hz**, which at 100 km/h means a fix only every ~28 m. Taking the timestamp of the fix after the line would therefore be off by up to a second. Instead, the engine interpolates:

```
if s_prev < boundary <= s_curr:
    t_cross = t_prev + (boundary - s_prev) / (s_curr - s_prev) * (t_curr - t_prev)
```

- Use **GPS fix timestamps**, not the time JavaScript received the fix.
- Later, Hermite interpolation using the speed at both fixes can improve accuracy when the car is accelerating or braking.
- Expected accuracy with a phone is about ±0.1–0.2 s. An external 10–25 Hz Bluetooth GPS (RaceBox, Dragy, etc.) would be a later upgrade (section 9).

### 4.3 Starts
- **Rolling**: the timer arms when you are within ~50 m of the start and heading in the route's direction. It starts when you cross the start gate, which is a line perpendicular to the route at its first point, using the same interpolation.
- **Standing (countdown start)**:
  1. You stop within ~15 m of the start. The engine confirms you are stationary (speed < 1 km/h for 2 s), then enables the **Start countdown** button. This is the only tap during a run, and it happens while the car is stopped.
  2. A **countdown of 3–10 s** runs. You set its length on the pre-drive screen; the default is 5 s.
     - The timing is **exact and even**: one beat per second with no random hold, so GO always comes exactly N seconds after the tap.
     - It shows as large numbers plus an F1-style row of lights, with a beep each second and a different tone on GO.
     - The countdown runs on a monotonic clock. The GO instant is matched to the GPS time base, so the timer start lines up with the fix timestamps used for the gates.
  3. **The timer starts at GO**, not when the car moves, so your reaction time is part of the lap time. That is fair across all drivers, and the countdown length you choose doesn't affect the time.
  4. **Jump start: a +5 s penalty**, applied only when the car has really moved **more than 10 m forward before GO** (see 4.3.1). Smaller movements are ignored.
     - The run keeps going. The HUD shows a **"JUMP START +5.0 s"** banner for a few seconds.
     - The penalty is added to sector 1, so every split, delta, colour and the final time include it. A jump can never earn a purple sector or a better delta.
     - Penalised runs are still **valid**. They count on the leaderboard with the penalty included and can be used as a reference.
     - The countdown can also be cancelled before GO with no penalty.
  5. **Reaction time** is the time from GO to first motion (accelerometer). It is saved with the run and shown in the summary. After a jump start it is shown as negative, e.g. "−0.4 s".

#### 4.3.1 Jump-start detection without false penalties
GPS **position** is the unreliable part. A phone sitting still often shows its position wandering 3–10 m (or more near buildings and trees). Comparing raw positions against a 10 m threshold would therefore produce false penalties. Instead, the engine estimates the distance moved before GO **from signals that don't drift**:

1. **Anchor**: during the ≥2 s stationary check and the countdown, average the GPS fixes to get one stable start position. The accelerometer baseline (gravity and phone mount angle) is captured at the same time.
2. **Is the car really moving?** Motion counts only when **both** of these agree:
   - **Accelerometer**: sustained forward acceleration above ~0.1 g for more than 300 ms, after a low-pass filter. Engine revs, door slams and phone taps are short spikes and get filtered out.
   - **GPS Doppler speed** (the fix's `speed` field) above ~1.5 m/s. Speed comes from the Doppler shift of the satellite signals, not from position differences, so it stays near zero while the car is stationary even when the reported position is wandering.
3. **How far before GO?** Once motion is confirmed, the distance travelled up to GO is the **integral of Doppler speed over time**. Accelerometer data fills the gaps between 1 Hz fixes. Only movement **forward along the route** counts, so rolling back on a hill is never penalised.
4. **Decision**: a penalty applies only if the integrated distance is **> 10 m**.
   - **Cross-check**: if the GPS reports poor accuracy (> 15 m) during the countdown, motion must also be confirmed by the projected route position `s` being clearly ahead of the anchor once the fixes settle. If the signals still disagree, **no penalty is given**, because the driver gets the benefit of the doubt.
5. **Server re-check**: the trace upload includes a short accelerometer log covering 10 s before GO to 5 s after it. `verify-run` repeats the same decision, so the phone and the server agree on penalties for leaderboard runs.

The 10 m threshold is a setting in the engine (`JUMP_START_THRESHOLD_M`). Because the method doesn't rely on raw positions, it could be lowered later, e.g. to 3–5 m, without causing false penalties. Under the threshold, creeping a few metres gives a tiny head start, which is accepted as the cost of never penalising GPS noise.

### 4.4 Delta and colours (at each sector end k)
```
split_k   = t_cross_k - t_start                    // elapsed at sector end
sector_k  = split_k - split_(k-1)
delta_k   = split_k - ref.split_times_ms[k]         // negative = ahead
colour_k  = purple  if sector_k < route_sector_records[k]   (fastest ever, any user)
            green   if sector_k < ref sector_k
            yellow  otherwise
```
- Purple overrides green.
- The sector records are **downloaded before the drive**, so colours work offline.
- On a **private** route the only driver is you, so purple means your own all-time best for that sector.

### 4.5 Laps
On loop routes, crossing the finish gate completes the lap and immediately starts the next one at the same instant. Each lap is compared against the selected reference lap.

---

## 5. Screens

1. **Auth**: sign up or log in (email, Apple, Google), then choose a username. New accounts are **drivers**.
2. **Social feed (home)**: a full-screen map.
   - Zoomed out, public routes show as clustered start pins. Zoomed in, they show as coloured polylines.
   - Data loads by viewport through a PostGIS bounding-box query, using simplified geometry.
   - Tapping a route opens **Route detail**.
3. **Route detail / leaderboard**: the route on a map, its length, sector count, start type and owner. Below that, a ranked list of each user's best time, with the purple sector records. There is a "Drive this route" button.
4. **My routes**:
   - **Creators** see a list of their routes with thumbnails and personal bests. Each route has a "Make public" toggle.
     - If you already have a public route, turning on "Make public" for another one opens a dialog asking whether to replace it.
     - The dialog also warns that the start and end points become visible to everyone.
   - **Drivers** see the public routes they have driven and a **"Request creator access"** button (or "Request pending" once sent).
5. **Route builder** (*creators only*; the server refuses saves from anyone else):
   - Tap waypoints on the map; they snap to roads through the Directions API.
   - You can drag waypoints and undo.
   - A **Loop** toggle closes the route.
   - A sector slider (50–1000 m) shows a live preview of the sector boundaries as tick marks on the route.
   - Standing/rolling toggle, a name field, and Save.
6. **Pre-drive setup**: choose the route, then the reference run. The run list shows date, total time and sector times, with your PB at the top. On a public route you can also choose the current leader. You also choose the screen mode (Full HUD or Delta-only). On standing-start routes you set the **countdown length (3–10 s)**, which is remembered per user.
7. **Drive screen**: the screen stays awake and there are no controls to tap while moving (large Stop button only).
   - **Standing start**: while you wait at the start line, a big **Start countdown** button appears once you are stopped there. It is followed by the full-screen countdown (numbers, lights and beeps) and GO. A jump start shows a red **JUMP START +5.0 s** banner for a few seconds while the run continues.
   - **Full HUD**: running time, current delta (updated at each sector end), last sector time and its delta, sector k/N, distance to the next gate, and speed.
   - **Delta-only**: a huge `-0.42` / `+1.07`, with the full-screen background coloured yellow, green or purple by the last sector.
   - Both modes give haptic and sound cues at each gate. Optional voice callouts ("Sector 3, minus zero point four") let the driver keep their eyes on the road.
8. **Run summary**: total time (with any penalty shown separately, e.g. `1:42.310 (incl. +5.000 jump start)`), reaction time (standing starts), a table of sector times coloured like the HUD, the trace on the map coloured by sector, and Save or Discard. Valid runs on public routes are queued for upload to the leaderboard.
9. **Run history**: all runs per route, with the option to delete runs.
10. **Admin panel** (*admins only*; hidden for everyone else):
    - **Requests**: pending creator requests with the user's name and note, and Approve or Reject.
    - **Users**: search by username, see each user's role, and **grant or revoke creator** with one toggle.
    - **Public routes**: every published route, with Unpublish and Hide actions for moderation.
    - **Log**: who granted or revoked what, and when.

---

## 6. Leaderboard integrity

- A run only counts if it is `valid` (no off-route time, every gate crossed in order, acceptable GPS quality) **and** `verified`. Jump-start penalties are included in the leaderboard time.
- **Verification**: an Edge Function `verify-run` downloads the uploaded trace and replays it through the **same `packages/timing` engine**. If the times it computes match the times the phone reported (within tolerance), the run is marked `verified` and the leaderboard and sector records are updated.
- Plausibility checks: maximum speed and acceleration limits, and a check that the GPS timestamps are monotonic and not too sparse.
- This stops basic cheating, such as faking times in the request. A spoofed GPS trace is much harder to detect, so it is out of scope for v1.

---

## 7. Milestones

| Phase | Scope | Exit criteria |
|---|---|---|
| **0: Foundations and spikes** | Monorepo, Expo app, Supabase project. **Device GPS spike**: log real update rates and accuracy on a few iOS and Android phones while driving. | Recorded GPS traces saved as test fixtures. Confirmed which update rates are realistic. |
| **1: Timing engine** | `packages/timing` with projection, windowing, interpolation, starts, sectors, laps, delta and colours. Unit tests run against synthetic traces and the recorded fixtures. | Gate crossing error below ~0.2 s on fixtures. Loop and self-crossing routes handled. |
| **2: Accounts, roles and routes** | Auth, **roles** (`user_roles`, RLS helpers, bootstrap admin), **admin panel** (grant/revoke, requests, audit log), the route builder with road snapping, sector preview, loop/start options, and My routes. | Only users the admin has approved can create routes; a direct API insert from a driver is rejected (covered by a test). Creators' routes are private. |
| **3: Drive (MVP)** | Pre-drive reference picker, both drive screens, cues, run summary, offline save and later sync. A **GPS replay mode** plays a recorded trace into the app so drives can be tested at a desk. | A full drive works end to end on a real road, with deltas at each sector. **← First usable version** |
| **4: Social** | Publish one route (creators), admin moderation of public routes, verify-run function, leaderboard, sector records (purple), feed map with clustering, route detail. | Two users can compete on a public route, and the feed map shows it. |
| **5: Polish** | Voice callouts, profiles and avatars, sharing a run as an image, external BLE GPS support, Hermite interpolation, battery tuning. | Ready for app store submission. |

---

## 8. Risks and how to handle them

| Risk | Mitigation |
|---|---|
| **Safety, legal and app store policy.** Timed leaderboards on *public roads* can be read as encouraging street racing. Apple's guideline 1.4.5 rejects apps that urge risky activities, and there is liability exposure. | Position the app for **track days, closed roads, hill-climbs and karting**. Require accepting the terms before the first drive. Do not reward anything just for being fast on unclosed roads (or label public roads clearly). Keep the drive screen free of controls to tap, and use a mount and voice cues. **Decide this early, because it affects the app store listing.** |
| GPS accuracy (1 Hz, urban canyons, tunnels) | Interpolation, quality scoring, invalidating runs with gaps, and BLE GPS later. |
| Battery and heat during long sessions | High-accuracy GPS runs only while a run is armed. A dark UI saves power on OLED screens. |
| No signal on mountain roads | Offline-first: everything needed for a drive is cached, and uploads are queued. |
| Background location permission (store review) | Request "while using" first. Ask for background access only so the timer keeps running if the screen locks, and explain why. |
| Privacy (routes that start at home) | Warn when publishing. Never expose other users' raw traces. |

---

## 9. Later ideas (not in v1)
- Live "ghost" delta between sectors, interpolated from the reference trace.
- Following other users, and a social feed of runs.
- Car profiles (compare times by car class).
- External 10–25 Hz GNSS receivers over BLE.
- Video overlay export (time and delta burned into dashcam footage).

---

## 10. Decisions made
- **Standing start**: countdown of 3–10 s (user-set), exact even timing with no random hold; the timer starts at GO.
- **Jump start**: +5 s penalty, only for confirmed forward movement of more than 10 m before GO (section 4.3.1). The run stays valid.

- **Route creation needs permission**: only users the admin approves (role `creator`) can create and publish routes. Everyone else is a driver who drives public routes. The admin role is set in the database, not from the app.

## 11. Open questions (need decisions)

1. **Platforms**: iOS and Android both (assumed), or one first?
2. **Green means faster than what?** Faster than the *selected reference* (assumed, per the brief), or faster than *your personal best* for that sector (the F1 convention)?
3. **Purple on private routes**: is "your own all-time best sector" right, given that nobody else drives them?
4. **Laps**: on a loop route, should laps run back to back in one session (assumed), with a fixed lap count or until you stop?
5. **Changing your public route**: when you swap to a different route, should the old leaderboard be archived (still viewable) or hidden?
6. **Can others save a public route** to their own list, or only drive it from the feed?
7. **Positioning**: public roads or tracks/closed roads (see the safety risk above)?
8. **Map provider**: is Mapbox OK, or do you prefer Google Maps?
9. **Scope of the permission**: should approval be needed to create **any** route (assumed), or only to **publish** one? With the second option, everyone could build private routes and only approved users could make one public.
10. **Access requests**: should drivers be able to request creator access in the app (assumed), or should you only grant it yourself, e.g. to people you know?
