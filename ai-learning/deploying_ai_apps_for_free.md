# Deploying AI Apps for Free — Streamlit Community Cloud Guide

*Part of my AI Learning Journey | Last updated: April 2026*
*v1 — written during the planning phase of the Search Ad Copy Evaluator project*

---

## What This Covers

Building an AI app locally is one thing. Getting it to a public URL that anyone can visit — reliably, persistently, and at zero cost — is another. This guide covers the full deployment path for a Python Streamlit app using Streamlit Community Cloud, including secrets management, GitHub integration, and the gotchas that catch solo builders.

This is the deployment stack chosen for the LLM Eval Toolkit after evaluating Heroku, Render, Railway, Hugging Face Spaces, and local ngrok tunnels. The rationale is documented in `llm_api_selection_for_solo_builds.md`.

---

## Why Streamlit Community Cloud

| Factor | Streamlit Community Cloud | Alternatives |
|--------|--------------------------|-------------|
| Cost | Free for public apps | Heroku starts at $5/month; Render free tier spins down |
| Setup | Connect GitHub repo → deploy. Done. | Others require Dockerfile or build config |
| Secrets | Built-in secrets manager — keys never in code | Must handle manually elsewhere |
| Deployment | Auto-deploys on every push to main | Requires manual trigger on most platforms |
| Domain | `yourapp.streamlit.app` — persistent URL | ngrok URL changes on every restart |
| Framework lock-in | Streamlit only | Render/Railway support any framework |

The only meaningful downside: **free tier apps go to sleep after inactivity.** First visitor after a sleep period waits 30–60 seconds for the app to wake up. This is documented behaviour, not a bug. Handle it with a loading message and a README note.

---

## Prerequisites

Before deploying you need:
- A personal GitHub account
- Your app code in a **public** GitHub repo (Community Cloud requires public repos on the free tier)
- A `requirements.txt` in the repo root listing all dependencies
- A working local version of the app (`streamlit run app/main.py` with no errors)
- Your API key ready (Gemini, OpenAI, or whichever provider you use)

---

## Step-by-Step Deployment

### Step 1 — Prepare the repo

Your repo must have this structure at minimum:

```
your-repo/
├── app/
│   └── main.py          # Your Streamlit app entry point
├── requirements.txt     # All dependencies
└── README.md
```

**requirements.txt** must be complete and accurate. If it's missing a library the app uses, the deployment will fail.

Example for a Gemini-powered Streamlit app:
```
streamlit
google-generativeai
python-dotenv
```

Verify locally by creating a fresh virtual environment, installing from `requirements.txt`, and running the app. If it works in a clean environment, it will work on Streamlit Community Cloud.

---

### Step 2 — Set up .gitignore before anything else

Before your first commit, create `.gitignore` in the repo root:

```
# Environment
.env
.venv/
venv/
__pycache__/

# Streamlit secrets (local only)
.streamlit/secrets.toml

# OS
.DS_Store
Thumbs.db
```

**This is critical.** If you commit `.env` or `secrets.toml` with API keys inside, those keys are exposed publicly even if you delete the file in a later commit — git history preserves it. Set up `.gitignore` before writing your first API key anywhere.

---

### Step 3 — Set up local secrets

For local development, create `.streamlit/secrets.toml` in your project root:

```toml
GOOGLE_API_KEY = "your_key_here"
```

Access in your app code:
```python
import streamlit as st
api_key = st.secrets["GOOGLE_API_KEY"]
```

This file is excluded from git via `.gitignore`. It only exists on your local machine.

For environment variable fallback during testing:
```python
import os
api_key = st.secrets.get("GOOGLE_API_KEY") or os.environ.get("GOOGLE_API_KEY")
```

---

### Step 4 — Create a Streamlit Community Cloud account

