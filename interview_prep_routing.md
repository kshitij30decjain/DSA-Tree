# Interview Preparation Guide: Frontend Architecture

This document covers the frontend architectural patterns used in this codebase to manage user routing, database state synchronization, and page redirection. It is designed to help you prepare for technical interview questions related to state management, SSR/CSR redirection, and Next.js App Router patterns.

---

## Core Concept: Hybrid Routing & State Synchronization

In a modern server-client framework like Next.js, application state needs to be synchronized across three distinct layers:
1. **Persistent Database Layer (Supabase PostgreSQL)**: Holds the ground truth of user selections (e.g., the active chatbot).
2. **Reactive Client Layer (React Context)**: Provides fast, in-memory updates for UI elements.
3. **Navigational Route Layer (Next.js App Router URLs)**: Keeps the address bar synchronized with the active view.

---

## Pattern 1: Server-Side Login Redirection (SSR)

### Problem
When a user logs in, redirecting them to a generic dashboard (`/chat`) and then using a client-side `useEffect` to fetch their preferred chatbot and redirect them again causes a **double redirect**. This results in visible layout flashing, increased latency, and a poor user experience.

### Solution
Perform database checks directly on the server during the authentication flow (within Server Components and Route Handlers) to issue a single redirect command to the browser.

```
[User Login / SSO Callback]
            │
            ▼
┌────────────────────────┐
│  Server-Side Handler   │
│  (ADFS, JumpCloud, etc)│
└───────────┬────────────┘
            │
            ▼ (Fetch user's home workspace & active chatbot preference)
┌────────────────────────┐
│  Supabase DB Query     │
└───────────┬────────────┘
            │
            ├─────────────── None ────────────────► Redirect to /chat
            ├─────────────── NULL ────────────────► Redirect to /chatbot/{default_id}
            └─────────── Chatbot ID ──────────────► Redirect to /chatbot/{chatbot_id}
```

### Key Files
* **[login/page.tsx](file:///home/ubuntu/Downloads/frontend/jieum-bot/frontend/jieum-chatbot/app/[locale]/login/page.tsx)**: Resolves workspace and active chatbot server-side for password logins.
* **[redirect/route.ts](file:///home/ubuntu/Downloads/frontend/jieum-bot/frontend/jieum-chatbot/app/[locale]/jieum-ui/login/redirect/route.ts)**: Handles SSO (ADFS) callbacks, queries preferences, and redirects directly.

---

## Pattern 2: Client-Side Layout Guards (CSR)

### Problem
If a user is inside the application and attempts to manually navigate to the generic `/chat` path (or clicks a home link), the application needs to respect their saved chatbot preference and direct them back to the chatbot view.

### Solution
Use Next.js Root/Workspace Layouts to acts as **Sticky Navigation Guards**. A layout wraps child pages and uses React hooks to detect route transitions, checking the in-memory global state or querying the database to redirect the user to the correct path.

```typescript
// Example from layout.tsx
useEffect(() => {
  if (loading) return
  
  const isChatPage = typeof window !== "undefined" && window.location.pathname.endsWith("/chat")
  
  if (isChatPage && activeChatbotId !== "None") {
    // Force user to their active chatbot page if they navigated to /chat
    router.replace(`/${locale}/${workspaceId}/chatbot/${activeChatbotId}`)
  }
}, [loading, activeChatbotId])
```

### Key Files
* **[layout.tsx](file:///home/ubuntu/Downloads/frontend/jieum-bot/frontend/jieum-chatbot/app/[locale]/[workspaceid]/layout.tsx)**: Guards the general chat route and redirects users if they have an active chatbot selected.

---

## Pattern 3: Global Context with Background Sync

### Problem
Updating local UI state and waiting for a database write to complete before updating the route makes the interface feel sluggish and unresponsive.

### Solution
Implement **Optimistic UI Updates** and **Background Synchronization** inside custom React Hooks and Contexts. 

When a user selects a chatbot from the dropdown:
1. The dropdown handler immediately updates the React Context (`activeChatbotId`).
2. The UI reacts instantly (e.g., showing the new bot icon).
3. An asynchronous database write (`updateActiveChatbot(...)`) is fired in the background.
4. The router navigates to the new chatbot route (`/chatbot/{chatbot_id}`).

```typescript
// Example from chatbot-selector.tsx
const handleSelect = (chatbotId: string) => {
  const targetId = chatbotId === "None" ? "None" : chatbotId
  
  // 1. Instantly update client context
  setActiveChatbotId(targetId)
  
  // 2. Sync to database in background (fire-and-forget)
  updateActiveChatbot(profile.user_id, targetId)
    .catch(err => console.error("Error updating active chatbot:", err))
    
  // 3. Perform routing transition
  router.push(`/${workspaceId}/chatbot/${targetId}`)
}
```

---

## Potential Interview Questions & Answers

### Q: Why use React Context instead of Redux or Zustand for this feature?
> **Answer**: React Context is ideal here because the state (`activeChatbotId`) is relatively simple and consumed by only a few key components (the layout, the selector dropdown, and the route hooks). Introducing Redux or Zustand would add unnecessary boilerplate, bundle size, and architectural complexity for state that is primarily persisted in and retrieved from a database anyway.

### Q: How do you handle cases where database writes fail during background synchronization?
> **Answer**: In a production environment, we should handle this by implementing rollback logic in our catch blocks. If `updateActiveChatbot` fails, we catch the error, alert the user or log it, and roll back the React Context state to the previous value to keep the UI in sync with the database.

### Q: What is the main benefit of Next.js App Router over Pages Router for this architecture?
> **Answer**: Next.js App Router allows us to mix Server Components/Route Handlers with Client Components seamlessly. We can run secure, high-performance database queries on the server during requests (like routing login flows) while maintaining rich client-side interactivity and context on the page layouts.
