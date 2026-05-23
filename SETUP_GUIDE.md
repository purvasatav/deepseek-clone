# DeepSeek Clone — Complete Setup Guide

A full-stack AI chat application built with **Next.js 15**, **MongoDB**, **Clerk Auth**, and the **DeepSeek API**.

---

## 📁 Project Structure

```
deepseek/
├── app/
│   ├── api/
│   │   ├── chat/
│   │   │   ├── ai/route.js        ← DeepSeek API call
│   │   │   ├── create/route.js    ← Create new chat
│   │   │   ├── delete/route.js    ← Delete a chat
│   │   │   ├── get/route.js       ← Fetch user's chats
│   │   │   └── rename/route.js    ← Rename a chat
│   │   └── clerk/route.js         ← Clerk webhook handler
│   ├── globals.css                ← Tailwind + custom styles
│   ├── layout.js                  ← Root layout (ClerkProvider)
│   ├── page.jsx                   ← Main chat UI
│   └── prism.css                  ← Code syntax highlighting
├── assets/                        ← SVG icons + QR code
├── components/
│   ├── ChatLabel.jsx              ← Sidebar chat item (rename/delete)
│   ├── Message.jsx                ← Chat message (Markdown + Prism)
│   ├── PromptBox.jsx              ← Input box + send logic
│   └── Sidebar.jsx                ← Sidebar with chat history
├── config/
│   └── db.js                      ← MongoDB connection (cached)
├── context/
│   └── AppContext.jsx             ← Global state (chats, selectedChat)
├── models/
│   ├── Chat.js                    ← Mongoose Chat schema
│   └── User.js                    ← Mongoose User schema
├── middleware.ts                  ← Clerk auth middleware
├── .env                           ← Environment variables (fill in yours)
├── package.json
└── next.config.mjs
```

---

## 🔧 Prerequisites

