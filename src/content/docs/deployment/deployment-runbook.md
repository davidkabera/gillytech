---
title: "Gillytech Deployment Runbook"
description: Companion to `ADR-0001-fullstack-architecture.md`. This runbook covers the
            concrete steps to stand up and redeploy the stack Supabase Cloud (DB +
            Auth), the Express API on Render, and the React/Vite frontend on Vercel —
            plus the specific incidents encountered during migration and how they were
            resolved.
---

---

## 1. Prerequisites

- Supabase Cloud project created.
- GitHub repo `cto-kabera/gillytech` (monorepo: `frontend/`, `backend/`).
- Render account connected to the GitHub repo.
- Vercel account connected to the GitHub repo.

---

## 2. Environment Variables Reference

| Variable | Used by | Where set |
|---|---|---|
| `SUPABASE_URL` | Backend | Render env vars |
| `SUPABASE_JWT_SECRET` | Backend | Render env vars |
| `VITE_SUPABASE_URL` | Frontend | Vercel env vars |
| `VITE_SUPABASE_ANON_KEY` | Frontend | Vercel env vars |
| `VITE_API_URL` | Frontend | Vercel env vars — set to the live Render URL |

> The `SUPABASE_JWT_SECRET` on Render **must** match the JWT secret shown in
> the Supabase project's API settings, not a locally generated one. This was
> the root cause of Incident 3 below.

---

## 3. Database & Auth Setup (Supabase Cloud)

### 3.1 Seed initial users directly in `auth.users`

Used to create the admin/teacher/student accounts without going through the
signup flow:

```sql
insert into auth.users (
  instance_id, id, aud, role, email,
  encrypted_password, email_confirmed_at,
  raw_app_meta_data, raw_user_meta_data,
  created_at, updated_at,
  confirmation_token, recovery_token,
  email_change_token_new, email_change,
  is_super_admin, is_sso_user
) values (
  '00000000-0000-0000-0000-000000000000',
  gen_random_uuid(),
  'authenticated',
  'authenticated',
  'admin@gillytech.dev',
  crypt('CHANGE_ME', gen_salt('bf')),
  now(),
  '{"provider":"email","providers":["email"]}',
  '{"role":"admin"}',
  now(), now(),
  '', '', '', '',
  false, false
);
```

Repeat for `teacher@gillytech.dev` and `amara@gillytech.dev` (student) with
appropriate `raw_user_meta_data`.

**Note the empty-string values for `confirmation_token`, `recovery_token`,
`email_change_token_new`, `email_change`, and explicit `false` for
`is_super_admin` / `is_sso_user`.** Omitting these (leaving them `NULL`) is
what caused Incident 1 below — always include them on manual inserts.

### 3.2 Verify the `handle_new_user` trigger

Confirm the trigger that syncs `auth.users` → `public.users` fired correctly
for each seeded user:

```sql
select id, email, role from public.users
where email in (
  'admin@gillytech.dev', 'teacher@gillytech.dev', 'amara@gillytech.dev'
);
```

If rows are missing here despite existing in `auth.users`, the trigger
either didn't fire (check it's attached to `auth.users` `AFTER INSERT`) or
errored silently — check Supabase's Postgres logs.

### 3.3 Grant table permissions + RLS policy

```sql
grant usage on schema public to authenticated;
grant select on public.users to authenticated;

alter table public.users enable row level security;

create policy "Authenticated users can read users"
on public.users
for select
to authenticated
using (true);
```

Adjust the `using` clause to scope reads (e.g. `using (auth.uid() = id)`) if
open `SELECT` access is too broad for production.

### 3.4 Configure Auth URLs for production

In Supabase Dashboard → Authentication → URL Configuration:
- **Site URL**: the live Vercel domain.
- **Redirect URLs**: add `https://<vercel-domain>/*` and
  `http://localhost:5173/*` (and `4173` if previewing a production build
  locally).

---

## 4. Backend Deployment (Render)

1. Create a new Web Service on Render, pointed at the GitHub repo with
   **Root Directory** set to `backend/`.
2. Set the build/start commands appropriate to the Express app (e.g.
   `npm install` / `npm start`).
3. Add environment variables: `SUPABASE_URL`, `SUPABASE_JWT_SECRET`, and any
   others the API needs.
4. Confirm CORS in `index.js` allows the production frontend origin in
   addition to local dev ports:

