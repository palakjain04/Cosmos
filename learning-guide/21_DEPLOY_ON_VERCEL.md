# Lesson 21 — Deploy on Vercel, Step by Step

**Time:** 30–45 minutes | **You need:** Lesson 20 Steps 1–5 done (configurable API, rehearsal passed, `my_cosmos` pushed to GitHub), a free **Vercel** account

## Goal

By the end of this lesson your COSMOS runs **entirely on Vercel**, as two projects made from your one GitHub repository:

| Vercel project    | Folder        | Address (example)                    |
| ----------------- | ------------- | ------------------------------------ |
| `my-cosmos-api` | `backend/`  | `https://my-cosmos-api.vercel.app` |
| `my-cosmos`     | `frontend/` | `https://my-cosmos.vercel.app`     |

Every `git push` then redeploys both automatically.

## Understand

### One repository, two Vercel projects

Vercel lets you import the **same** repository more than once. Each import is its own **project** with its own **Root Directory**. One project points at `frontend/`, the other at `backend/`. They build and deploy separately, but you push code only once.

### How Vercel runs the website

The website is plain static files after `npm run build`. Vercel recognises Vite (the **Vite** framework preset), builds it, and serves `dist/` from its worldwide network (a **CDN**). A small `vercel.json` adds the **rewrite rule** from Lesson 20, so a refresh on `/papers/demo-00100` still works.

### How Vercel runs the API

Vercel recognises **FastAPI** by itself. It finds `fastapi` in `requirements.txt` and looks for a FastAPI object named **`app`** in a file like `app/main.py`. That's exactly what you've had since Lesson 11.

The API runs as a **function**: Vercel starts it when a request arrives and may stop it when it's quiet. Two things follow from that:

- **Cold start:** after a quiet period, the first request runs all the startup code again: loading the papers, building the search index, scoring opportunities. That first request takes several seconds. Later ones are fast.
- **No disk writes:** the API only **reads** `data/papers.json`, which is fine. Never try to save files at runtime.

Limits on the free (**Hobby**) plan, from Vercel's docs: Python functions up to **500 MB**, **2 GB** memory, **300 seconds** per request. Your API needs far less.

### Environment variables need a redeploy

On Vercel, a new or changed environment variable only affects **new** deployments. After changing one: **Redeploy**. For the website this matters twice over, because `VITE_` values are baked in at **build** time.

## Do it now

### Step 1 — Add the Vercel files

**1a. The website's rewrite rule.** This exact file comes from Vercel's Vite guide.

**File:** `my_cosmos/frontend/vercel.json`

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ]
}
```

Real files such as `/assets/index-abc.js` and `/data/papers.json` still load normally: Vercel serves existing files **before** it applies rewrites.

**1b. Pin the Node.js version** to the one you've used all course. In the frontend terminal (`my_cosmos\frontend`):

```powershell
npm pkg set engines.node=22.x
```

**1c. Pin the Python version.** Vercel reads it from a `.python-version` file. 3.12 matches the package versions in `requirements.txt`.

**File:** `my_cosmos/backend/.python-version`

```text
3.12
```

**1d. Push.** In a terminal inside `my_cosmos`:

```powershell
git add .
```

```powershell
git commit -m "Vercel configuration"
```

```powershell
git push
```

### Step 2 — Sign in to Vercel

1. Go to **https://vercel.com** and click **Sign Up** (or **Log In**).
2. Choose **Continue with GitHub**, and allow Vercel to see your repositories when GitHub asks. You can limit it to just `my_cosmos`.
3. Choose the free **Hobby** plan if asked.

### Step 3 — Deploy the API

1. In the Vercel dashboard click **Add New…** (or **Create New**) → **Project**.
2. Under **Import Git Repository**, find **my_cosmos** and click **Import**.
3. **Project Name:** type `my-cosmos-api`.
4. **Root Directory:** click **Edit**, choose **`backend`**, click **Continue**.
5. **Framework Preset:** it should show **FastAPI**. If it shows **Other**, leave it; Vercel still detects FastAPI from `requirements.txt` while building.
6. Open **Environment Variables** and add:

   | Key                     | Value                                                  |
   | ----------------------- | ------------------------------------------------------ |
   | `COSMOS_CORS_ORIGINS` | `http://localhost:4173` (temporary, fixed in Step 5) |
7. Click **Deploy**. Wait until Vercel says the deployment is ready (1–3 minutes: installing scikit-learn takes a moment).
8. Open the project's page. Its address is shown under **Domains**, e.g. `my-cosmos-api.vercel.app`. If that name was taken, Vercel adds a suffix. **Copy your exact address.**
9. Open `https://my-cosmos-api.vercel.app/health` (use **your** address). The first visit may take a few seconds (cold start). You should see:

   ```json
   {"status":"ok","answer_provider":"extractive"}
   ```
10. Also try `/stats` and `/docs`.

### Step 4 — Deploy the website

1. Back in the dashboard: **Add New…** → **Project** → **Import** the **same** `my_cosmos` repository again.
2. **Project Name:** `my-cosmos`.
3. **Root Directory:** **Edit** → choose **`frontend`** → **Continue**.
4. **Framework Preset:** **Vite**. Leave the build and output settings as they are.
5. Open **Environment Variables** and add both:| Key                | Value                                                                                               |
   | ------------------ | --------------------------------------------------------------------------------------------------- |
   | `VITE_DATA_MODE` | `api`                                                                                             |
   | `VITE_API_URL`   | `https://my-cosmos-api.vercel.app` ← your API address from Step 3, **no `/` at the end** |
