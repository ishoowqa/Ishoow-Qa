# Aria — AI Chat Application

A complete, production-quality AI chat app: streaming responses from OpenAI
and Gemini, real authentication and persistence via Supabase, file & image
attachments, voice input, full English/Arabic (RTL) support, and a dark/light
theme — built with Next.js 14, TypeScript, and Tailwind CSS.

Every control in the UI is wired to a real implementation. Nothing here is a
static mock: sending a message calls a real streaming API route, attaching a
file uploads to real object storage, and settings write to a real database
row.

## 1. Prerequisites

- Node.js 18.18+ (20 LTS recommended)
- A free [Supabase](https://supabase.com) project
- An API key from [OpenAI](https://platform.openai.com/api-keys) and/or
  [Google AI Studio](https://aistudio.google.com/apikey) (Gemini) — you only
  need one to get chatting; models without a key simply show as unavailable
  in the model selector instead of failing silently.

## 2. Set up Supabase

1. Create a new project at [supabase.com](https://supabase.com) (the free
   tier is enough).
2. Open **SQL Editor** in the Supabase dashboard, paste the contents of
   [`supabase/schema.sql`](./supabase/schema.sql), and run it. This creates:
   - `user_settings`, `conversations`, `messages`, `attachments` tables
   - Row Level Security policies so users can only ever read/write their own
     data
   - A trigger that creates a `user_settings` row automatically on signup
   - A trigger that bumps `conversations.updated_at` whenever a message is
     added (drives the sidebar's "recently updated" ordering)
   - A private `attachments` Storage bucket with matching RLS policies
3. Go to **Project Settings → API** and copy:
   - `Project URL` → `NEXT_PUBLIC_SUPABASE_URL`
   - `anon public` key → `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `service_role` key → `SUPABASE_SERVICE_ROLE_KEY` (used only server-side,
     for account deletion)
4. (Optional) Under **Authentication → Email**, disable "Confirm email" if
   you want new signups to land straight in the app without a verification
   step during local development.

## 3. Configure environment variables

```bash
cp .env.example .env.local
```

Fill in the Supabase values from step 2, plus whichever AI provider key(s)
you have:

```bash
NEXT_PUBLIC_SUPABASE_URL=https://xxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...

OPENAI_API_KEY=sk-...       # powers the "Fast" and "Advanced" models
GEMINI_API_KEY=AIza...      # powers the "Balanced" model

NEXT_PUBLIC_SITE_URL=http://localhost:3000
```

You can also point the "Advanced" model at any OpenAI-compatible endpoint
(Groq, Together, OpenRouter, a local Ollama server, etc.) by setting
`OPENAI_COMPATIBLE_BASE_URL`, `OPENAI_COMPATIBLE_API_KEY`, and
`OPENAI_COMPATIBLE_MODEL` instead of relying on `OPENAI_API_KEY` for it — see
`src/lib/ai/registry.ts` for exactly how each of the three model slots
resolves to a provider.

## 4. Install and run locally

```bash
npm install
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) — you'll land on the
sign-up page. Create an account, and you're in.

## 5. Project structure

```
src/
  app/
    login/, signup/, forgot-password/, reset-password/   ← auth pages
    chat/                                                 ← chat UI (protected)
      [id]/page.tsx        ← a single conversation (server component)
      page.tsx             ← "new chat" landing view
    settings/               ← appearance, language, chat, account settings
    api/
      chat/route.ts         ← streaming completion (SSE) — the core AI route
      conversations/        ← CRUD + search
      messages/              ← create/edit/delete/like
      upload/route.ts       ← file upload to Supabase Storage
      settings/route.ts     ← read/update user preferences
      account/route.ts      ← account deletion
  components/
    chat/                   ← message list, bubbles, input, model selector…
    sidebar/                ← conversation list, search, user menu
    settings/, auth/, ui/   ← settings panel, auth shell, design-system primitives
  hooks/                    ← useChat, useConversations, useFileUpload, useSpeechRecognition…
  lib/
    ai/                     ← provider abstraction (OpenAI, Gemini, registry)
    supabase/                ← browser/server Supabase clients + auth middleware
    i18n/                    ← English/Arabic translation dictionary
    markdown.ts              ← Markdown → sanitized HTML with syntax highlighting
  contexts/                 ← theme, i18n, auth, toast, sidebar state
supabase/schema.sql          ← full database schema + RLS policies
```

## 6. How the AI provider architecture works

`src/lib/ai/registry.ts` defines three model "slots" shown in the UI — Fast,
Balanced, Advanced — each mapped to a provider (`src/lib/ai/openai-provider.ts`,
`gemini-provider.ts`) and a concrete model name. A slot's `available` flag is
computed from whether its required environment variable is set; the model
selector shows unavailable slots as disabled with an explanation instead of
letting you pick a model that will just fail.

To add a new model or swap providers, edit `buildModelDefinitions()` in
`registry.ts` — nothing else in the app references a specific model name.
Both providers implement the same `ChatProvider` interface
(`src/lib/ai/types.ts`) and stream responses as an async generator, so adding
a fourth provider means writing one new class and one new registry entry.

## 7. Deployment (free tier)

**Vercel** (recommended):

1. Push this repo to GitHub.
2. Import it at [vercel.com/new](https://vercel.com/new).
3. Add the same environment variables from `.env.local` in the Vercel
   project's **Settings → Environment Variables**.
4. Deploy. Vercel's free tier covers this project comfortably.
5. Set `NEXT_PUBLIC_SITE_URL` to your deployed URL, and add that URL to
   Supabase's **Authentication → URL Configuration → Redirect URLs** (needed
   for the password-reset email link to work in production).

**Cloudflare Pages**: works the same way via the `@cloudflare/next-on-pages`
adapter — set the same environment variables in the Pages project settings.

Supabase's free tier and Vercel's free tier are both sufficient to run this
app at small scale with no paid dependencies.

## 8. Notes on specific features

- **Streaming**: `/api/chat` streams via Server-Sent Events over a plain
  `fetch` (not `EventSource`, since it needs to POST a body). The client
  parser lives in `src/hooks/use-chat.ts`.
- **Stop generating**: aborts the client's `fetch`; the server still saves
  whatever partial text was generated before the abort, so nothing is lost.
- **Voice input**: uses the browser's native `SpeechRecognition` API
  (Chrome, Edge, Safari). Firefox doesn't implement it yet — the mic button
  shows a clear "not supported" message there instead of doing nothing.
- **Images**: image attachments are re-encoded to base64 server-side before
  being sent to either provider (see `/api/chat/route.ts`), since Gemini's
  API doesn't accept arbitrary external URLs.
- **"Save chat history" setting**: when turned off, the current conversation
  is deleted from the database the moment you navigate away from it — so it
  never persists past the viewing session.
- **RTL**: switching the language to Arabic flips `dir="rtl"` on `<html>`;
  every layout in the app uses Tailwind's logical-property utilities
  (`ps-`, `pe-`, `ms-`, `me-`, `start-`, `end-`) so spacing mirrors correctly
  rather than just the text direction.

## 9. What you need to configure yourself

- `OPENAI_API_KEY` and/or `GEMINI_API_KEY` — without at least one, the model
  selector will show every model as unavailable and chat will return a clear
  "not configured" error rather than pretending to work.
- `SUPABASE_SERVICE_ROLE_KEY` — required only for the account-deletion
  feature; every other feature works with just the anon key.
