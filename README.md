# Offline AI

A personal AI workspace with an Ollama chat, saved conversations, appearance and chat preferences, file summaries, voice input, optional accounts, image generation, and browser based motion clips. It is packaged as an installable web app for the computer or phone that can reach the server.

## Move it to another device

The interface can be installed from a browser using its **Install app** option when available. The installable shell is cached for quick reopening, but chat, accounts, image generation, and saved history still need the Python server and their local services to be running. Installing it on another device does not move your account database or Ollama model there.

To run the app on another computer, copy the source folder, install Python and Ollama there, pull a model, then follow the local setup below. Chat history and accounts are stored in `chat.db`; generated images are in `generated/`; voice models are in `models/`. Back up these folders only to storage you trust. Do not copy `.env` into a public repository.

The Ollama endpoint can be changed with `OLLAMA_BASE_URL` in `.env`, for example when Ollama runs on the same private network at another address. Do not expose Ollama or this development server directly to the public internet.

## Run it locally

1. Install [Ollama](https://ollama.com/download) for your operating system.
2. Download the small default chat model once:

   ```sh
   ollama pull gemma3:1b
   ```

   The app uses `gemma3:1b` by default. For an optional larger custom model, download Qwen3 4B (about 2.5 GB) and build the model recipe:

   ```sh
   ollama pull qwen3:4b
   ollama create ameer-offline-ai -f Modelfile
   ```

   This creates a named, customized model based on Qwen3 4B; it does not train the underlying model from scratch. To use it in the app, add `OLLAMA_MODEL=ameer-offline-ai` to `.env` and restart the app. On computers with limited memory, Qwen3 4B may respond slowly.

3. Create the app's virtual environment. The chat server itself uses only Python's standard library. Install the optional local voice transcription library only if you want microphone transcription:

   ```sh
   python3 -m venv .venv
   .venv/bin/pip install -r requirements.txt
   .venv/bin/pip install -r requirements-voice.txt  # optional voice input
   ```

4. Start Ollama, then start the app:

   ```sh
   .venv/bin/python server.py
   ```

5. Open <http://127.0.0.1:8000> in your browser and allow microphone access when prompted.

If you already have the app running, stop the old server with **Ctrl+C**, start it again with the command above, then refresh the browser.

On first voice use, Whisper's small multilingual speech model downloads into `models/`. After it is downloaded, microphone transcription runs locally. Turn on **Speak replies aloud** to hear answers using a voice available in your browser or operating system.

## Ask about an image in chat

Use the **＋** button beside the message box to attach a PNG, JPEG, or WebP image up to 4 MB, then ask a question or leave the message blank for a description. Image questions are sent to Cloudflare Workers AI's hosted vision model; the local Gemma 3 1B model cannot inspect images. The app does not save the uploaded image in chat history, but the image is sent to Cloudflare to answer the question. Image questions and image generation share the app's daily limits and the Cloudflare account's free Workers AI allowance. Requests can pause when either allowance is used up. If the Cloudflare account is on a Paid plan, usage above the included allowance can incur charges. Read [Cloudflare's Workers AI data usage](https://developers.cloudflare.com/workers-ai/platform/data-usage/) and [free allowance details](https://developers.cloudflare.com/workers-ai/platform/pricing/) before using private images.

## Summarize study files

Use **＋** to attach one PDF, TXT, Markdown, CSV, DOCX, PPTX, or supported audio file (MP3, M4A, WAV, WebM, OGG, OPUS, FLAC, or AAC), up to 20 MB. Add a request such as “Make exam notes,” or leave it blank for a study guide with a summary, key terms, and practice questions. In local Ollama mode, extraction, audio transcription, and summarization run locally. In public Cloudflare mode, extracted study text is sent to Cloudflare Workers AI for the summary; uploaded file contents are not stored in chat history. The app summarizes up to the first 10,000 extracted characters. PDF reading uses Poppler's `pdftotext` program; scanned PDFs without selectable text need OCR before upload.

## Optional accounts and guest use

Visitors can use the app as guests; signup is optional and does not unlock extra features. Guest sessions and registered accounts have separate chat histories. **New chat** creates a separate saved conversation; use **History** in the sidebar to reopen a prior conversation. Existing messages are migrated into a conversation on the first start after this feature is installed. New accounts use email verification only; SMS signup is disabled. Passwords must be at least 8 characters and include uppercase and lowercase letters, a number, and a symbol. A one-time six-digit code expires after 5 minutes; failed attempts and resend requests are limited. The sign-in screen includes password reset by email; older phone-based accounts can still use texted reset codes. A successful reset signs out other sessions. Passwords are stored as salted PBKDF2 hashes and OTPs are stored as keyed hashes. The local prototype stores sessions, account records, pending signups and resets, and chat history in `chat.db`.

The sidebar **Settings** page saves theme, accent, text size, Enter-to-send, speech, and reduced-motion preferences in the current browser. It can export conversations as JSON or delete the currently open conversation after confirmation.

Email signup and password resets require the site operator to configure `SMTP_HOST` and `SMTP_FROM`; optional `SMTP_PORT` (default `587`), `SMTP_USERNAME`, and `SMTP_PASSWORD` enable authenticated SMTP. Existing phone-based accounts can use texted password resets with a Twilio account and `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, and `TWILIO_FROM_NUMBER`. Set provider settings as environment variables or in the private `.env` file next to `server.py`, then restart the server. SMS provider charges may apply. The OTP signing key is created locally in `.otp_secret`; for a hosted multi-instance deployment, configure the same private `OTP_SECRET` value on each instance and persist it safely.

Example `.env` entries (fill in only the provider you choose):

```text
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USERNAME=your-smtp-user
SMTP_PASSWORD=your-smtp-password
SMTP_FROM=Offline AI <noreply@example.com>

TWILIO_ACCOUNT_SID=your-account-sid
TWILIO_AUTH_TOKEN=your-auth-token
TWILIO_FROM_NUMBER=+15551234567
```

## Free image generation and motion clips

Chat and voice run locally. Image generation sends only the prompt to an online image service using the operator's server-side credentials; visitors do not need API keys. A shared free daily limit is enforced by the app, and usage can pause when it is reached. The hosting account must stay on its Free plan; a paid account can incur charges above its included allowance. See [current plan limits and pricing](https://developers.cloudflare.com/workers-ai/platform/pricing/).

1. Create a free Cloudflare account and open Workers AI in its dashboard.
2. Use **Use REST API** to create a Workers AI API token and copy the account ID. Cloudflare documents the token setup [here](https://developers.cloudflare.com/workers-ai/get-started/rest-api/).
3. In the same folder as `server.py`, the site operator creates `.env` with these two lines, replacing the example values:

   ```text
   CLOUDFLARE_ACCOUNT_ID=your_32_character_account_id
   CLOUDFLARE_API_TOKEN=your_workers_ai_token
   ```

4. Restart the app with `.venv/bin/python server.py`, open the local page, and choose **Image & animation**.

Keep `.env` private; it is excluded from Git. The app uses the FLUX.1 schnell model with four steps. Cloudflare's daily free allowance is shared with any other Workers AI use on that account, so image generation can stop early if other apps use it. The app's default caps are two image requests per guest/account per day and 30 per day app-wide; the owner console can change these or set either cap to 0 for no app-imposed cap. Cloudflare's own quotas, model availability, account plan, and host resources still apply and may limit requests or incur charges.

The motion clip feature runs in your browser and makes a five-second slow zoom or pan video from a still image. It does not create new AI-generated motion. The image stays on your computer, and the clip downloads as a WebM file. No paid service or API key is used for this feature.

## Owner admin dashboard

The app includes a private owner console at `http://127.0.0.1:8000/admin`. It checks whether the Ollama API can be reached and whether image/email credentials are configured (configured does not guarantee a provider will accept them); shows account, chat, image, and storage totals; lets the owner enable or disable signup and online image features, select an already-installed Ollama model, set context size and response temperature, edit assistant instructions, and adjust per-user and global daily image caps (0 means no app-imposed cap); searches and deletes accounts with their chat, sessions, usage records, and generated images; removes expired verification/session records; and can reset today's image counters. Deletion and usage reset require an explicit confirmation in the console. Provider quotas, available hardware, and plan billing are still enforced outside the app. The dashboard password is an operator secret; never add it to GitHub or share it with users. Its fields support browser password managers. After rotation, the salted password hash is stored in `chat.db` and takes precedence over `ADMIN_PASSWORD` in `.env`; protect database backups because they include account records and this admin credential hash.

If you forget a rotated admin password, the private `ADMIN_PASSWORD` in `.env` can be restored as the login by removing the saved password hash and admin sessions from the database. From the project directory, run:

```sh
python3 -c 'import sqlite3; db=sqlite3.connect("chat.db"); db.execute("DELETE FROM app_settings WHERE key IN (?,?)", ("admin_password_hash", "admin_password_salt")); db.execute("DELETE FROM admin_sessions"); db.commit(); db.close()'
```

Then sign in with the value in `.env` and set a replacement from the console.

Before starting the app, add a private `ADMIN_PASSWORD` entry to `.env` beside `server.py`. Use a unique password with at least 16 characters. Generate one locally with:

```sh
python3 -c 'import secrets; print(secrets.token_urlsafe(32))'
```

Copy the generated value into `.env` as `ADMIN_PASSWORD=your-generated-value`, then restart the server and open `/admin`. Keep `.env`, `chat.db`, and backups private. If the password or computer is lost, an admin password kept only there cannot be recovered; store a secure copy in a password manager. For a hosted HTTPS deployment, set `COOKIE_SECURE=1`, store the admin password in the host's secret settings, and persist the database on protected storage with backups.

GitHub stores the source code; it does not keep this Python server online or provide the dashboard's database. The included Render Blueprint can host a temporary free demo over HTTPS. Its data persistence limits are described below.

## Publishing readiness

The repository includes a Render Blueprint for a **free demo**. That demo uses Cloudflare Workers AI for text chat and image features instead of local Ollama. Prompts, attached images, and extracted study text are sent to Cloudflare for model responses. Configure the Cloudflare account ID and API token as private Render environment variables; never add their values to GitHub. Cloudflare's free Workers AI allowance is shared across the account and can be exhausted. See [Workers AI pricing and limits](https://developers.cloudflare.com/workers-ai/platform/pricing/).

The free Render demo sleeps after inactivity and its filesystem is temporary. Chat histories, accounts, admin settings, and generated image files can disappear after a restart, sleep, or redeploy. Signup is disabled by default in Cloudflare mode. PDF summaries need Poppler (`pdftotext`), and local audio transcription needs the optional voice dependencies and model files; these are not installed in the free demo configuration. This setup is for trying the app, not for accounts or data that must persist. A reliable public service needs durable database and file storage, backups, abuse controls, and ongoing provider/host quota management.

### Deploy the free Render demo

1. Rotate any Gmail or Cloudflare credentials that have been shared, then sign in to Render and choose **New → Blueprint**.
2. Connect the public GitHub repository `sulaimanaliyu2009m-hub/offline-ai` and deploy the Blueprint. Render reads `render.yaml` and asks for `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_API_TOKEN`, and `ADMIN_PASSWORD` as private values. Use a fresh Workers AI token with AI permissions and a unique admin password of at least 16 characters.
3. Wait for the deploy to finish, then open the `onrender.com` address shown in the Render dashboard. Use `/admin` for the private owner dashboard.
4. Keep signup disabled for this temporary demo; the free service does not keep its SQLite data across restarts. Do not store important chats or account data there.

The app uses Render's assigned `PORT`, binds publicly only when hosted, and sets secure cookies in that hosted mode. Local startup remains on `127.0.0.1`. Render documents its [Blueprint deploy flow](https://render.com/docs/blueprint-spec) and [free-service limitations](https://render.com/docs/free).

### Publish this source on GitHub

1. Sign in to GitHub and create a new repository. Choose a repository name and visibility. Since this folder already has project files, do not initialize the new repository with a README, license, or `.gitignore`.
2. In a terminal, go to this project folder and stage only the public source and setup files:

   ```sh
   cd ~/Documents/ChatGPT/v
   git add .gitignore .env.example Modelfile README.md requirements.txt requirements-voice.txt server.py
   git diff --cached --name-only
   ```

   Check the staged file list. It should not contain `.env`, `chat.db`, `.otp_secret`, `.venv`, `models/`, or `generated/`.
3. Commit and connect the repository GitHub created for you (replace the sample URL with your own repository URL):

   ```sh
   git commit -m "Prepare Offline AI for GitHub"
   git branch -M main
   git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPOSITORY.git
   git remote -v
   git push -u origin main
   ```

   GitHub may ask you to authenticate in a browser. Never paste an access token into chat or commit it to a file.

There is no license file yet. Before inviting others to reuse, modify, or distribute this code, choose and add a license that matches your intent. GitHub's guide covers [creating a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-new-repository) and [pushing locally hosted code](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github).

## Current scope

This remains a prototype. Local mode uses Python's built-in HTTP server bound to `127.0.0.1`. The Render Blueprint exposes a free demo over HTTPS but does not provide durable SQLite storage or production abuse protection.
