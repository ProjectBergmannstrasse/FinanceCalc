# FinanceCalc v38.5 — single-folder data + Supabase backup

## Where your data went before this (the problem)

v38.4 had a five-tier storage manager that picked the **highest-priority available** tier. OPFS was priority 1 and is available in every modern browser, so it always won. Result: your data was split across three places, none of them a folder you can open.

| Location | What was in it | Visible to you? |
|---|---|---|
| OPFS `my-money-data/` | `core.json`, `photos/` | **No** — sandboxed inside the browser profile |
| IndexedDB `mymoney_v15` | `core`, `photos`, `meta`, `handles` | No |
| localStorage | 8 keys: theme, section, camera grant, capture-off, dismissed prompt, emergency snapshot, photo cache, `__mm_t` | No |

The "pick a folder" tier existed at priority 2 but **could never win**, so choosing a folder did almost nothing.

Supabase: not present in v38.4 at all. Only v24.6 ever had any trace of it.

## Where it goes now

Pick a folder once. Everything lives there:

```
<your folder>/
  core.json          every record
  settings.json      all settings (mirrors what was in localStorage)
  supabase.json      backup config
  photos/            receipts
  backups/           timestamped snapshots, newest 30 kept
```

The folder now **outranks OPFS**, so once chosen it is the real store. IndexedDB and localStorage keep a mirror purely as crash safety — if the folder is unavailable (permission revoked, drive unplugged) the app keeps working and re-syncs.

## Steps

1. Open the app, go to **Settings → Data**, click **Choose data folder**.
2. Pick or create a folder — e.g. `Documents/MyMoney`. Grant read/write.
3. Migration runs immediately: existing records, all photos, and all settings are copied in, and a first snapshot is written to `backups/`.
4. Re-granting: browsers ask permission again after a full restart. Click once and it reconnects.

**Browser support:** the folder picker needs Chrome, Edge, Opera, or Brave (desktop). Safari and Firefox do not support `showDirectoryPicker`, so on those the app falls back to OPFS exactly as before — Supabase backup still works.

## Supabase backup

Run this once in the Supabase SQL editor:

```sql
create table public.financecalc_backups (
  id          bigint generated always as identity primary key,
  device_id   text        not null,
  created_at  timestamptz not null default now(),
  version     text,
  core        jsonb       not null,
  settings    jsonb
);

alter table public.financecalc_backups enable row level security;

-- Simplest workable policy for a personal, single-user app.
-- Anyone with the anon key can insert and read. Tighten this if the
-- key will ever sit on a machine you do not control.
create policy "anon insert" on public.financecalc_backups
  for insert to anon with check (true);
create policy "anon read" on public.financecalc_backups
  for select to anon using (true);

create index on public.financecalc_backups (device_id, created_at desc);
```

Then in the app: **Settings → Data → Supabase**, paste your project URL (`https://xxxx.supabase.co`) and the **anon** key. Config is written to `supabase.json` in your folder.

- **Automatic:** every save queues a push, debounced 8 seconds, so a burst of edits sends one backup.
- **Manual:** `FinanceCalcData.SupaBackup.push()`
- **Restore newest:** `FinanceCalcData.SupaBackup.restoreLatest()` returns the most recent row for inspection before you apply it.

Each backup is a new row, so you get full history, not a single overwritten blob.

## Security note

The anon key is stored in plain text in `supabase.json` and in localStorage — that is normal for an anon key, but it is not a secret that protects your data. With the permissive policy above, anyone holding that key can read your backups. For a personal app on your own machine that is usually fine. If you want it locked down, use Supabase Auth and change the policies to `auth.uid() = user_id`.

## Verification

The storage layer was tested against a mock File System Access API and mock Supabase endpoint — 22 assertions, all passing: migration of records/photos/settings, folder outranking OPFS, forced-tier override still working, settings round-trip, backup snapshots, pruning to 30, correct REST endpoint and headers, payload contents, HTTP-failure handling, and restore.

Not verified: behaviour against a real browser folder picker and a live Supabase project — that needs your machine and your project.
