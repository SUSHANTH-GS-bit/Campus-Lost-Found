# Campus Lost & Found

A REVA campus lost-and-found website using plain HTML/CSS/JavaScript, Supabase Auth,
PostgreSQL, Storage and Realtime. The optional CLIP-based AI matching service is kept
separate from the website deployment.

## Project structure

```text
campus-lost-found/
├── web/
│   ├── index.html
│   ├── app.js
│   └── config.js
├── supabase/
│   └── schema.sql
├── ai_service/
│   ├── main.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .env.example
└── README.md
```

## IMPORTANT: fix for the 500 MB Vercel Function limit

The 5.5 GB deployment bundle is caused by packaging the optional AI service and its
`sentence-transformers` / PyTorch / CLIP dependencies as a serverless function.
Do **not** deploy `ai_service` as a Vercel Function.

Deploy only the `web` folder to Vercel:

1. Push this project to GitHub.
2. In Vercel, import the repository.
3. In **Project Settings → General → Root Directory**, choose `web`.
4. Framework Preset: **Other**. Leave Build Command and Output Directory empty/default.
5. Deploy. Vercel will serve the static HTML, CSS and JavaScript. No Python function or
   heavy ML dependencies are included in this deployment.
6. If Vercel was previously configured to use the repository root, change the Root
   Directory to `web` and redeploy. Remove any old root-level Python function or
   requirements file from the deployed project if one exists.

The website works without AI matching. In `web/config.js`, leave `AI_URL: ""`.
If you later want AI matching, deploy `ai_service` as a separate Docker service on a
container platform (for example Render, Railway or Google Cloud Run), then set its
public HTTPS URL as `AI_URL` in `web/config.js`. The AI service uses a large model and
needs a plan with sufficient RAM; it is not suitable for Vercel's 500 MB function
bundle limit.

## 1. Configure Supabase

1. Create a Supabase project.
2. Open **SQL Editor → New query**.
3. Paste and run the complete `supabase/schema.sql`.
4. In **Project Settings → API**, copy the Project URL and anon/public key.
5. Edit `web/config.js`:

```js
export const CONFIG = {
  SUPABASE_URL: "https://YOUR-PROJECT.supabase.co",
  SUPABASE_ANON_KEY: "YOUR-ANON-KEY",
  ALLOWED_DOMAIN: "reva.edu.in",
  AI_URL: ""
};
```

Use the anon/public key only in the browser. Never put the Supabase service-role key
in `web/config.js` or any frontend file.

The allowed email domain defaults to `reva.edu.in` in the SQL schema and frontend
configuration. Keep both values consistent if your university uses a different domain.

## 2. Run the website locally

Open a terminal in the `web` directory and run:

```bash
python -m http.server 8080
```

Then visit `http://localhost:8080`.

Do not open `index.html` directly as a `file://` URL; ES modules and browser security
may prevent the app from loading correctly.

## 3. Create an admin account

First sign up using an allowed university email. Then run this in Supabase SQL Editor,
replacing the email:

```sql
update public.profiles
set role = 'admin'
where id = (
  select id from auth.users where email = 'you@reva.edu.in'
);
```

Sign out and sign in again. The Moderation tab should appear.

## 4. Optional AI matching service (separate deployment)

The AI service is optional. The normal website features do not require it.

For local testing:

```bash
cd ai_service
python -m venv .venv
# Windows:
.venv\Scripts\activate
# macOS/Linux:
source .venv/bin/activate

pip install -r requirements.txt
```

Copy `.env.example` to `.env`, fill in the values, then run:

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

For Docker, build and run from the `ai_service` directory:

```bash
docker build -t campus-lost-found-ai .
docker run --env-file .env -p 8000:8000 campus-lost-found-ai
```

Set `AI_URL` in `web/config.js` to the deployed AI service's HTTPS base URL.
Keep `SUPABASE_SERVICE_KEY` only in the AI service environment.

## Features

- University-domain sign-up and sign-in.
- Lost/found item feed with search, category filters and sorting.
- Item reports with optional photo uploads.
- Hidden-detail verification for found-item claims.
- Claim-attempt limits enforced in the database.
- Finder/claimer contact details returned only to permitted users.
- Student item management and admin moderation.
- Live feed updates through Supabase Realtime.
- Optional AI similarity suggestions using CLIP embeddings.

## Security and deployment notes

- The SQL schema controls privileged actions through PostgreSQL functions and row-level
  security policies. Do not expose the Supabase service-role key in the browser.
- Public photo URLs are enabled by the supplied schema so photos can appear in the feed.
  Avoid uploading sensitive documents, ID numbers or identifiable faces.
- The included project has not been tested against your live Supabase project. Run the
  test checklist below before using it with real students.

## Test checklist

1. Sign-up with a non-university email should be refused.
2. Create two university accounts, A and B.
3. A posts a found item with a hidden detail; B sees it in the live feed.
4. B enters an incorrect hidden detail three times; another attempt should be locked.
5. B enters the correct detail; the item becomes Claimed and permitted contact details
   become available.
6. A third account must not see private contact details.
7. The poster marks the item Returned; an admin can see and moderate posts.