6. Click **Deploy**.
7. When it's done, copy the website's address under **Domains**, e.g. `my-cosmos.vercel.app`.

> **Only want the demo site (Lesson 20, Path A)?** Skip Step 3, and in this step add **no** environment variables. You're done after the deploy.

### Step 5 — Let the website talk to the API

The API still only allows `localhost:4173`. Change that to your real website:

1. Open the **my-cosmos-api** project → **Settings** → **Environment Variables**.
2. Next to `COSMOS_CORS_ORIGINS`, click the **⋯** menu → **Edit**.
3. Set the value to your website address, exactly: `https://my-cosmos.vercel.app` (with `https://`, no `/` at the end). **Save**.
4. Go to the project's **Deployments** tab → on the newest deployment click **⋯** → **Redeploy** → **Redeploy**.

### Step 6 — Check the live site

Open `https://my-cosmos.vercel.app` (yours) and go through this list:

- [ ] The Explore page shows **1,500** papers and the "Demo data" notice.
- [ ] Open a paper page, then press **F5**: it still loads (the rewrite works).
- [ ] Search *"AI techniques for detecting spacecraft faults"*: results with RELEVANCE boxes.
- [ ] The paper page shows three tabs of recommendations.
- [ ] Research Map, Trends and Opportunities load.
- [ ] Assistant: a cited answer; `zzzz qqqq` gives "not enough evidence".
- [ ] **F12 → Network:** requests go to `my-cosmos-api.vercel.app` with status **200**, and the Console shows no red CORS errors.

**Share the website address — that's your COSMOS, live.** 🚀

### Step 7 — Your everyday workflow from now on

1. Change code on your computer and test it as usual (`npm run dev:api` + `uvicorn`).
2. `git add .` → `git commit -m "..."` → `git push`.
3. Vercel rebuilds **both** projects. Watch progress in each project's **Deployments** tab.
4. Changed an **environment variable**? Redeploy that project (Step 5, point 4).

Never add the key to the **website** project: everything there ends up in the browser. Your site is public and each answer costs money, so set a spending limit in your Anthropic account.

## What just happened?

- **One repository, two projects.** Vercel built `frontend/` with Vite and served it from its CDN. It ran `backend/` as a Python function, found `app` in `app/main.py`, and installed `requirements.txt` with Python 3.12 (from `.python-version`).
- `vercel.json` told the CDN: "anything that isn't a real file → `index.html`", so React Router handles every address.
- `VITE_API_URL` was baked into the website at build time. That's why the **website** was deployed *after* the API had an address.
- `COSMOS_CORS_ORIGINS` let the API accept requests from your website's address only. That's configuration, not code: no file changed.
- Each `git push` creates new deployments. Vercel keeps the old ones in the **Deployments** tab, so if something breaks you can go back to a deployment that worked.

## Try it yourself

1. Change the hero text, push, and watch the **my-cosmos** project redeploy by itself.
2. Open a **Preview** deployment (Vercel makes one for every branch you push). Does its API calling work? Why not? (Hint: its address isn't in `COSMOS_CORS_ORIGINS`.)
3. Open **my-cosmos-api** → **Logs** and load a few pages of your site. Find the cold start, where the first request is slower.
4. In **Settings** → **Domains**, add your own domain if you have one. Remember to update `COSMOS_CORS_ORIGINS` and `VITE_API_URL` afterwards, then redeploy both.

## If it does not work

| Problem                                                                  | Fix                                                                                                                                                                                                                                                   |
| ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **404: NOT_FOUND** when refreshing a page on the website           | `vercel.json` is missing, or it isn't **inside `frontend/`**. Push it and redeploy.                                                                                                                                                         |
| Every website page says **"Couldn't reach the research universe"** | Open `https://<your-api>/health` directly. If it works, `VITE_API_URL` is wrong or missing in the **website** project: fix it and **Redeploy** the website. If not, see the API rows below.                                           |
| First load of the day fails, a refresh works                             | A cold start. The API builds its indexes at startup (a few seconds). Retry, or add a Retry button (Lesson 20, Try it 3).                                                                                                                              |
| Console: **blocked by CORS policy**                                | `COSMOS_CORS_ORIGINS` in the **API** project must exactly match the address in the browser bar: `https://`, no trailing `/`, and the same domain (`my-cosmos.vercel.app`, not a preview URL). **Redeploy** the API after editing. |
| API deploy fails: **No FastAPI entrypoint found** (or similar)     | Root Directory must be **`backend`**, and `backend/app/main.py` must create `app = FastAPI(...)`.                                                                                                                                         |
| API deploy fails installing **numpy** / **scikit-learn**     | Check `backend/.python-version` says `3.12` and was pushed. The versions in `requirements.txt` don't support newer Pythons.                                                                                                                     |
| API page shows **FUNCTION_INVOCATION_FAILED**                      | Open **my-cosmos-api** → **Logs** and read the Python error. Usually a missing file: check `backend/data/papers.json` is on GitHub.                                                                                                    |
| Website deploy fails at `tsc`                                          | Run `npm run build` on your computer, fix the errors, push again.                                                                                                                                                                                   |
| Website still calls the old API address                                  | `VITE_` values are baked in at build time: after changing them, **Redeploy** the website.                                                                                                                                                     |
| Changed an environment variable but nothing changed                      | Every variable change needs a **Redeploy** of that project.                                                                                                                                                                                     |

---

**Back to:** [00_START_HERE.md](00_START_HERE.md)
