# DevStash — Project Overview

> **One fast, searchable, AI-enhanced hub for all dev knowledge & resources.**

DevStash solves the problem of scattered developer knowledge — snippets in VS Code, prompts buried in chat history, commands lost in bash history, links spread across bookmarks, docs in random folders. Instead of context-switching between five different tools, developers get a single, organized workspace.

---

## Target Users

| Persona | Core Need |
|---|---|
| **Everyday Developer** | Quick access to snippets, prompts, commands, links |
| **AI-first Developer** | Organized prompts, contexts, workflows, system messages |
| **Content Creator / Educator** | Code blocks, explanations, course notes |
| **Full-stack Builder** | Patterns, boilerplates, API examples |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 16 / React 19 (SSR + API routes, single repo) |
| Language | TypeScript |
| Database | Neon PostgreSQL |
| ORM | Prisma 7 (migrations only — never `db push`) |
| Auth | NextAuth v5 (email/password + GitHub OAuth) |
| File Storage | Cloudflare R2 |
| AI | OpenAI `gpt-5-nano` |
| Styling | Tailwind CSS v4 + shadcn/ui |
| Cache | Redis (TBD) |

---

## Data Model

### Entity Relationship Diagram

```
┌──────────┐       ┌──────────────┐       ┌──────────────┐
│   User   │──1:N──│     Item     │──N:1──│   ItemType   │
│          │       │              │       │              │
│ id       │       │ id           │       │ id           │
│ email    │       │ title        │       │ name         │
│ name     │       │ contentType  │       │ icon         │
│ isPro    │       │ content      │       │ color        │
│ stripeId │       │ fileUrl      │       │ isSystem     │
│ stripeSub│       │ fileName     │       │ userId (null │
│          │       │ fileSize     │       │  for system) │
└──────────┘       │ url          │       └──────────────┘
     │             │ description  │
     │             │ language     │              ┌───────┐
     │             │ isFavorite   │──N:M─────────│  Tag  │
     │             │ isPinned     │              │       │
     │             │ createdAt    │              │ id    │
     │             │ updatedAt    │              │ name  │
     │             └──────┬───────┘              └───────┘
     │                    │
     │                    │ N:M
     │                    │
     │             ┌──────┴───────┐
     └──1:N───────│  Collection  │
                   │              │
                   │ id           │
                   │ name         │
                   │ description  │
                   │ isFavorite   │
                   │ defaultTypeId│
                   │ createdAt    │
                   │ updatedAt    │
                   └──────────────┘
```

### Prisma Schema

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ── User (extends NextAuth) ──────────────────────────────

model User {
  id                  String       @id @default(cuid())
  name                String?
  email               String?      @unique
  emailVerified       DateTime?
  image               String?
  isPro               Boolean      @default(false)
  stripeCustomerId    String?      @unique
  stripeSubscriptionId String?     @unique

  items               Item[]
  collections         Collection[]
  itemTypes           ItemType[]
  accounts            Account[]
  sessions            Session[]

  createdAt           DateTime     @default(now())
  updatedAt           DateTime     @updatedAt
}

// NextAuth required models
model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String?
  access_token      String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?
  session_state     String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}

// ── Item Types ───────────────────────────────────────────

model ItemType {
  id       String  @id @default(cuid())
  name     String
  icon     String
  color    String
  isSystem Boolean @default(false)

  userId   String?
  user     User?   @relation(fields: [userId], references: [id], onDelete: Cascade)

  items    Item[]

  @@unique([name, userId])
}

// ── Items ────────────────────────────────────────────────

model Item {
  id          String   @id @default(cuid())
  title       String
  contentType String   // "text" | "url" | "file"
  content     String?  // text/markdown content (null if file)
  fileUrl     String?  // Cloudflare R2 URL (null if text)
  fileName    String?  // original filename
  fileSize    Int?     // bytes
  url         String?  // for link-type items
  description String?
  language    String?  // programming language (for snippets/commands)
  isFavorite  Boolean  @default(false)
  isPinned    Boolean  @default(false)

  userId      String
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  itemTypeId  String
  itemType    ItemType @relation(fields: [itemTypeId], references: [id])

  tags        TagsOnItems[]
  collections ItemCollection[]

  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@index([userId, itemTypeId])
  @@index([userId, isFavorite])
  @@index([userId, isPinned])
}

// ── Collections ──────────────────────────────────────────

model Collection {
  id            String   @id @default(cuid())
  name          String
  description   String?
  isFavorite    Boolean  @default(false)
  defaultTypeId String?

  userId        String
  user          User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  items         ItemCollection[]

  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  @@index([userId])
}