1. Go to [share.streamlit.io](https://share.streamlit.io)
2. Sign in with your GitHub account
3. Authorise Streamlit to access your repositories

No credit card. No billing information. The free tier is genuinely free for public apps.

---

### Step 5 — Deploy the app

1. In Streamlit Community Cloud, click **New app**
2. Select your GitHub repo
3. Set the branch: `main`
4. Set the main file path: `app/main.py`
5. Click **Advanced settings** → open the **Secrets** section
6. Paste your secrets in TOML format:
   ```toml
   GOOGLE_API_KEY = "your_actual_key_here"
   ```
7. Click **Deploy**

Deployment takes 2–5 minutes on first run. Streamlit installs your `requirements.txt` dependencies and starts the app.

---

### Step 6 — Verify the live URL

Once deployed:
- Your app is live at `https://[your-github-username]-[repo-name]-[random-suffix].streamlit.app`
- Open in a **private browser window** to test without cached state
- Run your complete smoke test sequence (document this in your launch checklist)
- Update your README with the live URL

---

### Step 7 — Continuous deployment

Every push to the `main` branch triggers an automatic redeploy. This means:
- Fix a bug → commit → push → app updates within 2–3 minutes
- No manual deployment steps after initial setup

**One gotcha:** If you push broken code, the app will fail to start and show an error page. Always test locally before pushing to main. For a solo build, this is sufficient quality control.

---

## Secrets Management — The Full Picture

| Environment | Where the key lives | How the app reads it |
|-------------|--------------------|--------------------|
| Local development | `.streamlit/secrets.toml` (gitignored) | `st.secrets["KEY_NAME"]` |
| Streamlit Community Cloud | Secrets section in app settings dashboard | `st.secrets["KEY_NAME"]` (same code) |
| CI/testing (if needed) | Environment variable | `os.environ.get("KEY_NAME")` |

The same `st.secrets["KEY_NAME"]` call works in both local and production environments. Zero code changes between environments.

**To rotate a key:**
1. Generate new key in Google AI Studio (or whichever provider)
2. Update `.streamlit/secrets.toml` locally
3. Update the Secrets section in Streamlit Community Cloud dashboard
4. Revoke the old key in the provider console

---

## Handling the Cold Start Problem

Free tier apps go to sleep after ~7 days of inactivity. First visitor after sleep waits 30–60 seconds.

**Mitigation options:**

1. **Clear loading message** — add a spinner or status message so users know the app is waking up, not broken:
   ```python
   with st.spinner("Loading evaluator — this may take a moment on first visit..."):
       # your initialization code
   ```

2. **README note** — document the cold start behaviour clearly so it's not mistaken for an error

3. **Manual warm-up** — if you know someone important is visiting (e.g. a hiring manager you just shared the link with), visit the URL yourself first to wake it up

4. **Streamlit Teams plan** — paid plan eliminates sleep mode. Not worth it for a portfolio project but good to know the option exists.

---

## Common Deployment Failures

| Error | Most likely cause | Fix |
|-------|------------------|-----|
| `ModuleNotFoundError` | Library missing from `requirements.txt` | Add the missing library and push |
| `FileNotFoundError` | Wrong main file path set during deploy | Check the path in app settings — should be `app/main.py` |
| `KeyError: 'YOUR_KEY'` | Secret not added to Streamlit dashboard | Add the key in App Settings → Secrets |
| App loads but API calls fail | Wrong key value in Streamlit secrets | Verify the key in dashboard — no extra spaces or quotes |
| App shows old version after push | Deployment still in progress | Wait 3–5 minutes and hard refresh |
| 429 rate limit errors | Free tier API quota exhausted | Check your API provider's daily limit reset time |

---

## Post-Deployment Checklist

- [ ] Live URL opens correctly in a private browser window
- [ ] All app tabs load without errors
- [ ] API calls return real responses (not mock data)
- [ ] Secrets are not visible anywhere in the app UI or page source
- [ ] README updated with live URL
- [ ] Cold start behaviour documented in README

---

## Further Reading

- [Streamlit Community Cloud documentation](https://docs.streamlit.io/deploy/streamlit-community-cloud)
- [Streamlit secrets management](https://docs.streamlit.io/deploy/streamlit-community-cloud/deploy-your-app/secrets-management)
- [LLM API Selection for Solo Builds](./llm_api_selection_for_solo_builds.md)

---

*Written as part of my public AI learning journey. I am a Senior TPM and Designated PM at Microsoft AI, building real AI products and documenting what I learn. See the `llm-eval-toolkit` repo for a hands-on project applying these concepts.*
