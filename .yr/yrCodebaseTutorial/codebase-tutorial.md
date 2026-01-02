# Tutorial: Understanding Your NextJS Task App with Supabase

## Table of Contents
1. [Overview: What is this app?](#overview)
2. [NextJS basics in this app](#nextjs-basics)
3. [Project structure](#project-structure)
4. [Authentication flow](#authentication)
5. [Task management](#task-management)
6. [AI integration (OpenAI)](#ai-integration)
7. [Stripe payments](#stripe-payments)
8. [Image storage](#image-storage)
9. [How it all connects](#how-it-all-connects)

---

## Overview: What is this app? {#overview}

This is a **task management application** (like a to-do list app) where users can:
- Sign in with Google OAuth or email/password
- Create, edit, and delete tasks
- Get AI-suggested labels for their tasks (using OpenAI)
- Attach images to tasks (stored in Supabase Storage)
- Subscribe to premium features (using Stripe)

Think of it as a full-stack SaaS (Software as a Service) application that demonstrates modern web development practices.

---

## NextJS basics in this app {#nextjs-basics}

### What is NextJS?
NextJS is a **React framework** that adds powerful features on top of React:
- **File-based routing**: Files in the `app/` folder automatically become routes
- **Server and client components**: Some components run on the server, others in the browser
- **Built-in optimizations**: Image optimization, code splitting, etc.

### Key NextJS concepts used here

1. **"use client" directive**
   - Components that use React hooks (like `useState`, `useEffect`) or event handlers need `"use client"` at the top
   - This tells NextJS: "This component needs to run in the browser, not on the server"
   - Example: `app/page.tsx` uses `"use client"` because it renders `LoginForm` which has interactive state

2. **File-based routing**
   ```
   app/
     page.tsx          → http://localhost:3000/
     dashboard/
       page.tsx        → http://localhost:3000/dashboard
     task/
       page.tsx        → http://localhost:3000/task?id=123
   ```

3. **Layouts**
   - `app/layout.tsx` wraps all pages
   - Provides shared UI (Header, Footer, styling)

---

## Project structure {#project-structure}

```
task-app-working/
├── app/                    # NextJS pages (routes)
│   ├── page.tsx           # Home page (login)
│   ├── layout.tsx         # Root layout (wraps all pages)
│   ├── dashboard/         # Dashboard page
│   ├── task/              # Task detail/edit page
│   └── profile/           # User profile/subscription page
│
├── components/            # Reusable React components
│   ├── LoginForm.tsx     # Login/signup form
│   ├── TaskList.tsx      # List of tasks
│   ├── CreateTaskForm.tsx # Form to create tasks
│   ├── RouteGuard.tsx    # Protects routes (requires login)
│   └── ui/               # UI components (buttons, inputs, etc.)
│
├── hooks/                # Custom React hooks (reusable logic)
│   ├── useAuth.ts        # Authentication logic
│   ├── useTaskManager.ts # Task CRUD operations
│   ├── useOpenAI.ts      # OpenAI API calls
│   └── useSubscription.ts # Stripe subscription management
│
├── supabase/
│   ├── functions/        # Edge Functions (serverless backend)
│   │   ├── create-task-with-ai/  # Creates task + AI label
│   │   ├── openai-chat/          # General OpenAI chat
│   │   ├── create-stripe-session/ # Stripe checkout
│   │   └── stripe-webhook/       # Handles Stripe events
│   └── migrations/       # Database schema changes
│
└── types/                # TypeScript type definitions
```

---

## Authentication flow {#authentication}

### How authentication works

1. **User visits the home page** (`app/page.tsx`)

   ```tsx
   // app/page.tsx
   export default function Home() {
     return <LoginForm />;
   }
   ```

2. **LoginForm component** uses the `useAuth` hook
   - The `useAuth` hook contains all the authentication logic
   - Located in `hooks/useAuth.ts`

3. **Inside useAuth hook**:

   ```typescript
   // Creates a Supabase client for the browser
   const supabase = createBrowserClient(
     process.env.NEXT_PUBLIC_SUPABASE_URL!,
     process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
   );
   ```
   - `createBrowserClient` creates a Supabase client that runs in the browser
   - Uses environment variables (stored in `.env.local`) for Supabase URL and anon key

4. **Login methods**:

   a) **Email/Password**:

   ```typescript
   const handleLogin = async (e: React.FormEvent) => {
     e.preventDefault();
     const { error } = await supabase.auth.signInWithPassword({
       email,
       password,
     });
   };
   ```

   b) **Google OAuth**:

   ```typescript
   const handleGoogleLogin = async () => {
     await supabase.auth.signInWithOAuth({
       provider: "google",
       options: {
         redirectTo: `${window.location.origin}/dashboard`,
       },
     });
   };
   ```
   - This redirects the user to Google's login page
   - After login, Google redirects back to `/dashboard`

5. **Session management**:

   ```typescript
   useEffect(() => {
     // Check if user is already logged in
     const { data: { session } } = await supabase.auth.getSession();
     
     // Listen for auth changes (login/logout)
     supabase.auth.onAuthStateChange((_event, session) => {
       updateSessionState(session);
     });
   }, []);
   ```

6. **Route protection** (`components/RouteGuard.tsx`):
   - Checks if user is logged in
   - Redirects to login if not authenticated
   - Redirects logged-in users away from login page

---

## Task management {#task-management}

### How tasks are created

1. **User clicks "Create Task"** on the dashboard
   - Opens a dialog with `CreateTaskForm`

2. **Form submission** (`components/CreateTaskForm.tsx`):

   ```typescript
   const handleSubmit = async (e: React.FormEvent) => {
     e.preventDefault();
     await onSubmit(title, description); // Calls parent's handler
   };
   ```

3. **Dashboard handles creation** (`app/dashboard/page.tsx`):

   ```typescript
   const handleCreateTask = async (title: string, description: string) => {
     await createTask(title, description); // From useTaskManager hook
     await refreshTasks();
   };
   ```

4. **useTaskManager hook** (`hooks/useTaskManager.ts`):
   - Instead of directly inserting into the database, it calls a **Supabase Edge Function**:

   ```typescript
   const response = await fetch(FUNCTION_ENDPOINT, {
     method: "POST",
     headers: {
       "Content-Type": "application/json",
       Authorization: `Bearer ${session!.access_token}`,
     },
     body: JSON.stringify({ title, description }),
   });
   ```
   - **Why Edge Function?** Because creating a task also triggers AI to suggest a label, and we want to do both in one secure server-side operation

5. **Edge Function** (`supabase/functions/create-task-with-ai/index.ts`):
   - Verifies the user is authenticated
   - Creates the task in the database
   - Calls OpenAI to suggest a label
   - Updates the task with the label
   - Returns the created task

### How tasks are displayed

1. **Dashboard loads tasks**:

   ```typescript
   useEffect(() => {
     fetchTasks(); // Runs when component mounts
   }, []);
   ```

2. **fetchTasks in useTaskManager**:

   ```typescript
   const { data, error } = await supabase
     .from("tasks")
     .select("*")
     .eq("user_id", session!.user.id)  // Only get current user's tasks
     .order("created_at", { ascending: false });
   ```

3. **Tasks are rendered** by `TaskList` component:
   - Maps over the tasks array
   - Each task is a `TaskRow` with edit/delete actions

### How tasks are updated

1. **User clicks a task** → goes to `/task?id=123`
2. **Task detail page** (`app/task/page.tsx`) uses `useTaskManager(taskId)`:

   ```typescript
   const { task, updateTask, saveTask } = useTaskManager(taskId);
   ```
   - Fetches the specific task
   - `updateTask` updates local state (optimistic update)
   - `saveTask` writes to the database

3. **Saving**:

   ```typescript
   const { error } = await supabase
     .from("tasks")
     .update({ ...taskData, updated_at: new Date().toISOString() })
     .eq("task_id", taskData.task_id);
   ```

---

## AI integration (OpenAI) {#ai-integration}

### How AI labeling works

1. **When creating a task**, the Edge Function calls OpenAI:

   ```typescript
   // In create-task-with-ai Edge Function
   const prompt = `Based on this task title: "${title}" and description: "${description}", 
                   suggest ONE of these labels: work, personal, priority, shopping, home. 
                   Reply with just the label word and nothing else.`;
   
   const completion = await openai.chat.completions.create({
     messages: [{ role: "user", content: prompt }],
     model: "gpt-4o-mini",
   });
   ```

2. **Why use Edge Function?**
   - Keeps OpenAI API key secure (never exposed to browser)
   - Validates user before calling OpenAI
   - Can handle errors gracefully

3. **General OpenAI chat** (`hooks/useOpenAI.ts`):
   - Provides a `chat` function for other OpenAI calls
   - Also calls the `openai-chat` Edge Function
   - Example usage in `components/OpenAIChatExample.tsx`

---

## Stripe payments {#stripe-payments}

### How subscriptions work

1. **User clicks "Manage Subscription"** on profile page
2. **useSubscription hook** (`hooks/useSubscription.ts`):

   ```typescript
   const manageSubscription = async (accessToken: string) => {
     const response = await fetch(
       `${SUPABASE_URL}/functions/v1/create-stripe-session`,
       {
         method: "POST",
         headers: {
           Authorization: `Bearer ${accessToken}`,
         },
       }
     );
     const data = await response.json();
     window.location.href = data.url; // Redirect to Stripe checkout
   };
   ```

3. **Edge Function** (`supabase/functions/create-stripe-session/index.ts`):
   - Creates a Stripe checkout session
   - Returns a URL to redirect the user

4. **After payment**, Stripe sends a webhook to:
   - `supabase/functions/stripe-webhook/index.ts`
   - Updates user's subscription status in database

5. **Why Edge Functions?**
   - Keeps Stripe secret keys secure
   - Validates webhook signatures
   - Updates database securely

---

## Image storage {#image-storage}

### How image uploads work

1. **User drags/drops an image** on task detail page
2. **useTaskManager hook** handles upload:

   ```typescript
   const uploadImage = async (file: File) => {
     const fileName = `${task.user_id}/${task.task_id}.${fileExt}`;
     const { error } = await supabase.storage
       .from("task-attachments")  // Bucket name
       .upload(fileName, file, {
         upsert: true,  // Overwrite if exists
       });
   };
   ```

3. **Supabase Storage**:
   - Files stored in a bucket named `task-attachments`
   - Path format: `{user_id}/{task_id}.{extension}`
   - Public URL: `{SUPABASE_URL}/storage/v1/object/public/task-attachments/{fileName}`

4. **Displaying images**:

   ```tsx
   <Image
     src={`${SUPABASE_URL}/storage/v1/object/public/task-attachments/${task.image_url}`}
     alt="Task attachment"
   />
   ```

---

## How it all connects {#how-it-all-connects}

### User journey example

1. User visits `/` → sees `LoginForm`
2. Logs in with Google → redirected to `/dashboard`
3. `RouteGuard` checks authentication → allows access
4. Dashboard loads → `useTaskManager` fetches tasks from Supabase
5. User creates task → calls Edge Function → creates task + AI label
6. User clicks task → goes to `/task?id=123` → edits and uploads image
7. Image uploaded to Supabase Storage → URL saved in task
8. User goes to `/profile` → manages Stripe subscription

### Data flow diagram

```
Browser (NextJS App)
    ↓
Supabase Client (createBrowserClient)
    ↓
┌─────────────────────────────────────┐
│  Supabase Services:                │
│  - Auth (Google OAuth, Email)      │
│  - Database (PostgreSQL)            │
│  - Storage (Images)                 │
│  - Edge Functions (Serverless)     │
└─────────────────────────────────────┘
    ↓
Edge Functions call:
  - OpenAI API (for AI labels)
  - Stripe API (for payments)
```

### Key concepts summary

1. **React Hooks** (`useAuth`, `useTaskManager`, etc.):
   - Encapsulate logic and state
   - Reusable across components

2. **Supabase Client**:
   - Created with `createBrowserClient`
   - Handles auth, database, and storage

3. **Edge Functions**:
   - Serverless functions on Supabase
   - Secure API key handling
   - Run on server, not in browser

4. **Environment Variables**:
   - `NEXT_PUBLIC_*` are exposed to browser
   - Others are server-only
   - Stored in `.env.local`

---

## Common patterns in this codebase

### 1. Custom hooks pattern

```typescript
// hooks/useAuth.ts
export function useAuth() {
  const [user, setUser] = useState(null);
  // ... logic ...
  return { user, handleLogin, handleSignup, ... };
}

// components/LoginForm.tsx
const { user, handleLogin } = useAuth();
```

### 2. Loading states

```typescript
const [isLoading, setIsLoading] = useState(true);
// ... fetch data ...
setIsLoading(false);
```

### 3. Error handling

```typescript
try {
  await someOperation();
} catch (error: any) {
  setError(error.message);
  console.error("Error:", error);
}
```

### 4. TypeScript types
- Defined in `types/` directory
- Used throughout for type safety

---

## Tips for learning

1. **Start with the user flow**: Pick a feature and trace it from UI to database
2. **Use browser DevTools**: Check Network tab, Console, React DevTools
3. **Add console.logs**: Log values to understand flow
4. **Read Supabase docs**: Understand the services being used
5. **Experiment**: Change small things and see what happens

---

## Next steps

1. **Explore the code**: Pick a feature and trace it end-to-end
2. **Add a feature**: Try adding task search or filtering
3. **Modify UI**: Change styling or add new components
4. **Understand database**: Check `supabase/migrations/` for schema
5. **Test Edge Functions**: Deploy and test locally

This app demonstrates:
- React hooks and state management
- NextJS routing and layouts
- Supabase integration (Auth, Database, Storage, Functions)
- External API integration (OpenAI, Stripe)
- TypeScript usage

If you want details on any part, ask.

