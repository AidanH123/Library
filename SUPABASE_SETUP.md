# Putting Larry's Library online with Supabase

Until this is set up, the library saves books in whichever browser you use. With Supabase (free), your shelves are stored online: they look the same on every device, visitors see your books, and their recommendations reach you. Only you can sign in and change things.

## 1. Create a free Supabase project

1. Go to <https://supabase.com> and sign up.
2. Click **New project**. Pick any name (for example `larrys-library`), set a database password (save it somewhere, you won't need it day to day) and choose the region closest to you.
3. Wait a minute or two while it starts up.

## 2. Create the tables and rules

1. In your project, open **SQL Editor** in the left sidebar and click **New query**.
2. Paste everything below.
3. **Replace all three `you@example.com` with the email you'll sign in with.**
4. Click **Run**. You should see "Success. No rows returned".

```sql
-- Tables: each book or recommendation is stored as one row of JSON
create table if not exists public.books (
  id text primary key,
  data jsonb not null,
  updated_at timestamptz not null default now()
);
create table if not exists public.recs (
  id text primary key,
  data jsonb not null,
  created_at timestamptz not null default now()
);
alter table public.books enable row level security;
alter table public.recs  enable row level security;

grant select on public.books, public.recs to anon, authenticated;
grant insert, update, delete on public.books to authenticated;
grant insert on public.recs to anon, authenticated;
grant delete on public.recs to authenticated;

-- Everyone can see the shelves
create policy "Anyone can read books" on public.books
  for select using (true);
create policy "Anyone can read recommendations" on public.recs
  for select using (true);

-- Only you can add, edit or remove books   (replace the email)
create policy "Owner manages books" on public.books
  for all to authenticated
  using ((auth.jwt() ->> 'email') = 'you@example.com')
  with check ((auth.jwt() ->> 'email') = 'you@example.com');

-- Anyone can recommend a book (small entries only); only you can remove them   (replace the email)
create policy "Anyone can recommend" on public.recs
  for insert to anon, authenticated
  with check (length(data->>'title') between 1 and 300 and pg_column_size(data) < 20000);
create policy "Owner removes recommendations" on public.recs
  for delete to authenticated
  using ((auth.jwt() ->> 'email') = 'you@example.com');
```

## 3. Create your sign-in

1. Open **Authentication → Users** and click **Add user → Create new user**.
2. Enter the **same email** you used above and a password. Tick **Auto Confirm User**, then create it.
3. Turn off public sign-ups so nobody else can make an account: under **Authentication**, open the sign-in/provider settings and switch off **Allow new users to sign up**, then save.

## 4. Connect the site

1. Open **Project Settings → API** (it may be under **Data API** or **API Keys**).
2. Copy the **Project URL** (looks like `https://abcdefgh.supabase.co`) and the **anon public** key (or the **publishable** key starting `sb_publishable_`).
3. Send both to Claude, or paste them yourself near the top of the script in `index.html`:

```js
const SUPABASE_URL='https://abcdefgh.supabase.co';
const SUPABASE_KEY='your-anon-or-publishable-key';
```

These two values are meant to be public; the rules in step 2 are what protect your library. Never share the **service_role** or **secret** key.

## 5. Move your books in

1. Open your site and click **Owner sign in** at the bottom.
2. If this browser still has books from before, click **Upload … books saved in this browser**. Or use **Import backup** with a backup file, or **Import from Goodreads or StoryGraph**.