// ── Join Tables ──────────────────────────────────────────

model ItemCollection {
  itemId       String
  collectionId String
  addedAt      DateTime @default(now())

  item         Item       @relation(fields: [itemId], references: [id], onDelete: Cascade)
  collection   Collection @relation(fields: [collectionId], references: [id], onDelete: Cascade)

  @@id([itemId, collectionId])
}

// ── Tags ─────────────────────────────────────────────────

model Tag {
  id    String        @id @default(cuid())
  name  String        @unique
  items TagsOnItems[]
}

model TagsOnItems {
  itemId String
  tagId  String

  item   Item @relation(fields: [itemId], references: [id], onDelete: Cascade)
  tag    Tag  @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([itemId, tagId])
}
```

---

## System Item Types

These are seeded on first run and cannot be modified or deleted by users (`isSystem: true`, `userId: null`).

| Type | Content Type | Color | Icon (Lucide) | Route |
|---|---|---|---|---|
| Snippet | `text` | `#3b82f6` blue | `Code` | `/items/snippets` |
| Prompt | `text` | `#8b5cf6` purple | `Sparkles` | `/items/prompts` |
| Command | `text` | `#f97316` orange | `Terminal` | `/items/commands` |
| Note | `text` | `#fde047` yellow | `StickyNote` | `/items/notes` |
| Link | `url` | `#10b981` emerald | `Link` | `/items/links` |
| File | `file` | `#6b7280` gray | `File` | `/items/files` |
| Image | `file` | `#ec4899` pink | `Image` | `/items/images` |

File and Image types are **Pro only**.

---

## Features

### Core (Free Tier)

- **Items** — Create, edit, delete items within a slide-out drawer (quick access). Each item has a type, optional tags, optional description, and content appropriate to its type (text/markdown, URL, or file).
- **Collections** — Group items of any type. An item can belong to multiple collections (e.g., a React snippet in both "React Patterns" and "Interview Prep").
- **Search** — Full-text search across content, tags, titles, and types.
- **Favorites & Pins** — Favorite collections and items; pin items to top of lists.
- **Recently Used** — Track and surface recently accessed items.
- **Markdown Editor** — Rich editing for text-based types (snippet, prompt, note, command) with syntax highlighting.
- **Import from File** — Import code content from uploaded files.
- **Dark/Light Mode** — Dark mode default, light mode optional.
- **Multi-collection Management** — Add/remove items to/from multiple collections; view which collections an item belongs to.
- **Auth** — Email/password and GitHub OAuth via NextAuth v5.

### Pro Features ($8/mo or $72/yr)

- **Unlimited items and collections** (free: 50 items, 3 collections)
- **File & Image uploads** — Stored in Cloudflare R2
- **Custom item types** (future)
- **AI Auto-tagging** — Suggest tags based on content
- **AI Code Explanation** — Explain selected code snippets
- **AI Summaries** — Summarize notes and longer content
- **AI Prompt Optimizer** — Improve AI prompts
- **Data Export** — JSON and ZIP formats

> **Dev note:** During development, all users have full access. Pro gating is wired up but not enforced until launch.

---

## Free vs. Pro Limits

| Feature | Free | Pro |
|---|---|---|
| Items | 50 | Unlimited |
| Collections | 3 | Unlimited |
| File/Image uploads | ✗ | ✓ |
| Custom types | ✗ | ✓ (future) |
| AI features | ✗ | ✓ |
| Data export | ✗ | ✓ |
| Search | Basic | Basic |

---

## UI/UX

### Design Principles

- Modern, minimal, developer-focused
- Dark mode default — clean typography, generous whitespace, subtle borders and shadows
- Reference aesthetic: Notion × Linear × Raycast
- Syntax highlighting for all code blocks (via a library like Prism or Shiki)

### Layout