Make sure you have:
- **Node.js** v18+ installed → https://nodejs.org
- **npm** v9+ (comes with Node)
- A **MongoDB** database (free at https://mongodb.com/atlas)
- A **Clerk** account (free at https://clerk.com)
- A **DeepSeek** API key (https://platform.deepseek.com)

---

## 🚀 Step-by-Step Setup

### Step 1 — Clone / Extract the Project

If you received this as a ZIP, extract it:
```bash
unzip deepseek.zip
cd deepseek
```

### Step 2 — Install Dependencies

```bash
npm install
```

This installs all packages listed in `package.json`:
- `next` 15 + React 19
- `@clerk/nextjs` — authentication
- `mongoose` — MongoDB ORM
- `openai` — DeepSeek API client (uses OpenAI-compatible SDK)
- `axios` — HTTP client
- `react-markdown` — render AI responses as Markdown
- `prismjs` — code syntax highlighting
- `react-hot-toast` — toast notifications
- `svix` — Clerk webhook verification

---

### Step 3 — Set Up MongoDB Atlas

1. Go to https://mongodb.com/atlas and create a free account
2. Create a new **Cluster** (free M0 tier is fine)
3. Under **Database Access**, add a database user with a password
4. Under **Network Access**, add your IP (or `0.0.0.0/0` for dev)
5. Click **Connect → Drivers** and copy the connection string:
   ```
   mongodb+srv://<user>:<password>@cluster0.xxxxx.mongodb.net/deepseek?retryWrites=true&w=majority
   ```

---

### Step 4 — Set Up Clerk Authentication

1. Go to https://clerk.com and create a free account
2. Create a new **Application**
3. Choose **Email** and/or **Google** as sign-in methods
4. Go to **API Keys** in the Clerk dashboard and copy:
   - `Publishable Key` (starts with `pk_test_...`)
   - `Secret Key` (starts with `sk_test_...`)
5. Go to **Webhooks** → **Add Endpoint**:
   - URL: `https://your-domain.com/api/clerk` (use ngrok for local dev, see Step 7)
   - Events to subscribe: `user.created`, `user.updated`, `user.deleted`
6. Copy the **Signing Secret** from the webhook

---

### Step 5 — Get a DeepSeek API Key

1. Go to https://platform.deepseek.com
2. Sign up / log in
3. Navigate to **API Keys** and create a new key
4. Copy the key (starts with `sk-...`)

> 💡 DeepSeek uses the OpenAI-compatible API format. The project uses the `openai` npm package pointed at DeepSeek's base URL.

---

### Step 6 — Configure Environment Variables

Open the `.env` file in the project root and fill in your keys:

```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_your_publishable_key_here
CLERK_SECRET_KEY=sk_test_your_secret_key_here
MONGODB_URI=mongodb+srv://user:password@cluster0.xxxxx.mongodb.net/deepseek
SIGNING_SECRET=whsec_your_clerk_webhook_signing_secret_here
DEEPSEEK_API_KEY=sk-your_deepseek_api_key_here
```

---

### Step 7 — Run Locally (Development)

```bash
npm run dev
```

Open http://localhost:3000 in your browser. You should see the DeepSeek UI.

**For Clerk webhooks to work locally**, use [ngrok](https://ngrok.com):
```bash
# In a separate terminal:
npx ngrok http 3000
# Copy the https URL, e.g. https://abc123.ngrok.io
# Set Clerk webhook endpoint to: https://abc123.ngrok.io/api/clerk
```

---

### Step 8 — How It All Works (Code Walkthrough)

#### Authentication Flow
- `middleware.ts` applies Clerk auth to all routes
- `app/layout.js` wraps everything in `<ClerkProvider>`
- `app/api/clerk/route.js` receives Clerk webhook events and saves/updates/deletes users in MongoDB via the `User` model

#### State Management
- `context/AppContext.jsx` holds global state:
  - `chats` — all chats for the logged-in user
  - `selectedChat` — the currently open chat
  - `createNewChat()` — calls `/api/chat/create`
  - `fetchUsersChats()` — calls `/api/chat/get`

#### Sending a Message
1. User types in `PromptBox.jsx` and hits Enter or clicks send
2. The prompt is optimistically added to the local state immediately
3. A POST to `/api/chat/ai` is made with `{ chatId, prompt }`
4. `app/api/chat/ai/route.js`:
   - Authenticates the user via Clerk
   - Finds the chat in MongoDB
   - Calls `openai.chat.completions.create()` pointing at `https://api.deepseek.com`
   - Saves both messages (user + assistant) to MongoDB
   - Returns the assistant message
5. Back in `PromptBox.jsx`, the response is word-by-word animated into the UI (streaming effect)

#### Message Rendering
- `components/Message.jsx` renders user messages as plain text
- AI messages are rendered with `react-markdown` (for bold, lists, headers, etc.)
- Code blocks are syntax-highlighted using `prismjs` + `prism.css`

#### Sidebar / Chat History
- `components/Sidebar.jsx` lists all chats from context
- `components/ChatLabel.jsx` shows each chat with a rename/delete menu
- Clicking a chat sets it as `selectedChat`, which updates the message list

---

### Step 9 — Deploy to Vercel

1. Push your code to GitHub
2. Go to https://vercel.com and import the repo
3. Add all `.env` variables in Vercel's **Environment Variables** settings
4. Deploy — Vercel auto-detects Next.js
5. Update your Clerk webhook URL to your Vercel production domain:
   `https://your-app.vercel.app/api/clerk`

---

## ⚙️ Available Scripts

| Command | Description |
|---|---|
| `npm run dev` | Start dev server with Turbopack |
| `npm run build` | Build for production |
| `npm run start` | Start production server |
| `npm run lint` | Run ESLint |

---

## 🛠️ Customization Tips

- **Change the AI model**: In `app/api/chat/ai/route.js`, change `"deepseek-chat"` to `"deepseek-reasoner"` for the R1 reasoning model
- **Add streaming**: Replace the current batch response with a streaming response using `openai.chat.completions.stream()`
- **Change colors**: Edit `globals.css` — the primary color is `--color-primary: #4d6bfe`
- **Add more features**: The sidebar buttons (DeepThink R1, Search) are currently visual only — wire them up to the API call params

---

## ❗ Common Issues

| Issue | Fix |
|---|---|
| `MONGODB_URI` not connecting | Check IP whitelist in MongoDB Atlas |
| Clerk webhook 400 error | Verify `SIGNING_SECRET` matches the webhook in Clerk dashboard |
| DeepSeek 401 error | Check `DEEPSEEK_API_KEY` is valid and has credits |
| `npm install` fails | Use Node v18+ and run `npm install --legacy-peer-deps` if needed |
| Images not loading in Next.js | All assets are in `/assets/` and imported via `assets.js` — no config needed |

---

## 📚 Tech Stack Reference

| Technology | Purpose | Docs |
|---|---|---|
| Next.js 15 | Full-stack React framework | https://nextjs.org |
| Tailwind CSS 4 | Utility-first styling | https://tailwindcss.com |
| Clerk | Auth (sign in, webhooks) | https://clerk.com/docs |
| MongoDB + Mongoose | Database + ORM | https://mongoosejs.com |
| DeepSeek API | AI chat completions | https://platform.deepseek.com/docs |
| OpenAI SDK | API client (DeepSeek-compatible) | https://github.com/openai/openai-node |
| react-markdown | Render AI responses | https://github.com/remarkjs/react-markdown |
| Prism.js | Code highlighting | https://prismjs.com |