```js
const allowedOrigins = [
  'http://localhost:5173',
  'http://localhost:4173',
  'https://<your-vercel-domain>.vercel.app',
];

app.use(cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
}));
```

5. Deploy and note the live backend URL (e.g.
   `https://gillytech.onrender.com`).

---

## 5. Frontend Deployment (Vercel)

1. Import `cto-kabera/gillytech` into Vercel.
2. Set **Root Directory** to `frontend/`.
3. Add environment variables: `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`,
   `VITE_API_URL` (pointed at the Render backend URL from step 4.5).
4. Deploy. Vercel will build and redeploy automatically on every push to the
   connected branch.

---

## 6. Git & Repository Hygiene

- Root `.gitignore` excludes `.supabase/` and root `node_modules/`.
- `frontend/.gitignore` and `backend/.gitignore` each exclude `.env` and
  their own `node_modules/`.
- If a large binary (e.g. a cached `@supabase/cli` binary) ends up tracked
  and blocks a push (`>100MB` limit):

```bash
git rm -r --cached node_modules
git commit --amend --no-edit
git push --force-with-lease
```

  Use `--force-with-lease` rather than `--force` to avoid clobbering remote
  history you haven't fetched.

---

## 7. Troubleshooting Runbook (Incidents Encountered)

### Incident 1 — GoTrue `unexpected_failure` on login
**Symptom**: Auth crashed with `unexpected_failure` for manually-seeded
users.
**Cause**: Raw SQL inserts into `auth.users` left internal token columns
`NULL` (`confirmation_token`, `recovery_token`, `email_change_token_new`,
`email_change`) and `is_super_admin` / `is_sso_user` unset.
**Fix**: Patch the affected rows:

```sql
update auth.users
set confirmation_token = '',
    recovery_token = '',
    email_change_token_new = '',
    email_change = '',
    is_super_admin = false,
    is_sso_user = false
where confirmation_token is null
   or recovery_token is null
   or email_change_token_new is null
   or email_change is null;
```
**Prevention**: Always include these fields explicitly on manual
`auth.users` inserts (see §3.1).

### Incident 2 — `Error 42501` on `public.users` reads
**Symptom**: Authenticated requests to read `public.users` failed with
Postgres permission error 42501.
**Cause**: `authenticated` role lacked `USAGE` on schema `public` and
`SELECT` on `public.users`; RLS was enabled with no policy, or disabled
without replacement grants.
**Fix**: See §3.3 — grant `USAGE`/`SELECT` and add an explicit `SELECT`
policy for `authenticated`.

### Incident 3 — `Invalid or expired token` from the backend
**Symptom**: Frontend sessions valid per Supabase, but backend rejected
every request as an invalid token.
**Cause**: `backend/.env` had a locally-generated JWT secret instead of the
actual Supabase Cloud project's JWT secret.
**Fix**: Copy the JWT secret from Supabase Dashboard → Settings → API →
JWT Secret into `SUPABASE_JWT_SECRET` (both local `.env` and the Render env
var). Redeploy the backend after changing.
**Prevention**: Treat the Supabase JWT secret as the single source of
truth; never generate a separate one for the backend.

### Incident 4 — Frontend crash: `can't access property "role", user is null`
**Symptom**: Runtime crash while auth state was still resolving.
**Cause**: Components read `user.role` before the async auth check
completed, when `user` was still `null`.
**Fix**: Use optional chaining (`user?.role`) and add a loading guard that
defers rendering until the auth state has resolved.

### Incident 5 — Push rejected for file `>100MB`
**Symptom**: `git push` rejected by GitHub for exceeding the 100MB file
size limit.
**Cause**: `@supabase/cli` binaries inside `node_modules` were committed
before `.gitignore` was in place.
**Fix**: See §6 — `git rm -r --cached node_modules`, amend, force-push with
lease.

---

## 8. Post-Deploy Verification Checklist

- [ ] `admin@gillytech.dev`, `teacher@gillytech.dev`, `amara@gillytech.dev`
      can all log in on the live Vercel URL.
- [ ] Frontend successfully calls the Render backend (check network tab for
      200s, not CORS errors).
- [ ] `public.users` reads succeed for an authenticated session (no 42501).
- [ ] Backend logs show no `Invalid or expired token` errors on
      authenticated requests.
- [ ] `git status` on a fresh clone shows no `.env` or `node_modules`
      tracked.