```
┌──────────────────────────────────────────────────────┐
│  ┌────────────┐  ┌────────────────────────────────┐  │
│  │  Sidebar   │  │         Main Content           │  │
│  │            │  │                                │  │
│  │  Types     │  │  Collections (color-coded      │  │
│  │  ─ Snippets│  │  cards, bg color = dominant    │  │
│  │  ─ Prompts │  │  item type)                    │  │
│  │  ─ Commands│  │                                │  │
│  │  ─ Notes   │  │  Items (color-coded cards,     │  │
│  │  ─ Links   │  │  border color = item type)     │  │
│  │  ─ Files 🔒│  │                                │  │
│  │  ─ Images🔒│  │  ┌─── Drawer ──────────────┐   │  │
│  │            │  │  │  Item detail / editor    │   │  │
│  │  Recent    │  │  │  (slides in from right)  │   │  │
│  │  Colls     │  │  └─────────────────────────┘   │  │
│  │            │  │                                │  │
│  └────────────┘  └────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

- **Sidebar** — Collapsible. Lists item types (with icons + counts), recent/favorite collections. On mobile, becomes a slide-out drawer.
- **Main** — Grid layout. Collection cards use the color of their dominant item type as a background tint. Individual item cards use the type color as a border accent.
- **Item Drawer** — Slides in from the right for quick view/edit without leaving context.

### Micro-interactions

- Smooth transitions on navigation and drawer open/close
- Hover states on all cards
- Toast notifications for CRUD actions
- Skeleton loading states

### Responsive

- Desktop-first, mobile-usable
- Sidebar collapses to hamburger menu on small screens

---

## Route Structure

```
/                          → Dashboard (collections + recent items)
/items/snippets            → All snippets
/items/prompts             → All prompts
/items/commands            → All commands
/items/notes               → All notes
/items/links               → All links
/items/files               → All files (Pro)
/items/images              → All images (Pro)
/collections               → All collections
/collections/[id]          → Single collection view
/search                    → Search results
/settings                  → User settings, billing, export
/auth/signin               → Sign in
/auth/signup               → Sign up
```

---

## API Routes

```
POST   /api/items              → Create item
GET    /api/items               → List items (with filters: type, tag, search, favorites, pinned)
GET    /api/items/[id]          → Get single item
PATCH  /api/items/[id]          → Update item
DELETE /api/items/[id]          → Delete item

POST   /api/collections         → Create collection
GET    /api/collections         → List collections
GET    /api/collections/[id]    → Get collection with items
PATCH  /api/collections/[id]    → Update collection
DELETE /api/collections/[id]    → Delete collection

POST   /api/collections/[id]/items    → Add item(s) to collection
DELETE /api/collections/[id]/items    → Remove item(s) from collection

GET    /api/tags                → List all tags
POST   /api/tags                → Create tag

POST   /api/upload              → Upload file to R2 (Pro)

POST   /api/ai/auto-tag         → AI tag suggestions (Pro)
POST   /api/ai/explain          → AI code explanation (Pro)
POST   /api/ai/summarize        → AI summary (Pro)
POST   /api/ai/optimize-prompt  → AI prompt optimization (Pro)

POST   /api/stripe/checkout     → Create Stripe checkout session
POST   /api/stripe/webhook      → Handle Stripe events
POST   /api/stripe/portal       → Create billing portal session

POST   /api/export              → Export user data (Pro)
```

---

## Seed Data

System item types should be seeded via a Prisma seed script:

```ts
// prisma/seed.ts
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

const systemTypes = [
  { name: "Snippet",  icon: "Code",       color: "#3b82f6", isSystem: true },
  { name: "Prompt",   icon: "Sparkles",   color: "#8b5cf6", isSystem: true },
  { name: "Command",  icon: "Terminal",    color: "#f97316", isSystem: true },
  { name: "Note",     icon: "StickyNote",  color: "#fde047", isSystem: true },
  { name: "Link",     icon: "Link",        color: "#10b981", isSystem: true },
  { name: "File",     icon: "File",        color: "#6b7280", isSystem: true },
  { name: "Image",    icon: "Image",       color: "#ec4899", isSystem: true },
];

async function main() {
  for (const type of systemTypes) {
    await prisma.itemType.upsert({
      where: { name_userId: { name: type.name, userId: null } },
      update: {},
      create: type,
    });
  }
  console.log("Seeded system item types");
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(() => prisma.$disconnect());
```

---

## Key Development Notes

1. **Migrations only** — Never use `prisma db push`. Create migrations with `prisma migrate dev` locally, apply with `prisma migrate deploy` in production.
2. **Pro gating** — Build the infrastructure (middleware checks, UI badges) but leave everything unlocked during dev.
3. **Prisma 7** — Fetch latest docs before implementing; APIs may differ from v5/v6.
4. **Next.js 16** — Confirm App Router patterns and any breaking changes from v15.
5. **File uploads** — Use presigned URLs to upload directly from client to R2, store the URL in the `Item` record.
6. **Search** — Start with Prisma full-text search on PostgreSQL. Evaluate dedicated search (e.g., Meilisearch) if performance becomes an issue.
