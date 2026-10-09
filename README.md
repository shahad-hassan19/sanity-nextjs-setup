# Next.js + Sanity: setup and code reference

A reusable reference for adding Sanity to a Next.js (App Router) site with the Studio built into the app at `/admin`, so one Vercel deploy ships the site and the admin. The Studio has four tabs: **Dashboard**, **Structure** (the content), **Analytics** and **Health**.

Each section gives a file's location and its full content, in the order you'd create them, with the terminal commands in between. Placeholders to replace:
- `<your-project-id>`: your Sanity project ID;
- `example.com`: your domain;
- `My Site`: your site's name.

Run every command from your project folder:

```bash
cd path/to/your-project
```

## Contents

1. [Environment variables: which ones and how to get them](#1-environment-variables-which-ones-and-how-to-get-them)
2. [`sanity.config.ts`](#2-sanityconfigts)
3. [`sanity.cli.ts`](#3-sanityclits)
4. [`sanity/env.ts`](#4-sanityenvts)
5. [`sanity/structure.ts`](#5-sanitystructurets)
6. [`sanity/schemaTypes/index.ts`](#6-sanityschematypesindexts)
7. [`sanity/schemaTypes/blogPost.ts`](#7-sanityschematypesblogpostts)
8. [The other schema types](#8-the-other-schema-types)
9. [`sanity/tools/analytics`](#9-sanitytoolsanalytics)
10. [Analytics environment variables: which ones and how to get them](#10-analytics-environment-variables-which-ones-and-how-to-get-them)
11. [`sanity/tools/health`](#11-sanitytoolshealth)
12. [`sanity/tools/dashboard`](#12-sanitytoolsdashboard)
13. [Run and check](#13-run-and-check)

---

## 1. Environment variables: which ones and how to get them

**Location:** `.env.local` in the project root, for your computer. It's never committed. On Vercel, the same variables go under Project → **Settings** → **Environment Variables**, for both Production and Preview.

**First, create the Sanity project** (skip this if you already have one):

```bash
npm create sanity@latest -- --create-project "My Site" --dataset production --bare
```

This signs you in to Sanity in the browser and prints the new **Project ID**. You can also create the project at https://www.sanity.io/manage → **Create new project**.

Save the template below as `.env.example`, then copy it:

```bash
cp .env.example .env.local
```

| Variable | Required | Where to get it |
|---|---|---|
| `NEXT_PUBLIC_SANITY_PROJECT_ID` | Yes | sanity.io/manage → your project. The Project ID is at the top. |
| `NEXT_PUBLIC_SANITY_DATASET` | Yes | sanity.io/manage → project → **Datasets**. Usually `production`. |
| `SANITY_API_WRITE_TOKEN` | Yes (server only) | sanity.io/manage → project → **API** → **Tokens** → **Add API token**, with permission **Editor**. Copy it straight away, because it's shown only once. |
| `NEXT_PUBLIC_SITE_URL` | Yes in production | Your live domain with no trailing slash, e.g. `https://example.com`. Use `http://localhost:3000` on your computer. |
| `HEALTH_DOMAINS` | Optional | Comma-separated domains for the Health tab's expiry checks. If empty, it uses the domain of `NEXT_PUBLIC_SITE_URL`. |
| `GOOGLE_ANALYTICS_MEASUREMENT_ID` | Optional | See [section 10](#10-analytics-environment-variables-which-ones-and-how-to-get-them). |
| `GOOGLE_ANALYTICS_PROPERTY_ID`, `GOOGLE_ANALYTICS_CLIENT_EMAIL`, `GOOGLE_ANALYTICS_PRIVATE_KEY` | For the Analytics tab | See [section 10](#10-analytics-environment-variables-which-ones-and-how-to-get-them). |
| `RESEND_API_KEY`, `ENQUIRY_EMAIL_TO`, `ENQUIRY_EMAIL_FROM` | Optional | Form emails. resend.com → **API Keys**. The FROM address must be on a domain verified in Resend → **Domains**. |

**Allow the Studio to sign in from each address.** At sanity.io/manage → project → **API** → **CORS origins** → **Add CORS origin**, add each address below with **Allow credentials** ticked:
- `http://localhost:3000`
- your Vercel address
- your custom domain

**`.env.example`**

````bash
# Copy to .env.local for development, and set the same values in Vercel
# (Production + Preview). Never commit .env.local.
# Only NEXT_PUBLIC_* values reach the browser; everything else stays on the server.
# Restart `npm run dev` after changing this file.

# ── Sanity (required) ─────────────────────────────────────────────────────────
# sanity.io/manage → your project: Project ID at the top, dataset under Datasets.
NEXT_PUBLIC_SANITY_PROJECT_ID=
NEXT_PUBLIC_SANITY_DATASET=production

# Server-only token with Editor rights: sanity.io/manage → API → Tokens →
# Add API token → "Editor". Writes Error Log documents and enquiries, and is used
# by the Health tab's write check.
SANITY_API_WRITE_TOKEN=

# ── Site URL (required in production) ─────────────────────────────────────────
# The live domain, no trailing slash. The Health tab checks its pages, domain and TLS.
NEXT_PUBLIC_SITE_URL=https://example.com

# Optional: comma-separated domains for the Health tab's expiry checks.
# Empty: the domain of NEXT_PUBLIC_SITE_URL.
# HEALTH_DOMAINS="example.com,www.example.com"

# ── Google Analytics ──────────────────────────────────────────────────────────
# Tracking tag on the public site ("G-XXXXXXXXXX"). Empty: no tracking loads.
GOOGLE_ANALYTICS_MEASUREMENT_ID=
# The Studio's Analytics tab: a service account with Viewer access (see section 10).
GOOGLE_ANALYTICS_PROPERTY_ID=
GOOGLE_ANALYTICS_CLIENT_EMAIL=
GOOGLE_ANALYTICS_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"

# ── Email (optional) ──────────────────────────────────────────────────────────
# resend.com → API Keys. The FROM address must be on a domain verified in Resend.
RESEND_API_KEY=
ENQUIRY_EMAIL_TO=
ENQUIRY_EMAIL_FROM=
````

---

## 2. `sanity.config.ts`

**Location:** project root. This is the Studio's configuration:
- `basePath: "/admin"` makes it live at `/admin`;
- `auth: { loginMethod: "token" }` lets the custom tabs send your sign-in token to your API routes;
- `plugins` sets the order of the tabs, and the first one is where `/admin` opens.

The `"use client"` first line is required, because the Studio only runs in the browser.

Install the Studio packages:

```bash
npm install sanity styled-components @sanity/ui @sanity/icons next-sanity
```

**`sanity.config.ts`**

````ts
"use client";

import { defineConfig } from "sanity";
import { structureTool } from "sanity/structure";
import { structure } from "@/sanity/structure";
import { dataset, projectId } from "@/sanity/env";
import { schemaTypes } from "@/sanity/schemaTypes";
import { googleAnalyticsTool } from "@/sanity/tools/analytics";
import { healthTool } from "@/sanity/tools/health";
import { dashboardTool } from "@/sanity/tools/dashboard";

export default defineConfig({
  name: "default",
  title: "My Site CMS",
  basePath: "/admin",
  projectId,
  dataset,
  auth: { loginMethod: "token" },
  releases: { enabled: false },
  scheduledDrafts: { enabled: false },
  // Dashboard first: the Studio opens on its first tool.
  plugins: [dashboardTool(), structureTool({ structure }), googleAnalyticsTool(), healthTool()],
  schema: { types: schemaTypes },
});
````

---

## 3. `sanity.cli.ts`

**Location:** project root. It's used by terminal commands like `npx sanity …`. Those commands don't read `.env.local` and can't resolve `@/…` imports, so write the project ID and dataset in directly.

**`sanity.cli.ts`**

````ts
import { defineCliConfig } from "sanity/cli";

// `npx sanity …` commands read neither .env.local nor the "@/" import alias,
// so the project ID and dataset are written here directly.
export default defineCliConfig({
  api: { projectId: "<your-project-id>", dataset: "production" },
});
````

Check that the CLI can reach the project:

```bash
npx sanity documents query --api-version 2026-08-12 'count(*)'
```

---

## 4. `sanity/env.ts`

**Location:** `sanity/env.ts`. It reads the project ID and dataset from the environment variables, and is shared by the Studio, your data fetching and the API routes. `isSanityConfigured` is `false` when either variable is missing.

```bash
mkdir -p sanity lib/sanity
```

**`sanity/env.ts`**

````ts
export const apiVersion = "2026-08-12";

const configuredProjectId = process.env.NEXT_PUBLIC_SANITY_PROJECT_ID;
const configuredDataset = process.env.NEXT_PUBLIC_SANITY_DATASET;

// The valid-looking fallback lets the public site build before a Sanity project is connected.
// The Studio route shows setup instructions instead of using this value.
export const projectId = configuredProjectId || "replacewithprojectid";
export const dataset = configuredDataset || "production";

export const isSanityConfigured = Boolean(configuredProjectId && configuredDataset);
````

This is the read client used by the site, and by the Health tab's read check:

**`lib/sanity/client.ts`**

````ts
import { createClient } from "next-sanity";
import { apiVersion, dataset, projectId } from "@/sanity/env";

export const client = createClient({
    projectId,
    dataset,
    apiVersion,
    useCdn: true,
    perspective: "published",
});
````

---

## 5. `sanity/structure.ts`

**Location:** `sanity/structure.ts`. It sets the layout of the Structure tab's sidebar: the groups, the order, and single-document items like Site Settings. Add a line here for every content type you create.

**`sanity/structure.ts`**

````ts
import type { StructureResolver } from "sanity/structure";

/** Studio sidebar: grouped by area instead of one flat list of every type. */
export const structure: StructureResolver = (S) =>
  S.list()
    .title("Content")
    .items([
      // Single document: opens the form directly, no list of one.
      S.listItem()
        .title("Site Settings")
        .id("siteSettings")
        .child(S.document().schemaType("siteSettings").documentId("siteSettings")),

      S.listItem()
        .title("Enquiries")
        .id("enquiries")
        .child(
          S.documentTypeList("enquiry")
            .title("Enquiries")
            .defaultOrdering([{ field: "submittedAt", direction: "desc" }]),
        ),

      S.divider(),

      S.documentTypeListItem("blogPost").title("Blog Posts"),
      // Add your own content types here, e.g.
      // S.documentTypeListItem("product").title("Products"),

      S.divider(),

      S.documentTypeListItem("pageSeo").title("Page SEO"),

      S.listItem()
        .title("System")
        .child(
          S.list()
            .title("System")
            .items([
              S.documentTypeListItem("renewal").title("Renewals"),
              S.documentTypeListItem("errorLog").title("Error Logs"),
            ]),
        ),
    ]);
````

---

## 6. `sanity/schemaTypes/index.ts`

**Location:** `sanity/schemaTypes/index.ts`. It lists every content type the Studio knows about. A new schema file must be imported and added to `schemaTypes` here, or it won't show up.

```bash
mkdir -p sanity/schemaTypes
```

**`sanity/schemaTypes/index.ts`**

````ts
import { blogPost } from "./blogPost";
import { enquiry } from "./enquiry";
import { errorLog } from "./errorLog";
import { blockContent, seoType } from "./objects";
import { pageSeo } from "./pageSeo";
import { renewal } from "./renewal";
import { siteSettings } from "./siteSettings";

export const schemaTypes = [
  // Documents
  siteSettings,
  blogPost,
  pageSeo,
  enquiry,
  // System (used by the Health tab)
  errorLog,
  renewal,
  // Objects (reusable field groups)
  blockContent,
  seoType,
];
````

---

## 7. `sanity/schemaTypes/blogPost.ts`

**Location:** `sanity/schemaTypes/blogPost.ts`. These are the fields of a blog post. Change the category list to suit your site.

**`sanity/schemaTypes/blogPost.ts`**

````ts
import {defineField, defineType} from 'sanity'

export const blogPost = defineType({
    name: 'blogPost',
    title: 'Blog Post',
    type: 'document',
    fields: [
        defineField({
            name: 'title',
            title: "Title",
            type: 'string',
            validation: (rule) => rule.required(),
        }),
        defineField({
            name: 'slug',
            title: "Slug",
            type: 'slug',
            options: {
                source: 'title',
                maxLength: 96,
            },
            validation: (rule) => rule.required(),
        }),
        defineField({
            name: 'excerpt',
            title: "Excerpt",
            type: 'text',
            rows: 3,
            validation: (rule) => rule.required().max(240),
        }),
        defineField({
            name: "category",
            title: "Category",
            type: "string",
            options: {
                list: [
                    { title: "News", value: "News" },
                    { title: "Guides", value: "Guides" },
                    { title: "Tips", value: "Tips" },
                ],
            },
            validation: (rule) => rule.required(),
        }),
        defineField({
            name: 'coverImage',
            title: "Cover Image",
            type: 'image',
            options: { hotspot: true },
            fields: [defineField({ name: "alt", title: "Alternative Text", type: "string", validation: (rule) => rule.required() })],
        }),
        defineField({
            name: 'heroImage',
            title: "Hero Image",
            type: 'image',
            description: "Large image at the top of the article. Falls back to the Cover Image if left empty.",
            options: { hotspot: true },
            fields: [defineField({ name: "alt", title: "Alternative Text", type: "string", validation: (rule) => rule.required() })],
        }),
        defineField({
            name: 'body',
            title: "Body",
            type: 'blockContent',
            validation: (rule) => rule.required()
        }),
        defineField({
            name: 'readTime',
            title: "Read Time (minutes)",
            type: 'number',
            validation: (rule) => rule.integer().positive(),
        }),
        defineField({
            name: "author",
            title: "Author",
            type: "string",
            description: "The byline beside the date on the article page. Leave empty to show none.",
            initialValue: "Editorial Team"
        }),
        defineField({
            name: 'publishedAt',
            title: "Published At",
            type: 'datetime',
            initialValue: () => new Date().toISOString(),
            validation: (rule) => rule.required(),
        }),
        defineField({
            name: "seo",
            title: "SEO",
            type: "seo"
        }),
        defineField({
            name: "featured",
            title: "Featured",
            type: "boolean",
            initialValue: false
        }),
        defineField({
            name: "active",
            title: "Active",
            type: "boolean",
            initialValue: true
        }),
    ],
    orderings: [
       {
            title: "Newest first",
            name: "publishedAtDesc",
            by: [{
                field: "publishedAt",
                direction: "desc"
            }]
        },
    ],
   preview: {
        select: {
            title: "title",
            subtitle: "category",
            media: "coverImage"
        },
   },
})
````

---

## 8. The other schema types

**Location:** `sanity/schemaTypes/`. Each file defines one or more content types, and every one of them is registered in `index.ts` (section 6).

| File | Content types |
|---|---|
| `objects.ts` | Reusable field groups: `blockContent` (rich text) and `seo` |
| `siteSettings.ts` | `siteSettings`: one document holding the site name, contact details, socials and default SEO |
| `pageSeo.ts` | `pageSeo`: SEO for fixed pages like `/` and `/about` |
| `enquiry.ts` | `enquiry`: form submissions from the website |
| `errorLog.ts` | `errorLog`: server errors, written by `instrumentation.ts` and shown in the Health tab |
| `renewal.ts` | `renewal`: domain, hosting and subscription renewals, shown in the Health tab |

**`sanity/schemaTypes/objects.ts`**

````ts
import { defineArrayMember, defineField, defineType } from "sanity";

export const blockContent = defineType({
  name: "blockContent",
  title: "Rich Text",
  type: "array",
  of: [
    defineArrayMember({
      type: "block",
      styles: [
        { title: "Normal", value: "normal" },
        { title: "Heading 2", value: "h2" },
        { title: "Heading 3", value: "h3" },
        { title: "Quote", value: "blockquote" },
      ],
      marks: {
        annotations: [
          {
            name: "link",
            title: "Link",
            type: "object",
            fields: [
              defineField({
                name: "href",
                title: "URL",
                type: "url",
                validation: (rule) =>
                  rule.uri({ allowRelative: true, scheme: ["http", "https", "mailto", "tel"] }),
              }),
            ],
          },
        ],
      },
    }),
  ],
});


export const seoType = defineType({
  name: "seo",
  title: "SEO",
  type: "object",
  fields: [
    defineField({
      name: "title",
      title: "Meta Title",
      type: "string",
      description: "The full title shown in Google and the browser tab, exactly as typed. Empty: the page name plus the site name.",
      validation: (rule) => rule.max(60).warning("Keep search titles at 60 characters or fewer."),
    }),
    defineField({
      name: "description",
      title: "Meta Description",
      type: "text",
      description: "The summary under the title in Google results and link previews.",
      rows: 3,
      validation: (rule) => rule.max(160).warning("Keep descriptions at 160 characters or fewer."),
    }),
    defineField({
      name: "image",
      title: "Social Sharing Image",
      type: "image",
      description: "Shown when the page is shared on WhatsApp, Facebook, X… Empty: the page's main image, then the site default.",
      options: { hotspot: true },
      fields: [
        defineField({
          name: "alt",
          title: "Alternative Text",
          type: "string",
          validation: (rule) => rule.required(),
        }),
      ],
    }),
  ],
});
````

**`sanity/schemaTypes/siteSettings.ts`**

````ts
import { defineArrayMember, defineField, defineType } from "sanity";

export const siteSettings = defineType({
  name: "siteSettings",
  title: "Site Settings",
  type: "document",
  initialValue: {
    title: "My Site",
    contact: {
      email: "hello@example.com",
    },
  },
  fields: [
    defineField({
      name: "title",
      title: "Site Name",
      type: "string",
      validation: (rule) => rule.required(),
    }),
    defineField({ name: "tagline", title: "Tagline", type: "string" }),
    defineField({
      name: "slogan",
      title: "Slogan",
      type: "string",
      description: "A short line used in banners.",
    }),
    defineField({ name: "signature", title: "Signature Line", type: "string" }),
    defineField({
      name: "contact",
      title: "Contact Details",
      type: "object",
      options: { collapsible: true, collapsed: false },
      fields: [
        defineField({ name: "phone", title: "Phone", type: "string" }),
        defineField({ name: "whatsapp", title: "WhatsApp URL", type: "url" }),
        defineField({ name: "email", title: "Email", type: "string", validation: (rule) => rule.email() }),
        defineField({
          name: "addressLines",
          title: "Address Lines",
          type: "array",
          of: [defineArrayMember({ type: "string" })],
        }),
        defineField({ name: "hours", title: "Opening Hours", type: "string" }),
        defineField({ name: "mapUrl", title: "Map Directions URL", type: "url" }),
        defineField({ name: "googleBusinessUrl", title: "Google Business URL", type: "url" }),
      ],
    }),
    defineField({
      name: "socials",
      title: "Social Links",
      type: "array",
      of: [
        defineArrayMember({
          type: "object",
          name: "social",
          fields: [
            defineField({
              name: "platform",
              title: "Platform",
              type: "string",
              options: {
                list: [
                  { title: "Instagram", value: "instagram" },
                  { title: "LinkedIn", value: "linkedin" },
                  { title: "Facebook", value: "facebook" },
                  { title: "X", value: "x" },
                ],
              },
              validation: (rule) => rule.required(),
            }),
            defineField({ name: "url", title: "URL", type: "url", validation: (rule) => rule.required() }),
          ],
          preview: { select: { title: "platform", subtitle: "url" } },
        }),
      ],
    }),
    defineField({ name: "defaultSeo", title: "Default SEO", type: "seo" }),
  ],
  preview: { select: { title: "title", subtitle: "tagline" } },
});
````

**`sanity/schemaTypes/pageSeo.ts`**

````ts
import { defineField, defineType } from "sanity";

export const pageSeo = defineType({
  name: "pageSeo",
  title: "Page SEO",
  type: "document",
  fields: [
    defineField({
      name: "route",
      title: "Website Route",
      type: "string",
      description:
        "For example: / or /about. Blog posts and other documents keep their SEO on their own document.",
      validation: (rule) => rule.required().regex(/^\//, { name: "absolute route" }),
    }),
    defineField({ name: "seo", title: "SEO", type: "seo", validation: (rule) => rule.required() }),
  ],
  preview: { select: { title: "route", subtitle: "seo.title" } },
});
````

**`sanity/schemaTypes/enquiry.ts`**

````ts
import { defineField, defineType } from "sanity";

/**
 * A form submission from the website (contact form, newsletter, …). Created by your
 * form's API route with SANITY_API_WRITE_TOKEN. It's an inbox, not hand-written
 * content: editors move `status` along and keep `internalNotes`, and every submitted
 * field is read-only. Add or rename fields to match your own form.
 */
const ro = { readOnly: true } as const;

export const enquiry = defineType({
  name: "enquiry",
  title: "Enquiry",
  type: "document",
  fields: [
    defineField({
      name: "status",
      title: "Status",
      type: "string",
      options: {
        list: [
          { title: "New", value: "new" },
          { title: "Contacted", value: "contacted" },
          { title: "Closed", value: "closed" },
        ],
        layout: "radio",
      },
      initialValue: "new",
    }),
    defineField({ name: "service", title: "Form", type: "string", ...ro }),
    defineField({ name: "fullName", title: "Name", type: "string", ...ro }),
    defineField({ name: "phone", title: "Phone", type: "string", ...ro }),
    defineField({ name: "email", title: "Email", type: "string", ...ro }),
    defineField({ name: "subject", title: "Subject", type: "string", ...ro }),
    defineField({ name: "message", title: "Message", type: "text", rows: 8, ...ro }),
    defineField({ name: "submittedAt", title: "Submitted at", type: "datetime", ...ro }),
    defineField({
      name: "internalNotes",
      title: "Internal Notes",
      type: "text",
      rows: 4,
      description: "Team notes. Never shown on the website.",
    }),
  ],
  orderings: [
    { title: "Newest first", name: "submittedAtDesc", by: [{ field: "submittedAt", direction: "desc" }] },
  ],
  preview: {
    select: { name: "fullName", email: "email", service: "service", status: "status" },
    prepare: ({ name, email, service, status }) => ({
      title: `${name || email || "Unnamed"}${status === "new" ? " •" : ""}`,
      subtitle: [service, status].filter(Boolean).join(" — "),
    }),
  },
});
````

**`sanity/schemaTypes/errorLog.ts`**

````ts
import { defineField, defineType } from "sanity";

/**
 * Server error captured by Next's `onRequestError` hook (see root `instrumentation.ts`)
 * and written by `app/_lib/error-log.ts`. Surfaced in the Studio's "Health" tool.
 * Repeated errors with the same fingerprint are folded into one document with an
 * incrementing `occurrences` count. Every captured field is read-only; editors can
 * only tick `resolved`.
 */
const ro = { readOnly: true } as const;

export const errorLog = defineType({
  name: "errorLog",
  title: "Error Log",
  type: "document",
  fields: [
    defineField({ name: "message", title: "Message", type: "text", rows: 3, ...ro }),
    defineField({ name: "route", title: "Route", type: "string", ...ro }),
    defineField({ name: "method", title: "Method", type: "string", ...ro }),
    defineField({
      name: "routeType",
      title: "Route type",
      type: "string",
      description: "route (Route Handler) · render · action · proxy",
      ...ro,
    }),
    defineField({ name: "digest", title: "Digest", type: "string", ...ro }),
    defineField({ name: "environment", title: "Environment", type: "string", ...ro }),
    defineField({ name: "stack", title: "Stack", type: "text", rows: 10, ...ro }),
    defineField({ name: "fingerprint", title: "Fingerprint", type: "string", ...ro }),
    defineField({ name: "occurrences", title: "Occurrences", type: "number", initialValue: 1, ...ro }),
    defineField({ name: "firstSeenAt", title: "First seen", type: "datetime", ...ro }),
    defineField({ name: "lastSeenAt", title: "Last seen", type: "datetime", ...ro }),
    defineField({
      name: "resolved",
      title: "Resolved",
      type: "boolean",
      initialValue: false,
      description: "Tick once the underlying issue is fixed, to hide it from the active list.",
    }),
  ],
  orderings: [
    { title: "Last seen", name: "lastSeenDesc", by: [{ field: "lastSeenAt", direction: "desc" }] },
    { title: "Most frequent", name: "occurrencesDesc", by: [{ field: "occurrences", direction: "desc" }] },
  ],
  preview: {
    select: { message: "message", route: "route", occurrences: "occurrences", resolved: "resolved" },
    prepare: ({ message, route, occurrences, resolved }) => ({
      title: `${resolved ? "✓ " : ""}${(message ?? "Unknown error").slice(0, 80)}`,
      subtitle: [route, occurrences ? `×${occurrences}` : null].filter(Boolean).join("  ·  "),
    }),
  },
});
````

**`sanity/schemaTypes/renewal.ts`**

````ts
import { defineField, defineType } from "sanity";

/**
 * Manually tracked renewals / subscriptions (domains bought elsewhere, hosting
 * plans, SaaS seats, paid APIs…) that have no automatic expiry feed. Surfaced in
 * the Studio "Health" tool alongside the auto-detected domain + TLS expiries, so
 * an admin sees what is lapsing soon without logging into each provider.
 */
export const renewal = defineType({
  name: "renewal",
  title: "Renewal / Subscription",
  type: "document",
  fields: [
    defineField({ name: "name", title: "Name", type: "string", validation: (rule) => rule.required() }),
    defineField({
      name: "category",
      title: "Category",
      type: "string",
      options: {
        list: [
          { title: "Domain", value: "domain" },
          { title: "DNS / email", value: "dns" },
          { title: "Hosting", value: "hosting" },
          { title: "TLS certificate", value: "tls" },
          { title: "SaaS subscription", value: "saas" },
          { title: "Paid API / service", value: "api" },
          { title: "Other", value: "other" },
        ],
      },
      initialValue: "saas",
    }),
    defineField({ name: "provider", title: "Provider", type: "string", description: "e.g. GoDaddy, Vercel, Resend" }),
    defineField({
      name: "renewalDate",
      title: "Renews / expires on",
      type: "date",
      options: { dateFormat: "YYYY-MM-DD" },
      validation: (rule) => rule.required(),
    }),
    defineField({
      name: "autoRenews",
      title: "Auto-renews",
      type: "boolean",
      initialValue: true,
      description: "If on, a card on file renews it automatically — shown as a reminder, not an outage.",
    }),
    defineField({ name: "cost", title: "Cost per period", type: "number" }),
    defineField({
      name: "currency",
      title: "Currency",
      type: "string",
      options: { list: ["USD", "EUR", "INR", "GBP", "MAD", "CDF"] },
      initialValue: "USD",
    }),
    defineField({ name: "url", title: "Management URL", type: "url" }),
    defineField({ name: "accountEmail", title: "Account / login", type: "string" }),
    defineField({ name: "notes", title: "Notes", type: "text", rows: 3 }),
    defineField({
      name: "muted",
      title: "Mute warnings",
      type: "boolean",
      initialValue: false,
      description: "Keep the record but stop it affecting the Health status.",
    }),
  ],
  orderings: [
    { title: "Renews soonest", name: "renewalAsc", by: [{ field: "renewalDate", direction: "asc" }] },
  ],
  preview: {
    select: { name: "name", provider: "provider", renewalDate: "renewalDate", muted: "muted" },
    prepare: ({ name, provider, renewalDate, muted }) => ({
      title: `${muted ? "🔕 " : ""}${name ?? "Untitled"}`,
      subtitle: [provider, renewalDate].filter(Boolean).join("  ·  renews "),
    }),
  },
});
````

Check that the schemas compile:

```bash
npx tsc --noEmit
```

---

## 9. `sanity/tools/analytics`

**Location:** `sanity/tools/analytics/`. This is the Studio's **Analytics** tab, and it has two halves:
- **The tab** (`sanity/tools/…`) runs in the browser. It sends your sign-in token to your own endpoint, `/api/admin/google-analytics`.
- **The server half** (`app/api/admin/…` and `app/_lib/…`) checks that token with Sanity, then reads Google Analytics with a service account. The secret keys never reach the browser.

Install the packages and create the folders:

```bash
npm install google-auth-library server-only recharts
```

```bash
mkdir -p sanity/tools/analytics app/_lib app/api/admin/google-analytics
```

### 9a. Shared by all the tabs

`sanity/tools/useStudioToken.ts` reads the signed-in user's token inside the Studio. `app/_lib/studio-auth.ts` checks a token with Sanity on the server.

**`sanity/tools/useStudioToken.ts`**

````ts
import { useMemo } from "react";
import { useClient } from "sanity";

import { apiVersion, projectId } from "@/sanity/env";

/**
 * Reads the current Studio auth token so a tool can call our own `/api/admin/*`
 * routes as the logged-in user. Falls back to the token Sanity persists in
 * localStorage when the client isn't configured with one directly.
 */
export function useStudioToken(): string | null {
  const client = useClient({ apiVersion });
  return useMemo(() => {
    const configured = client.config().token;
    if (configured) return configured;
    if (typeof window === "undefined") return null;
    try {
      const raw = window.localStorage.getItem(`__studio_auth_token_${projectId}`);
      if (!raw) return null;
      const parsed = JSON.parse(raw) as { token?: string };
      return parsed.token ?? null;
    } catch {
      return null;
    }
  }, [client]);
}
````

**`app/_lib/studio-auth.ts`**

````ts
import "server-only";

import type { NextRequest } from "next/server";

import { isSanityConfigured, projectId } from "@/sanity/env";

/**
 * Confirms the caller holds a Sanity session token with access to this project.
 * The admin dashboards render as tools inside the Studio (`/admin`), so the tool
 * forwards its Studio auth token as a Bearer credential and we verify it here.
 */
export async function isAuthorisedStudioUser(request: NextRequest): Promise<boolean> {
  if (!isSanityConfigured) return false;

  const header = request.headers.get("authorization") ?? "";
  const token = header.toLowerCase().startsWith("bearer ") ? header.slice(7).trim() : "";
  if (!token) return false;

  try {
    const response = await fetch(`https://${projectId}.api.sanity.io/v2021-10-21/users/me`, {
      headers: { Authorization: `Bearer ${token}` },
      cache: "no-store",
    });
    if (!response.ok) return false;
    const user = (await response.json()) as { id?: string } | null;
    return Boolean(user?.id);
  } catch {
    return false;
  }
}
````

### 9b. The tab

`index.ts` registers the tab with its name and icon, and `AnalyticsDashboard.tsx` is what the tab shows.

**`sanity/tools/analytics/index.ts`**

````ts
import { BarChartIcon } from "@sanity/icons/BarChart";
import { definePlugin } from "sanity";

import AnalyticsDashboard from "./AnalyticsDashboard";

/**
 * Adds an "Analytics" tab to the Studio (`/admin`) that renders the
 * Google Analytics dashboard. Data is fetched from `/api/admin/google-analytics`,
 * which verifies the caller's Studio session before calling the GA4 Data API.
 */
export const googleAnalyticsTool = definePlugin({
  name: "site-google-analytics",
  tools: [
    {
      name: "analytics",
      title: "Analytics",
      icon: BarChartIcon,
      component: AnalyticsDashboard,
    },
  ],
});
````

**`sanity/tools/analytics/AnalyticsDashboard.tsx`**

````tsx
import { useCallback, useEffect, useMemo, useState } from "react";
import { useCurrentUser } from "sanity";
import {
  Badge,
  Box,
  Button,
  Card,
  Container,
  Flex,
  Grid,
  Heading,
  Inline,
  Spinner,
  Stack,
  Text,
  TextInput,
} from "@sanity/ui";
import { ActivityIcon } from "@sanity/icons/Activity";
import { AddUserIcon } from "@sanity/icons/AddUser";
import { BarChartIcon } from "@sanity/icons/BarChart";
import { BoltIcon } from "@sanity/icons/Bolt";
import { ChartUpwardIcon } from "@sanity/icons/ChartUpward";
import { ClockIcon } from "@sanity/icons/Clock";
import { EyeOpenIcon } from "@sanity/icons/EyeOpen";
import { SyncIcon } from "@sanity/icons/Sync";
import { TrendUpwardIcon } from "@sanity/icons/TrendUpward";
import { UsersIcon } from "@sanity/icons/Users";
import type { ComponentType } from "react";
import {
  CartesianGrid,
  Legend,
  Line,
  LineChart,
  ResponsiveContainer,
  Tooltip,
  XAxis,
  YAxis,
} from "recharts";

import { useStudioToken } from "@/sanity/tools/useStudioToken";
import type {
  GoogleAnalyticsDashboardData,
  GoogleAnalyticsDateRange,
  GoogleAnalyticsRow,
} from "@/app/_lib/google-analytics-types";

type Tab = "pages" | "acquisition" | "events" | "audience";
type ValueFormat = "number" | "percent" | "duration";

interface Column {
  key: string;
  label: string;
  format?: ValueFormat;
  align?: "left" | "right";
}

const SERIES = {
  activeUsers: "#0e4a5b",
  sessions: "#b7975a",
  screenPageViews: "#b76e79",
};

const TABS: { id: Tab; label: string }[] = [
  { id: "pages", label: "Pages" },
  { id: "acquisition", label: "Acquisition" },
  { id: "events", label: "Events" },
  { id: "audience", label: "Audience" },
];

const fullNumber = new Intl.NumberFormat("en-US", { maximumFractionDigits: 2 });
const compactNumber = new Intl.NumberFormat("en-US", {
  notation: "compact",
  maximumFractionDigits: 1,
});

function asNumber(value: string | number | undefined) {
  const n = Number(value ?? 0);
  return Number.isFinite(n) ? n : 0;
}

function formatDuration(value: number) {
  const seconds = Math.max(0, Math.round(value));
  const minutes = Math.floor(seconds / 60);
  const remainder = seconds % 60;
  return minutes > 0 ? `${minutes}m ${remainder}s` : `${remainder}s`;
}

function formatValue(value: string | number | undefined, format: ValueFormat | undefined) {
  if (typeof value === "string" && !format) return value || "—";
  const n = asNumber(value);
  if (format === "percent") {
    return new Intl.NumberFormat("en-US", { style: "percent", maximumFractionDigits: 1 }).format(n);
  }
  if (format === "duration") return formatDuration(n);
  return fullNumber.format(n);
}

function formatGaDate(value: string | number | undefined) {
  const raw = String(value ?? "");
  if (!/^\d{8}$/.test(raw)) return raw;
  const date = new Date(`${raw.slice(0, 4)}-${raw.slice(4, 6)}-${raw.slice(6, 8)}T00:00:00Z`);
  return date.toLocaleDateString("en-US", { day: "numeric", month: "short" });
}

function offsetDate(daysAgo: number) {
  const date = new Date();
  date.setUTCDate(date.getUTCDate() - daysAgo);
  return date.toISOString().slice(0, 10);
}

const PAGE_COLUMNS: Column[] = [
  { key: "pagePathPlusQueryString", label: "Page" },
  { key: "pageTitle", label: "Title" },
  { key: "screenPageViews", label: "Views", format: "number", align: "right" },
  { key: "activeUsers", label: "Users", format: "number", align: "right" },
  { key: "averageEngagementTime", label: "Avg. engagement", format: "duration", align: "right" },
  { key: "eventCount", label: "Events", format: "number", align: "right" },
];

const ACQUISITION_COLUMNS: Column[] = [
  { key: "sessionDefaultChannelGroup", label: "Channel" },
  { key: "sessionSourceMedium", label: "Source / medium" },
  { key: "sessions", label: "Sessions", format: "number", align: "right" },
  { key: "activeUsers", label: "Users", format: "number", align: "right" },
  { key: "newUsers", label: "New", format: "number", align: "right" },
  { key: "engagedSessions", label: "Engaged", format: "number", align: "right" },
  { key: "engagementRate", label: "Engagement", format: "percent", align: "right" },
];

const EVENT_COLUMNS: Column[] = [
  { key: "eventName", label: "Event" },
  { key: "eventCount", label: "Count", format: "number", align: "right" },
  { key: "totalUsers", label: "Users", format: "number", align: "right" },
];

const GEOGRAPHY_COLUMNS: Column[] = [
  { key: "country", label: "Country" },
  { key: "city", label: "City" },
  { key: "activeUsers", label: "Users", format: "number", align: "right" },
  { key: "newUsers", label: "New", format: "number", align: "right" },
  { key: "sessions", label: "Sessions", format: "number", align: "right" },
];

const TECHNOLOGY_COLUMNS: Column[] = [
  { key: "deviceCategory", label: "Device" },
  { key: "browser", label: "Browser" },
  { key: "operatingSystem", label: "OS" },
  { key: "activeUsers", label: "Users", format: "number", align: "right" },
  { key: "sessions", label: "Sessions", format: "number", align: "right" },
  { key: "engagementRate", label: "Engagement", format: "percent", align: "right" },
];

function MetricCard({
  label,
  value,
  sub,
  icon: Icon,
  tone,
}: {
  label: string;
  value: string;
  sub?: string;
  icon: ComponentType;
  tone?: "positive";
}) {
  return (
    <Card padding={3} radius={3} border tone={tone ?? "default"} height="fill">
      <Flex align="flex-start" justify="space-between" gap={2}>
        <Stack gap={3} flex={1}>
          <Text size={1} muted style={{ whiteSpace: "nowrap" }}>
            {label}
          </Text>
          <Heading size={3} style={{ fontVariantNumeric: "tabular-nums" }}>
            {value}
          </Heading>
          {sub ? (
            <Text size={0} muted>
              {sub}
            </Text>
          ) : null}
        </Stack>
        <Box style={{ opacity: 0.55, fontSize: 20, flex: "none" }}>
          <Text size={3}>
            <Icon />
          </Text>
        </Box>
      </Flex>
    </Card>
  );
}

function AnalyticsTable({
  title,
  description,
  rows,
  columns,
}: {
  title: string;
  description: string;
  rows: GoogleAnalyticsRow[];
  columns: Column[];
}) {
  return (
    <Card radius={3} border>
      <Box padding={4} style={{ borderBottom: "1px solid var(--card-border-color)" }}>
        <Flex align="flex-start" justify="space-between" gap={3}>
          <Stack gap={2}>
            <Text weight="semibold">{title}</Text>
            <Text size={1} muted>
              {description}
            </Text>
          </Stack>
          <Badge tone="default" fontSize={0}>
            {fullNumber.format(rows.length)} rows
          </Badge>
        </Flex>
      </Box>

      {rows.length === 0 ? (
        <Box padding={5}>
          <Text align="center" size={1} muted>
            No Google Analytics data for this date range.
          </Text>
        </Box>
      ) : (
        <Box overflow="auto" style={{ maxHeight: 520 }}>
          <table
            style={{
              width: "100%",
              borderCollapse: "collapse",
              fontSize: 13,
              fontVariantNumeric: "tabular-nums",
            }}
          >
            <thead>
              <tr>
                {columns.map((column) => (
                  <th
                    key={column.key}
                    style={{
                      position: "sticky",
                      top: 0,
                      zIndex: 1,
                      textAlign: column.align === "right" ? "right" : "left",
                      padding: "10px 14px",
                      whiteSpace: "nowrap",
                      background: "var(--card-bg-color)",
                      borderBottom: "1px solid var(--card-border-color)",
                      color: "var(--card-muted-fg-color)",
                      fontWeight: 600,
                    }}
                  >
                    {column.label}
                  </th>
                ))}
              </tr>
            </thead>
            <tbody>
              {rows.map((row, index) => (
                <tr key={`${columns.map(({ key }) => row[key]).join("-")}-${index}`}>
                  {columns.map((column) => (
                    <td
                      key={column.key}
                      title={String(row[column.key] ?? "")}
                      style={{
                        textAlign: column.align === "right" ? "right" : "left",
                        padding: "9px 14px",
                        maxWidth: 320,
                        overflow: "hidden",
                        textOverflow: "ellipsis",
                        whiteSpace: "nowrap",
                        borderBottom: "1px solid var(--card-border-color)",
                        color: column.align === "right" ? "var(--card-fg-color)" : undefined,
                      }}
                    >
                      {formatValue(row[column.key], column.format)}
                    </td>
                  ))}
                </tr>
              ))}
            </tbody>
          </table>
        </Box>
      )}
    </Card>
  );
}

function ChartTooltip({
  active,
  payload,
  label,
}: {
  active?: boolean;
  payload?: { name?: string; value?: number; color?: string }[];
  label?: string | number;
}) {
  if (!active || !payload?.length) return null;
  return (
    <Card padding={3} radius={2} shadow={2} style={{ minWidth: 150 }}>
      <Stack gap={3}>
        <Text size={1} weight="semibold">
          {formatGaDate(label)}
        </Text>
        {payload.map((entry) => (
          <Flex key={entry.name} align="center" gap={2} justify="space-between">
            <Flex align="center" gap={2}>
              <span
                style={{
                  width: 8,
                  height: 8,
                  borderRadius: 999,
                  background: entry.color,
                  display: "inline-block",
                }}
              />
              <Text size={1} muted>
                {entry.name}
              </Text>
            </Flex>
            <Text size={1} weight="semibold">
              {fullNumber.format(asNumber(entry.value))}
            </Text>
          </Flex>
        ))}
      </Stack>
    </Card>
  );
}

export default function AnalyticsDashboard() {
  const currentUser = useCurrentUser();
  const token = useStudioToken();

  const defaultRange = useMemo<GoogleAnalyticsDateRange>(
    () => ({ startDate: offsetDate(27), endDate: offsetDate(0) }),
    [],
  );

  const [dateRange, setDateRange] = useState<GoogleAnalyticsDateRange>(defaultRange);
  const [data, setData] = useState<GoogleAnalyticsDashboardData | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);
  const [activeTab, setActiveTab] = useState<Tab>("pages");

  const loadData = useCallback(
    async (range: GoogleAnalyticsDateRange) => {
      if (!range.startDate || !range.endDate) {
        setError("Choose both a start date and an end date.");
        return;
      }
      if (range.startDate > range.endDate) {
        setError("Start date must be on or before end date.");
        return;
      }
      if (!token) {
        setError("Could not read your Studio session. Reload the page and try again.");
        return;
      }

      setLoading(true);
      setError(null);
      try {
        const params = new URLSearchParams({
          startDate: range.startDate,
          endDate: range.endDate,
        });
        const response = await fetch(`/api/admin/google-analytics?${params.toString()}`, {
          cache: "no-store",
          headers: { Authorization: `Bearer ${token}` },
        });
        const payload = await response.json();
        if (!response.ok) {
          throw new Error(payload.error ?? "Unable to load Google Analytics data.");
        }
        setData(payload as GoogleAnalyticsDashboardData);
      } catch (loadError) {
        setError(
          loadError instanceof Error ? loadError.message : "Unable to load Google Analytics data.",
        );
      } finally {
        setLoading(false);
      }
    },
    [token],
  );

  useEffect(() => {
    const id = window.setTimeout(() => {
      if (!token) {
        setError("Could not read your Studio session. Reload the page and try again.");
        return;
      }
      void loadData(defaultRange);
    }, 0);
    return () => window.clearTimeout(id);
  }, [token, defaultRange, loadData]);

  const applyPreset = (days: number) => {
    const nextRange = { startDate: offsetDate(days - 1), endDate: offsetDate(0) };
    setDateRange(nextRange);
    void loadData(nextRange);
  };

  const overview = data?.overview;

  return (
    <Box height="fill" overflow="auto">
      <Container width={4} paddingX={4} paddingY={5}>
        <Stack gap={5}>
          {/* Header */}
          <Flex align="flex-end" justify="space-between" gap={4} wrap="wrap">
            <Stack gap={3}>
              <Flex align="center" gap={2}>
                <Heading size={4}>Google Analytics</Heading>
                <Badge tone="primary" fontSize={0}>
                  GA4
                </Badge>
              </Flex>
              <Text size={1} muted>
                {currentUser?.name ? `Signed in as ${currentUser.name}. ` : ""}
                Traffic and engagement for your site, live from the GA4 Data API.
              </Text>
            </Stack>

            <Flex align="flex-end" gap={3} wrap="wrap">
              <Stack gap={2}>
                <Text size={1} weight="medium" muted>
                  Start date
                </Text>
                <TextInput
                  type="date"
                  fontSize={1}
                  value={dateRange.startDate}
                  max={dateRange.endDate}
                  onChange={(event) => {
                    const startDate = event.currentTarget.value;
                    setDateRange((current) => ({ ...current, startDate }));
                  }}
                />
              </Stack>
              <Stack gap={2}>
                <Text size={1} weight="medium" muted>
                  End date
                </Text>
                <TextInput
                  type="date"
                  fontSize={1}
                  value={dateRange.endDate}
                  min={dateRange.startDate}
                  max={offsetDate(0)}
                  onChange={(event) => {
                    const endDate = event.currentTarget.value;
                    setDateRange((current) => ({ ...current, endDate }));
                  }}
                />
              </Stack>
              <Button
                text="Apply"
                tone="primary"
                icon={SyncIcon}
                loading={loading}
                onClick={() => void loadData(dateRange)}
              />
            </Flex>
          </Flex>

          {/* Quick range */}
          <Flex align="center" gap={2} wrap="wrap">
            <Text size={1} weight="medium" muted>
              Quick range
            </Text>
            {[7, 28, 90].map((days) => (
              <Button
                key={days}
                mode="ghost"
                fontSize={1}
                padding={2}
                text={`${days} days`}
                disabled={loading}
                onClick={() => applyPreset(days)}
              />
            ))}
            {data ? (
              <Box flex={1} style={{ textAlign: "right" }}>
                <Text size={1} muted>
                  {data.property.timeZone} · updated{" "}
                  {new Date(data.generatedAt).toLocaleString("en-US")}
                </Text>
              </Box>
            ) : null}
          </Flex>

          {error ? (
            <Card padding={4} radius={3} tone="caution" border>
              <Stack gap={3}>
                <Text weight="semibold">Google Analytics is not available</Text>
                <Text size={1}>{error}</Text>
              </Stack>
            </Card>
          ) : null}

          {data && overview ? (
            <>
              {/* KPI grid */}
              <Grid gridTemplateColumns={[2, 2, 4]} gap={3}>
                <MetricCard
                  label="Active now"
                  value={fullNumber.format(data.realtime.activeUsers)}
                  sub="Last 30 minutes"
                  icon={ActivityIcon}
                  tone="positive"
                />
                <MetricCard
                  label="Active users"
                  value={fullNumber.format(overview.activeUsers)}
                  icon={UsersIcon}
                />
                <MetricCard
                  label="New users"
                  value={fullNumber.format(overview.newUsers)}
                  icon={AddUserIcon}
                />
                <MetricCard
                  label="Sessions"
                  value={fullNumber.format(overview.sessions)}
                  icon={TrendUpwardIcon}
                />
                <MetricCard
                  label="Page views"
                  value={fullNumber.format(overview.screenPageViews)}
                  icon={EyeOpenIcon}
                />
                <MetricCard
                  label="Engagement rate"
                  value={formatValue(overview.engagementRate, "percent")}
                  icon={BoltIcon}
                />
                <MetricCard
                  label="Avg. session"
                  value={formatDuration(overview.averageSessionDuration)}
                  icon={ClockIcon}
                />
                <MetricCard
                  label="Events"
                  value={fullNumber.format(overview.eventCount)}
                  icon={ChartUpwardIcon}
                />
              </Grid>

              {(data.quality.subjectToThresholding || data.quality.dataLossFromOtherRow) && (
                <Inline gap={2}>
                  {data.quality.subjectToThresholding && (
                    <Badge tone="default" fontSize={0}>
                      Privacy thresholding applied by GA4
                    </Badge>
                  )}
                  {data.quality.dataLossFromOtherRow && (
                    <Badge tone="default" fontSize={0}>
                      High-cardinality data includes an “other” row
                    </Badge>
                  )}
                </Inline>
              )}

              {/* Chart */}
              <Card padding={4} radius={3} border>
                <Stack gap={4}>
                  <Flex align="center" justify="space-between" gap={3}>
                    <Stack gap={2}>
                      <Text weight="semibold">Traffic over time</Text>
                      <Text size={1} muted>
                        Daily active users, sessions, and page views.
                      </Text>
                    </Stack>
                    <Box style={{ opacity: 0.5 }}>
                      <Text size={3}>
                        <BarChartIcon />
                      </Text>
                    </Box>
                  </Flex>
                  <Box style={{ height: 320, width: "100%" }}>
                    {data.daily.length === 0 ? (
                      <Flex align="center" justify="center" style={{ height: "100%" }}>
                        <Text size={1} muted>
                          No daily data for this date range.
                        </Text>
                      </Flex>
                    ) : (
                      <ResponsiveContainer width="100%" height="100%">
                        <LineChart
                          data={data.daily}
                          margin={{ top: 8, right: 12, left: -12, bottom: 0 }}
                        >
                          <CartesianGrid
                            strokeDasharray="3 3"
                            vertical={false}
                            stroke="var(--card-border-color)"
                          />
                          <XAxis
                            dataKey="date"
                            tickFormatter={formatGaDate}
                            minTickGap={28}
                            tickLine={false}
                            axisLine={false}
                            tick={{ fill: "var(--card-muted-fg-color)", fontSize: 12 }}
                          />
                          <YAxis
                            tickFormatter={(value) => compactNumber.format(asNumber(value))}
                            tickLine={false}
                            axisLine={false}
                            width={48}
                            tick={{ fill: "var(--card-muted-fg-color)", fontSize: 12 }}
                          />
                          <Tooltip
                            content={<ChartTooltip />}
                            cursor={{ stroke: "var(--card-border-color)" }}
                          />
                          <Legend
                            iconType="plainline"
                            wrapperStyle={{ fontSize: 12, paddingTop: 8 }}
                          />
                          <Line
                            type="monotone"
                            dataKey="activeUsers"
                            name="Active users"
                            stroke={SERIES.activeUsers}
                            strokeWidth={2}
                            dot={false}
                          />
                          <Line
                            type="monotone"
                            dataKey="sessions"
                            name="Sessions"
                            stroke={SERIES.sessions}
                            strokeWidth={2}
                            dot={false}
                          />
                          <Line
                            type="monotone"
                            dataKey="screenPageViews"
                            name="Page views"
                            stroke={SERIES.screenPageViews}
                            strokeWidth={2}
                            dot={false}
                          />
                        </LineChart>
                      </ResponsiveContainer>
                    )}
                  </Box>
                </Stack>
              </Card>

              {/* Tabs */}
              <Card padding={1} radius={3} border tone="transparent">
                <Inline gap={1}>
                  {TABS.map((tab) => (
                    <Button
                      key={tab.id}
                      mode={activeTab === tab.id ? "default" : "bleed"}
                      tone={activeTab === tab.id ? "primary" : "default"}
                      fontSize={1}
                      padding={3}
                      text={tab.label}
                      onClick={() => setActiveTab(tab.id)}
                    />
                  ))}
                </Inline>
              </Card>

              {activeTab === "pages" && (
                <AnalyticsTable
                  title="Pages and screens"
                  description="Top content ranked by views."
                  rows={data.pages}
                  columns={PAGE_COLUMNS}
                />
              )}
              {activeTab === "acquisition" && (
                <AnalyticsTable
                  title="Traffic acquisition"
                  description="Session channels and source / medium performance."
                  rows={data.acquisition}
                  columns={ACQUISITION_COLUMNS}
                />
              )}
              {activeTab === "events" && (
                <AnalyticsTable
                  title="Events"
                  description="Every reported GA4 event in this range."
                  rows={data.events}
                  columns={EVENT_COLUMNS}
                />
              )}
              {activeTab === "audience" && (
                <Stack gap={5}>
                  <AnalyticsTable
                    title="Audience geography"
                    description="Users and sessions by country and city."
                    rows={data.geography}
                    columns={GEOGRAPHY_COLUMNS}
                  />
                  <AnalyticsTable
                    title="Audience technology"
                    description="Device, browser, and operating-system usage."
                    rows={data.technology}
                    columns={TECHNOLOGY_COLUMNS}
                  />
                </Stack>
              )}
            </>
          ) : null}

          {!data && !error ? (
            <Flex align="center" justify="center" gap={3} style={{ minHeight: 320 }}>
              <Spinner muted />
              <Text size={1} muted>
                Loading Google Analytics…
              </Text>
            </Flex>
          ) : null}
        </Stack>
      </Container>
    </Box>
  );
}
````

### 9c. The server half

- `app/_lib/google-analytics-types.ts`: the shape of the data passed between server and tab.
- `app/_lib/google-analytics.ts`: signs in to Google with the service account and runs the GA4 reports.
- `app/api/admin/google-analytics/route.ts`: the endpoint the tab calls.

**`app/_lib/google-analytics-types.ts`**

````ts
/**
 * Shared Google Analytics dashboard types.
 *
 * Kept free of server-only imports so the Studio tool component
 * (`sanity/tools/analytics`) can import the types without pulling in
 * `app/_lib/google-analytics.ts` (which is `server-only`).
 */

export type GoogleAnalyticsRow = Record<string, string | number>;

export interface GoogleAnalyticsOverview {
  activeUsers: number;
  totalUsers: number;
  newUsers: number;
  sessions: number;
  engagementRate: number;
  averageSessionDuration: number;
  screenPageViews: number;
  eventCount: number;
}

export interface GoogleAnalyticsDateRange {
  startDate: string;
  endDate: string;
}

export interface GoogleAnalyticsDashboardData {
  generatedAt: string;
  dateRange: GoogleAnalyticsDateRange;
  property: {
    id: string;
    timeZone: string;
  };
  quality: {
    subjectToThresholding: boolean;
    dataLossFromOtherRow: boolean;
  };
  realtime: {
    activeUsers: number;
  };
  overview: GoogleAnalyticsOverview;
  daily: GoogleAnalyticsRow[];
  pages: GoogleAnalyticsRow[];
  acquisition: GoogleAnalyticsRow[];
  geography: GoogleAnalyticsRow[];
  technology: GoogleAnalyticsRow[];
  events: GoogleAnalyticsRow[];
}
````

**`app/_lib/google-analytics.ts`**

````ts
import "server-only";

import { JWT } from "google-auth-library";

import type {
  GoogleAnalyticsDashboardData,
  GoogleAnalyticsOverview,
  GoogleAnalyticsRow,
} from "@/app/_lib/google-analytics-types";

const ANALYTICS_SCOPE = "https://www.googleapis.com/auth/analytics.readonly";
const DATA_API = "https://analyticsdata.googleapis.com/v1beta";

/** Thrown when the GA env vars are missing or malformed — surfaced to the UI as a setup hint. */
export class GoogleAnalyticsConfigurationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "GoogleAnalyticsConfigurationError";
  }
}

const toIsoDate = (date: Date) => date.toISOString().slice(0, 10);

export function getDefaultGoogleAnalyticsDateRange() {
  const end = new Date();
  const start = new Date(end);
  start.setUTCDate(start.getUTCDate() - 27);
  return { startDate: toIsoDate(start), endDate: toIsoDate(end) };
}

function getConfiguration() {
  const propertyId = process.env.GOOGLE_ANALYTICS_PROPERTY_ID
    ?.trim()
    .replace(/^properties\//, "");
  const email = process.env.GOOGLE_ANALYTICS_CLIENT_EMAIL?.trim();
  const privateKey = process.env.GOOGLE_ANALYTICS_PRIVATE_KEY?.replace(/\\n/g, "\n");

  const missing = [
    !propertyId && "GOOGLE_ANALYTICS_PROPERTY_ID",
    !email && "GOOGLE_ANALYTICS_CLIENT_EMAIL",
    !privateKey && "GOOGLE_ANALYTICS_PRIVATE_KEY",
  ].filter(Boolean);

  if (missing.length > 0) {
    throw new GoogleAnalyticsConfigurationError(
      `Missing Google Analytics configuration: ${missing.join(", ")}.`,
    );
  }

  if (!/^\d+$/.test(propertyId as string)) {
    throw new GoogleAnalyticsConfigurationError(
      "GOOGLE_ANALYTICS_PROPERTY_ID must be the numeric GA4 property ID (Admin → Property Settings).",
    );
  }

  return {
    propertyId: propertyId as string,
    email: email as string,
    privateKey: privateKey as string,
  };
}

async function getAccessToken(email: string, privateKey: string) {
  const jwt = new JWT({ email, key: privateKey, scopes: [ANALYTICS_SCOPE] });
  const { access_token: accessToken } = await jwt.authorize();
  if (!accessToken) {
    throw new Error("Google rejected the service-account credentials (no access token returned).");
  }
  return accessToken;
}

/**
 * Lightweight liveness probe for the health dashboard: authorises the service
 * account and fetches the property metadata. Throws `GoogleAnalyticsConfigurationError`
 * when unconfigured, or a generic Error when the credentials / property access fail.
 */
export async function pingGoogleAnalytics(): Promise<{ propertyId: string }> {
  const { propertyId, email, privateKey } = getConfiguration();
  const accessToken = await getAccessToken(email, privateKey);
  const response = await fetch(`${DATA_API}/properties/${propertyId}/metadata`, {
    headers: { Authorization: `Bearer ${accessToken}` },
    cache: "no-store",
  });
  if (!response.ok) {
    const detail = await response.text().catch(() => "");
    throw new Error(`GA metadata check failed (${response.status}): ${detail.slice(0, 300)}`);
  }
  return { propertyId };
}

/* ---- GA Data API response shapes (only the fields we read) ---- */

interface GaHeader {
  name?: string;
}
interface GaReportRow {
  dimensionValues?: { value?: string }[];
  metricValues?: { value?: string }[];
}
interface GaReport {
  dimensionHeaders?: GaHeader[];
  metricHeaders?: GaHeader[];
  rows?: GaReportRow[];
  metadata?: {
    timeZone?: string;
    subjectToThresholding?: boolean;
    dataLossFromOtherRow?: boolean;
  };
}

async function gaFetch<T>(
  accessToken: string,
  propertyId: string,
  method: "batchRunReports" | "runRealtimeReport",
  body: unknown,
): Promise<T> {
  const response = await fetch(`${DATA_API}/properties/${propertyId}:${method}`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${accessToken}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
    cache: "no-store",
  });

  if (!response.ok) {
    const detail = await response.text().catch(() => "");
    throw new Error(
      `Google Analytics Data API ${method} failed (${response.status}): ${detail.slice(0, 500)}`,
    );
  }

  return (await response.json()) as T;
}

function parseReport(report: GaReport | undefined): GoogleAnalyticsRow[] {
  if (!report) return [];
  const dimensions = report.dimensionHeaders ?? [];
  const metrics = report.metricHeaders ?? [];

  return (report.rows ?? []).map((row) => {
    const parsed: GoogleAnalyticsRow = {};
    dimensions.forEach((header, index) => {
      if (!header.name) return;
      parsed[header.name] = row.dimensionValues?.[index]?.value ?? "";
    });
    metrics.forEach((header, index) => {
      if (!header.name) return;
      const value = row.metricValues?.[index]?.value;
      parsed[header.name] =
        value === undefined || value === null || value === "" ? 0 : Number(value);
    });
    return parsed;
  });
}

function metric(row: GoogleAnalyticsRow | undefined, name: string): number {
  const value = row?.[name];
  return typeof value === "number" && Number.isFinite(value) ? value : Number(value ?? 0) || 0;
}

const namedMetric = (name: string) => ({ name });
const dimension = (name: string) => ({ name });
const metricOrder = (metricName: string) => [{ metric: { metricName }, desc: true }];

type ReportRequest = {
  dimensions?: { name: string }[];
  metrics: { name: string }[];
  dateRanges: { startDate: string; endDate: string }[];
  orderBys?: unknown[];
  limit?: string;
};

const withRange = (
  startDate: string,
  endDate: string,
  request: Omit<ReportRequest, "dateRanges">,
): ReportRequest => ({ ...request, dateRanges: [{ startDate, endDate }] });

export async function getGoogleAnalyticsDashboard(
  startDate: string,
  endDate: string,
): Promise<GoogleAnalyticsDashboardData> {
  const { propertyId, email, privateKey } = getConfiguration();
  const accessToken = await getAccessToken(email, privateKey);

  const requests: ReportRequest[] = [
    // 0 — overview totals
    withRange(startDate, endDate, {
      metrics: [
        "activeUsers",
        "totalUsers",
        "newUsers",
        "sessions",
        "engagementRate",
        "averageSessionDuration",
        "screenPageViews",
        "eventCount",
      ].map(namedMetric),
    }),
    // 1 — daily series
    withRange(startDate, endDate, {
      dimensions: [dimension("date")],
      metrics: ["activeUsers", "newUsers", "sessions", "screenPageViews", "eventCount"].map(
        namedMetric,
      ),
      orderBys: [{ dimension: { dimensionName: "date" } }],
      limit: "400",
    }),
    // 2 — pages
    withRange(startDate, endDate, {
      dimensions: [dimension("pagePathPlusQueryString"), dimension("pageTitle")],
      metrics: [
        namedMetric("screenPageViews"),
        namedMetric("activeUsers"),
        { name: "averageEngagementTime", expression: "userEngagementDuration/activeUsers" } as {
          name: string;
        },
        namedMetric("eventCount"),
      ],
      orderBys: metricOrder("screenPageViews"),
      limit: "100",
    }),
    // 3 — acquisition
    withRange(startDate, endDate, {
      dimensions: [dimension("sessionDefaultChannelGroup"), dimension("sessionSourceMedium")],
      metrics: [
        "sessions",
        "activeUsers",
        "newUsers",
        "engagedSessions",
        "engagementRate",
      ].map(namedMetric),
      orderBys: metricOrder("sessions"),
      limit: "100",
    }),
    // 4 — geography
    withRange(startDate, endDate, {
      dimensions: [dimension("country"), dimension("city")],
      metrics: ["activeUsers", "newUsers", "sessions"].map(namedMetric),
      orderBys: metricOrder("activeUsers"),
      limit: "100",
    }),
    // 5 — technology
    withRange(startDate, endDate, {
      dimensions: [
        dimension("deviceCategory"),
        dimension("browser"),
        dimension("operatingSystem"),
      ],
      metrics: ["activeUsers", "sessions", "engagementRate"].map(namedMetric),
      orderBys: metricOrder("activeUsers"),
      limit: "100",
    }),
    // 6 — events
    withRange(startDate, endDate, {
      dimensions: [dimension("eventName")],
      metrics: ["eventCount", "totalUsers"].map(namedMetric),
      orderBys: metricOrder("eventCount"),
      limit: "100",
    }),
  ];

  // The GA Data API caps batchRunReports at 5 requests per call, so chunk them.
  const BATCH_LIMIT = 5;
  const batches: ReportRequest[][] = [];
  for (let i = 0; i < requests.length; i += BATCH_LIMIT) {
    batches.push(requests.slice(i, i + BATCH_LIMIT));
  }

  const [batchResults, realtime] = await Promise.all([
    Promise.all(
      batches.map((chunk) =>
        gaFetch<{ reports?: GaReport[] }>(accessToken, propertyId, "batchRunReports", {
          requests: chunk,
        }),
      ),
    ),
    gaFetch<GaReport>(accessToken, propertyId, "runRealtimeReport", {
      metrics: [namedMetric("activeUsers")],
    }),
  ]);

  const reports = batchResults.flatMap((result) => result.reports ?? []);
  if (reports.length !== requests.length) {
    throw new Error("Google Analytics returned an incomplete batch response.");
  }

  const [
    overviewReport,
    dailyReport,
    pagesReport,
    acquisitionReport,
    geographyReport,
    technologyReport,
    eventsReport,
  ] = reports;

  const overviewRow = parseReport(overviewReport)[0];
  const realtimeRow = parseReport(realtime)[0];

  const overview: GoogleAnalyticsOverview = {
    activeUsers: metric(overviewRow, "activeUsers"),
    totalUsers: metric(overviewRow, "totalUsers"),
    newUsers: metric(overviewRow, "newUsers"),
    sessions: metric(overviewRow, "sessions"),
    engagementRate: metric(overviewRow, "engagementRate"),
    averageSessionDuration: metric(overviewRow, "averageSessionDuration"),
    screenPageViews: metric(overviewRow, "screenPageViews"),
    eventCount: metric(overviewRow, "eventCount"),
  };

  return {
    generatedAt: new Date().toISOString(),
    dateRange: { startDate, endDate },
    property: {
      id: propertyId,
      timeZone: overviewReport?.metadata?.timeZone ?? "UTC",
    },
    quality: {
      subjectToThresholding: reports.some((r) => r.metadata?.subjectToThresholding === true),
      dataLossFromOtherRow: reports.some((r) => r.metadata?.dataLossFromOtherRow === true),
    },
    realtime: {
      activeUsers: metric(realtimeRow, "activeUsers"),
    },
    overview,
    daily: parseReport(dailyReport),
    pages: parseReport(pagesReport),
    acquisition: parseReport(acquisitionReport),
    geography: parseReport(geographyReport),
    technology: parseReport(technologyReport),
    events: parseReport(eventsReport),
  };
}
````

**`app/api/admin/google-analytics/route.ts`**

````ts
import { NextResponse, type NextRequest } from "next/server";

import {
  getGoogleAnalyticsDashboard,
  GoogleAnalyticsConfigurationError,
} from "@/app/_lib/google-analytics";
import { isAuthorisedStudioUser } from "@/app/_lib/studio-auth";

export const runtime = "nodejs";
export const dynamic = "force-dynamic";

const ISO_DATE = /^\d{4}-\d{2}-\d{2}$/;

export async function GET(request: NextRequest) {
  if (!(await isAuthorisedStudioUser(request))) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const startDate = request.nextUrl.searchParams.get("startDate") ?? "";
  const endDate = request.nextUrl.searchParams.get("endDate") ?? "";

  if (!ISO_DATE.test(startDate) || !ISO_DATE.test(endDate)) {
    return NextResponse.json(
      { error: "startDate and endDate must be YYYY-MM-DD." },
      { status: 400 },
    );
  }
  if (startDate > endDate) {
    return NextResponse.json(
      { error: "Start date must be on or before end date." },
      { status: 400 },
    );
  }

  try {
    const data = await getGoogleAnalyticsDashboard(startDate, endDate);
    return NextResponse.json(data, {
      headers: { "Cache-Control": "private, no-store" },
    });
  } catch (error) {
    if (error instanceof GoogleAnalyticsConfigurationError) {
      return NextResponse.json({ error: error.message }, { status: 503 });
    }
    console.error("[google-analytics] report failed", error);
    return NextResponse.json(
      {
        error:
          "Google Analytics could not be loaded. Check that the service account has Viewer access to the GA4 property and that the Data API is enabled.",
      },
      { status: 502 },
    );
  }
}
````

---

## 10. Analytics environment variables: which ones and how to get them

**Location:** `.env.local`, and the same four on Vercel → **Settings** → **Environment Variables**.

| Variable | What it does |
|---|---|
| `GOOGLE_ANALYTICS_MEASUREMENT_ID` | The tracking tag on the public site, which collects the visits. It looks like `G-XXXXXXXXXX`. |
| `GOOGLE_ANALYTICS_PROPERTY_ID` | Which GA4 property the Analytics tab reads. A number, e.g. `123456789`. |
| `GOOGLE_ANALYTICS_CLIENT_EMAIL` | The service account's email, which the tab's server half signs in as |
| `GOOGLE_ANALYTICS_PRIVATE_KEY` | The service account's private key |

**Step 1: Create the GA4 property and get the Measurement ID**
1. Go to https://analytics.google.com → **Admin** (the gear icon) → **Create** → **Property**. Enter a name, your time zone and currency, then click **Create**.
2. Choose platform **Web** and enter your domain. This creates a data stream.
3. Open **Admin** → **Data streams** → your web stream. Copy the **Measurement ID** (`G-XXXXXXXXXX`) into `GOOGLE_ANALYTICS_MEASUREMENT_ID`, and add the tracking tag to your site's layout.

**Step 2: Get the Property ID**

Go to **Admin** → **Property details** (or **Property settings**) and copy the **Property ID** number into `GOOGLE_ANALYTICS_PROPERTY_ID`.

**Step 3: Create the service account in Google Cloud**
1. Go to https://console.cloud.google.com and create a project, or pick an existing one.
2. Go to **APIs & Services** → **Library**, search for **Google Analytics Data API** and click **Enable**.
3. Go to **IAM & Admin** → **Service accounts** → **Create service account**. Name it e.g. `analytics-reader`, then click **Create and continue** → **Done**. No roles are needed here.
4. Open the new service account → **Keys** → **Add key** → **Create new key** → **JSON** → **Create**. A `.json` file downloads. Keep it private and never commit it.

**Step 4: Give the service account access to the property**

In Google Analytics → **Admin** → **Property access management** → **+** → **Add users**, paste the service account's email (the `client_email` in the JSON), choose the role **Viewer**, untick "Notify new users", and click **Add**.

**Step 5: Copy the values from the JSON key into `.env.local`**

```bash
GOOGLE_ANALYTICS_MEASUREMENT_ID=G-XXXXXXXXXX
GOOGLE_ANALYTICS_PROPERTY_ID=123456789
GOOGLE_ANALYTICS_CLIENT_EMAIL=analytics-reader@your-gcp-project.iam.gserviceaccount.com
GOOGLE_ANALYTICS_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nMIIE...\n-----END PRIVATE KEY-----\n"
```

- `GOOGLE_ANALYTICS_CLIENT_EMAIL` is the `client_email` value from the JSON.
- `GOOGLE_ANALYTICS_PRIVATE_KEY` is the `private_key` value from the JSON. Keep it on one line, inside double quotes, with the `\n` sequences exactly as they appear in the JSON.

To print the two values from the downloaded key (change the path to where your file is):

```bash
node -e "const k=require(process.argv[1]); console.log(k.client_email); console.log(JSON.stringify(k.private_key))" ~/Downloads/your-key.json
```

Restart the dev server so it picks up the new values:

```bash
npm run dev
```

Open http://localhost:3000/admin → **Analytics**. A new property shows zeros until visits come in, which usually takes a day.

**On Vercel:** add the same four variables and redeploy. Paste the private key without the surrounding quotes, keeping the `\n` sequences.

---

## 11. `sanity/tools/health`

**Location:** `sanity/tools/health/`. This is the Studio's **Health** tab. It shows:
- whether the environment variables are set;
- Sanity read and write checks;
- the email (Resend) and Google Analytics checks;
- page checks on the live site;
- domain and SSL certificate expiry;
- recent server errors;
- the renewals you enter by hand.

It uses the same token check as Analytics (section 9a).

```bash
mkdir -p sanity/tools/health app/api/admin/health
```

### 11a. The tab

**`sanity/tools/health/index.ts`**

````ts
import { ActivityIcon } from "@sanity/icons/Activity";
import { definePlugin } from "sanity";

import HealthDashboard from "./HealthDashboard";

/**
 * Adds a "Health" tab to the Studio (`/admin`) with live service checks and a
 * feed of recent server errors. Data comes from `/api/admin/health`, which
 * verifies the caller's Studio session.
 */
export const healthTool = definePlugin({
  name: "site-health",
  tools: [
    {
      name: "health",
      title: "Health",
      icon: ActivityIcon,
      component: HealthDashboard,
    },
  ],
});
````

**`sanity/tools/health/HealthDashboard.tsx`**

````tsx
import { useCallback, useEffect, useMemo, useState } from "react";
import { useClient, useCurrentUser } from "sanity";
import {
  Badge,
  Box,
  Button,
  Card,
  Container,
  Flex,
  Grid,
  Heading,
  Inline,
  Spinner,
  Stack,
  Text,
} from "@sanity/ui";
import { CheckmarkIcon } from "@sanity/icons/Checkmark";
import { SyncIcon } from "@sanity/icons/Sync";

import { apiVersion } from "@/sanity/env";
import { useStudioToken } from "@/sanity/tools/useStudioToken";
import type {
  HealthReport,
  HealthStatus,
  LoggedError,
  RenewalItem,
} from "@/app/_lib/health-types";

const STATUS_COLOR: Record<HealthStatus, string> = {
  ok: "#3fb950",
  degraded: "#d29922",
  down: "#f85149",
  not_configured: "#8b949e",
};

const STATUS_LABEL: Record<HealthStatus, string> = {
  ok: "Operational",
  degraded: "Degraded",
  down: "Down",
  not_configured: "Not configured",
};

const OVERALL_HEADLINE: Record<HealthStatus, string> = {
  ok: "All systems operational",
  degraded: "Some services degraded",
  down: "Outage detected",
  not_configured: "Setup incomplete",
};

function Dot({ status }: { status: HealthStatus }) {
  return (
    <span
      aria-hidden
      style={{
        width: 10,
        height: 10,
        borderRadius: 999,
        flex: "none",
        display: "inline-block",
        background: STATUS_COLOR[status],
        boxShadow: `0 0 0 3px ${STATUS_COLOR[status]}22`,
      }}
    />
  );
}

function latency(ms: number | null) {
  if (ms == null) return "—";
  return ms >= 1000 ? `${(ms / 1000).toFixed(2)} s` : `${ms} ms`;
}

function timeAgo(iso: string) {
  const diff = Date.now() - new Date(iso).getTime();
  const mins = Math.round(diff / 60000);
  if (mins < 1) return "just now";
  if (mins < 60) return `${mins}m ago`;
  const hours = Math.round(mins / 60);
  if (hours < 24) return `${hours}h ago`;
  return `${Math.round(hours / 24)}d ago`;
}

function formatDate(iso: string | null) {
  if (!iso) return "—";
  return new Date(iso).toLocaleDateString("en-US", {
    day: "numeric",
    month: "short",
    year: "numeric",
  });
}

function countdown(daysLeft: number | null) {
  if (daysLeft == null) return "unknown";
  if (daysLeft < 0) return `${Math.abs(daysLeft)}d ago`;
  if (daysLeft === 0) return "today";
  if (daysLeft === 1) return "tomorrow";
  return `in ${daysLeft}d`;
}

const th: React.CSSProperties = {
  position: "sticky",
  top: 0,
  zIndex: 1,
  textAlign: "left",
  padding: "10px 14px",
  whiteSpace: "nowrap",
  background: "var(--card-bg-color)",
  borderBottom: "1px solid var(--card-border-color)",
  color: "var(--card-muted-fg-color)",
  fontWeight: 600,
};

const td: React.CSSProperties = {
  padding: "9px 14px",
  borderBottom: "1px solid var(--card-border-color)",
  verticalAlign: "top",
};

function ErrorRow({ error, onResolved }: { error: LoggedError; onResolved: (id: string) => void }) {
  const [open, setOpen] = useState(false);
  const [saving, setSaving] = useState(false);
  const [failed, setFailed] = useState(false);
  // Written as the logged-in editor, the same as ticking "Resolved" in the Error Log form.
  const client = useClient({ apiVersion });

  const resolve = async (event: React.MouseEvent) => {
    event.stopPropagation(); // the row itself toggles the stack trace
    setSaving(true);
    setFailed(false);
    try {
      await client.patch(error._id).set({ resolved: true }).commit();
      onResolved(error._id);
    } catch {
      setFailed(true);
      setSaving(false);
    }
  };

  return (
    <>
      <tr onClick={() => setOpen((v) => !v)} style={{ cursor: error.stack ? "pointer" : "default" }}>
        <td style={{ ...td, whiteSpace: "nowrap", color: "var(--card-muted-fg-color)" }}>
          {timeAgo(error.lastSeenAt)}
        </td>
        <td style={{ ...td, whiteSpace: "nowrap", fontVariantNumeric: "tabular-nums" }}>
          <Inline gap={2}>
            <span>{error.method || "—"}</span>
            <span style={{ color: "var(--card-muted-fg-color)" }}>{error.route || "unknown"}</span>
          </Inline>
        </td>
        <td style={{ ...td, textAlign: "right", fontVariantNumeric: "tabular-nums" }}>
          {error.occurrences > 1 ? `×${error.occurrences}` : "1"}
        </td>
        <td style={{ ...td, maxWidth: 520 }}>
          <Text size={1} style={{ wordBreak: "break-word" }}>
            {error.message || "(no message)"}
          </Text>
          {error.environment ? (
            <Box marginTop={2}>
              <Badge fontSize={0} tone="default">
                {error.environment}
              </Badge>
            </Box>
          ) : null}
        </td>
        <td style={{ ...td, textAlign: "right", whiteSpace: "nowrap" }}>
          <Button
            icon={CheckmarkIcon}
            text={failed ? "Retry" : "Resolve"}
            tone={failed ? "critical" : "positive"}
            mode="ghost"
            fontSize={1}
            padding={2}
            loading={saving}
            onClick={resolve}
            title={failed ? "Could not save — check you have edit rights, then try again." : "Mark as fixed and hide it from this list"}
          />
        </td>
      </tr>
      {open && error.stack ? (
        <tr>
          <td colSpan={5} style={{ ...td, background: "var(--card-code-bg-color, rgba(127,127,127,0.06))" }}>
            <pre
              style={{
                margin: 0,
                fontSize: 12,
                lineHeight: 1.5,
                whiteSpace: "pre-wrap",
                wordBreak: "break-word",
                color: "var(--card-muted-fg-color)",
              }}
            >
              {error.stack}
            </pre>
          </td>
        </tr>
      ) : null}
    </>
  );
}

function categoryLabel(category: string) {
  const map: Record<string, string> = {
    domain: "Domain",
    dns: "DNS / email",
    hosting: "Hosting",
    tls: "TLS cert",
    saas: "Subscription",
    api: "Paid API",
    other: "Other",
  };
  return map[category] ?? category;
}

function RenewalsSection({ renewals }: { renewals: HealthReport["renewals"] }) {
  const items = renewals.items;
  return (
    <Stack gap={3}>
      <Flex align="center" gap={2} wrap="wrap">
        <Text weight="semibold" size={1}>
          Renewals &amp; expiry
        </Text>
        {renewals.monitoredDomains.length ? (
          <Text size={0} muted>
            auto-checking {renewals.monitoredDomains.join(", ")}
          </Text>
        ) : null}
      </Flex>
      <Card radius={3} border>
        {items.length === 0 ? (
          <Box padding={4}>
            <Text size={1} muted>
              Nothing tracked yet. Add records under “Renewals &amp; Subscriptions”, or set
              HEALTH_DOMAINS to auto-check domain and certificate expiry.
            </Text>
          </Box>
        ) : (
          <Box overflow="auto" style={{ maxHeight: 480 }}>
            <table style={{ width: "100%", borderCollapse: "collapse", fontSize: 13 }}>
              <thead>
                <tr>
                  <th style={th}>Item</th>
                  <th style={th}>Provider</th>
                  <th style={th}>Expires</th>
                  <th style={th}>Renewal</th>
                  <th style={th}>Status</th>
                </tr>
              </thead>
              <tbody>
                {items.map((item: RenewalItem) => (
                  <tr key={item.id}>
                    <td style={{ ...td }}>
                      <Flex align="center" gap={2} wrap="wrap">
                        <Text size={1}>{item.name}</Text>
                        <Badge fontSize={0} tone="default">
                          {item.source === "auto" ? "auto" : categoryLabel(item.category)}
                        </Badge>
                        {item.url ? (
                          <a
                            href={item.url}
                            target="_blank"
                            rel="noreferrer"
                            style={{ fontSize: 12, color: "var(--card-link-color, #4a9eff)" }}
                          >
                            open ↗
                          </a>
                        ) : null}
                      </Flex>
                      {item.detail ? (
                        <Text size={0} muted style={{ marginTop: 4, display: "block" }}>
                          {item.detail}
                        </Text>
                      ) : null}
                    </td>
                    <td style={{ ...td, whiteSpace: "nowrap", color: "var(--card-muted-fg-color)" }}>
                      {item.provider || "—"}
                      {item.cost != null ? (
                        <div style={{ fontSize: 12 }}>
                          {item.cost} {item.currency}
                        </div>
                      ) : null}
                    </td>
                    <td style={{ ...td, whiteSpace: "nowrap" }}>
                      {formatDate(item.expiresAt)}
                      <div style={{ fontSize: 12, color: "var(--card-muted-fg-color)" }}>
                        {countdown(item.daysLeft)}
                      </div>
                    </td>
                    <td style={{ ...td, whiteSpace: "nowrap", color: "var(--card-muted-fg-color)" }}>
                      {item.autoRenews ? "automatic" : "manual"}
                    </td>
                    <td style={td}>
                      <Flex align="center" gap={2}>
                        <Dot status={item.status} />
                        <span>{STATUS_LABEL[item.status]}</span>
                      </Flex>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </Box>
        )}
      </Card>
    </Stack>
  );
}

export default function HealthDashboard() {
  const currentUser = useCurrentUser();
  const token = useStudioToken();

  const [report, setReport] = useState<HealthReport | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  const load = useCallback(async () => {
    if (!token) {
      setError("Could not read your Studio session. Reload the page and try again.");
      return;
    }
    setLoading(true);
    setError(null);
    try {
      const response = await fetch("/api/admin/health", {
        cache: "no-store",
        headers: { Authorization: `Bearer ${token}` },
      });
      const payload = await response.json();
      if (!response.ok) {
        throw new Error(payload.error ?? "Unable to run the health check.");
      }
      setReport(payload as HealthReport);
    } catch (loadError) {
      setError(loadError instanceof Error ? loadError.message : "Unable to run the health check.");
    } finally {
      setLoading(false);
    }
  }, [token]);

  useEffect(() => {
    const id = window.setTimeout(() => {
      if (!token) {
        setError("Could not read your Studio session. Reload the page and try again.");
        return;
      }
      void load();
    }, 0);
    return () => window.clearTimeout(id);
  }, [token, load]);

  // Rows resolved in this session drop out at once, without re-running every health check.
  const [resolvedIds, setResolvedIds] = useState<string[]>([]);
  const markResolved = useCallback((id: string) => setResolvedIds((ids) => [...ids, id]), []);
  const errorItems = (report?.errors.items ?? []).filter((item) => !resolvedIds.includes(item._id));
  const errorCounts = useMemo(() => {
    const all = report?.errors.items ?? [];
    const gone = all.filter((item) => resolvedIds.includes(item._id));
    const within = (days: number) =>
      gone.filter((item) => Date.now() - new Date(item.lastSeenAt).getTime() < days * 86_400_000).length;
    return {
      last24h: Math.max(0, (report?.errors.last24h ?? 0) - within(1)),
      last7d: Math.max(0, (report?.errors.last7d ?? 0) - within(7)),
      unresolvedTotal: Math.max(0, (report?.errors.unresolvedTotal ?? 0) - gone.length),
    };
  }, [report, resolvedIds]);
  // The server's verdict, except that resolving the last error here lifts the
  // "degraded" it caused — the same rule as `overall` in app/_lib/health.ts.
  const overall = useMemo<HealthStatus>(() => {
    if (!report) return "not_configured";
    if (report.overall !== "degraded" || errorCounts.unresolvedTotal > 0) return report.overall;
    const others = [
      ...report.checks.map((c) => c.status),
      ...report.pages.probes.map((p) => p.status),
      ...report.renewals.items.map((r) => r.status),
    ];
    return others.some((s) => s === "degraded" || s === "down") ? report.overall : "ok";
  }, [report, errorCounts.unresolvedTotal]);
  const sortedChecks = useMemo(
    () =>
      [...(report?.checks ?? [])].sort(
        (a, b) =>
          ["down", "degraded", "not_configured", "ok"].indexOf(a.status) -
          ["down", "degraded", "not_configured", "ok"].indexOf(b.status),
      ),
    [report],
  );

  return (
    <Box height="fill" overflow="auto">
      <Container width={4} paddingX={4} paddingY={5}>
        <Stack gap={5}>
          {/* Header */}
          <Flex align="flex-end" justify="space-between" gap={4} wrap="wrap">
            <Stack gap={3}>
              <Heading size={4}>System Health</Heading>
              <Text size={1} muted>
                {currentUser?.name ? `Signed in as ${currentUser.name}. ` : ""}
                Live checks of the CMS, email, analytics, and public site, plus recent server errors.
              </Text>
            </Stack>
            <Flex align="center" gap={3} wrap="wrap">
              {report ? (
                <Text size={1} muted>
                  Checked {timeAgo(report.generatedAt)}
                </Text>
              ) : null}
              <Button
                text="Refresh"
                tone="primary"
                icon={SyncIcon}
                loading={loading}
                onClick={() => void load()}
              />
            </Flex>
          </Flex>

          {/* Overall banner */}
          {report ? (
            <Card
              padding={4}
              radius={3}
              border
              tone={
                overall === "ok"
                  ? "positive"
                  : overall === "degraded"
                    ? "caution"
                    : overall === "down"
                      ? "critical"
                      : "default"
              }
            >
              <Flex align="center" gap={3}>
                <Dot status={overall} />
                <Stack gap={2}>
                  <Text weight="semibold">{OVERALL_HEADLINE[overall]}</Text>
                  <Text size={1} muted>
                    {errorCounts.unresolvedTotal > 0
                      ? `${errorCounts.unresolvedTotal} unresolved error${
                          errorCounts.unresolvedTotal === 1 ? "" : "s"
                        } · `
                      : ""}
                    {report.checks.filter((c) => c.status === "ok").length}/{report.checks.length}{" "}
                    services OK
                  </Text>
                </Stack>
              </Flex>
            </Card>
          ) : null}

          {error ? (
            <Card padding={4} radius={3} tone="caution" border>
              <Stack gap={3}>
                <Text weight="semibold">Health check unavailable</Text>
                <Text size={1}>{error}</Text>
              </Stack>
            </Card>
          ) : null}

          {report ? (
            <>
              {/* Service checks */}
              <Stack gap={3}>
                <Text weight="semibold" size={1}>
                  Services
                </Text>
                <Grid gridTemplateColumns={[1, 2, 3]} gap={3}>
                  {sortedChecks.map((check) => (
                    <Card key={check.id} padding={3} radius={3} border height="fill">
                      <Stack gap={3}>
                        <Flex align="center" gap={2} justify="space-between">
                          <Flex align="center" gap={2}>
                            <Dot status={check.status} />
                            <Text size={1} weight="medium">
                              {check.label}
                            </Text>
                          </Flex>
                          <Text size={0} muted style={{ fontVariantNumeric: "tabular-nums" }}>
                            {check.latencyMs != null ? latency(check.latencyMs) : STATUS_LABEL[check.status]}
                          </Text>
                        </Flex>
                        <Text size={0} muted style={{ lineHeight: 1.5 }}>
                          {check.detail}
                        </Text>
                      </Stack>
                    </Card>
                  ))}
                </Grid>
              </Stack>

              {/* Public pages */}
              <Stack gap={3}>
                <Flex align="center" justify="space-between" gap={3}>
                  <Text weight="semibold" size={1}>
                    Public pages
                  </Text>
                  {report.pages.baseUrl ? (
                    <Text size={0} muted>
                      {report.pages.baseUrl}
                    </Text>
                  ) : null}
                </Flex>
                <Card radius={3} border>
                  {report.pages.probes.length === 0 ? (
                    <Box padding={4}>
                      <Text size={1} muted>
                        Set NEXT_PUBLIC_SITE_URL to probe live pages.
                      </Text>
                    </Box>
                  ) : (
                    <Box overflow="auto">
                      <table
                        style={{
                          width: "100%",
                          borderCollapse: "collapse",
                          fontSize: 13,
                        }}
                      >
                        <thead>
                          <tr>
                            <th style={th}>Path</th>
                            <th style={th}>Status</th>
                            <th style={{ ...th, textAlign: "right" }}>HTTP</th>
                            <th style={{ ...th, textAlign: "right" }}>Response</th>
                          </tr>
                        </thead>
                        <tbody>
                          {report.pages.probes.map((probe) => (
                            <tr key={probe.path}>
                              <td style={{ ...td, whiteSpace: "nowrap" }}>{probe.path}</td>
                              <td style={td}>
                                <Flex align="center" gap={2}>
                                  <Dot status={probe.status} />
                                  <span>{STATUS_LABEL[probe.status]}</span>
                                </Flex>
                              </td>
                              <td
                                style={{
                                  ...td,
                                  textAlign: "right",
                                  fontVariantNumeric: "tabular-nums",
                                }}
                              >
                                {probe.httpStatus ?? "—"}
                              </td>
                              <td
                                style={{
                                  ...td,
                                  textAlign: "right",
                                  fontVariantNumeric: "tabular-nums",
                                }}
                              >
                                {latency(probe.latencyMs)}
                              </td>
                            </tr>
                          ))}
                        </tbody>
                      </table>
                    </Box>
                  )}
                </Card>
              </Stack>

              {/* Renewals & expiry */}
              <RenewalsSection renewals={report.renewals} />

              {/* Recent errors */}
              <Stack gap={3}>
                <Flex align="center" gap={2} wrap="wrap">
                  <Text weight="semibold" size={1}>
                    Recent errors
                  </Text>
                  <Badge tone={errorCounts.last24h > 0 ? "critical" : "default"} fontSize={0}>
                    {errorCounts.last24h} in 24h
                  </Badge>
                  <Badge tone={errorCounts.last7d > 0 ? "caution" : "default"} fontSize={0}>
                    {errorCounts.last7d} in 7d
                  </Badge>
                  <Badge fontSize={0} tone="default">
                    {errorCounts.unresolvedTotal} unresolved
                  </Badge>
                </Flex>
                <Card radius={3} border>
                  {!report.errors.configured ? (
                    <Box padding={4}>
                      <Text size={1} muted>
                        Sanity is not configured, so errors are not being recorded.
                      </Text>
                    </Box>
                  ) : errorItems.length === 0 ? (
                    <Box padding={4}>
                      <Flex align="center" gap={2}>
                        <Dot status="ok" />
                        <Text size={1} muted>
                          No unresolved errors.
                        </Text>
                      </Flex>
                    </Box>
                  ) : (
                    <Box overflow="auto" style={{ maxHeight: 520 }}>
                      <table
                        style={{
                          width: "100%",
                          borderCollapse: "collapse",
                          fontSize: 13,
                        }}
                      >
                        <thead>
                          <tr>
                            <th style={th}>Last seen</th>
                            <th style={th}>Request</th>
                            <th style={{ ...th, textAlign: "right" }}>Count</th>
                            <th style={th}>Message</th>
                            <th style={th}>
                              <span style={{ display: "none" }}>Actions</span>
                            </th>
                          </tr>
                        </thead>
                        <tbody>
                          {errorItems.map((item) => (
                            <ErrorRow key={item._id} error={item} onResolved={markResolved} />
                          ))}
                        </tbody>
                      </table>
                    </Box>
                  )}
                </Card>
                <Text size={0} muted>
                  Errors from the live site are captured by Next’s <code>onRequestError</code> hook and
                  stored in Sanity. Press “Resolve” once an error is fixed to remove it from this list; it
                  comes back as a new entry if it happens again.
                </Text>
              </Stack>
            </>
          ) : null}

          {!report && !error ? (
            <Flex align="center" justify="center" gap={3} style={{ minHeight: 320 }}>
              <Spinner muted />
              <Text size={1} muted>
                Running health checks…
              </Text>
            </Flex>
          ) : null}
        </Stack>
      </Container>
    </Box>
  );
}
````

### 11b. The server half

- `app/_lib/health-types.ts`: the shape of the health report.
- `app/_lib/health.ts`: builds the report. Set `PROBE_PATHS` to your own pages and `REQUIRED_ENV` to the variables you use.
- `app/_lib/domain-checks.ts`: looks up domain registration and SSL certificate expiry.
- `app/_lib/error-log.ts`: saves a server error as an `errorLog` document. Needs `SANITY_API_WRITE_TOKEN`.
- `instrumentation.ts` (project root): a Next.js hook that runs on every server error and calls `error-log.ts`.
- `app/api/admin/health/route.ts`: the endpoint the tab calls.

**`app/_lib/health-types.ts`**

````ts
/**
 * Shared types for the system-health dashboard. No server-only imports so the
 * Studio tool component can consume them.
 */

export type HealthStatus = "ok" | "degraded" | "down" | "not_configured";

export interface HealthCheck {
  id: string;
  label: string;
  status: HealthStatus;
  latencyMs: number | null;
  detail: string;
}

export interface PageProbe {
  path: string;
  url: string;
  status: HealthStatus;
  httpStatus: number | null;
  latencyMs: number | null;
}

export interface LoggedError {
  _id: string;
  message: string;
  route: string;
  method: string;
  routeType: string;
  environment: string;
  digest: string;
  stack: string;
  occurrences: number;
  firstSeenAt: string;
  lastSeenAt: string;
}

export type RenewalSource = "auto" | "manual";

export interface RenewalItem {
  id: string;
  source: RenewalSource;
  name: string;
  category: string;
  provider: string;
  url: string;
  expiresAt: string | null;
  daysLeft: number | null;
  autoRenews: boolean;
  cost: number | null;
  currency: string;
  status: HealthStatus;
  detail: string;
}

export interface HealthReport {
  generatedAt: string;
  overall: HealthStatus;
  checks: HealthCheck[];
  pages: {
    baseUrl: string | null;
    probes: PageProbe[];
  };
  errors: {
    configured: boolean;
    last24h: number;
    last7d: number;
    unresolvedTotal: number;
    items: LoggedError[];
  };
  renewals: {
    monitoredDomains: string[];
    items: RenewalItem[];
  };
}
````

**`app/_lib/health.ts`**

````ts
import "server-only";

import { createClient } from "next-sanity";

import { apiVersion, dataset, isSanityConfigured, projectId } from "@/sanity/env";
import { client as sanityClient } from "@/lib/sanity/client";
import { GoogleAnalyticsConfigurationError, pingGoogleAnalytics } from "@/app/_lib/google-analytics";
import {
  getDomainRegistrationExpiry,
  getTlsCertificateExpiry,
  resolveMonitoredDomains,
} from "@/app/_lib/domain-checks";
import type {
  HealthCheck,
  HealthReport,
  HealthStatus,
  LoggedError,
  PageProbe,
  RenewalItem,
} from "@/app/_lib/health-types";

const PROBE_TIMEOUT_MS = 8000;

/** Public pages swept on every health check (relative to NEXT_PUBLIC_SITE_URL). */
const PROBE_PATHS = ["/", "/blog", "/contact"];

const REQUIRED_ENV = [
  "NEXT_PUBLIC_SANITY_PROJECT_ID",
  "NEXT_PUBLIC_SANITY_DATASET",
  "SANITY_API_WRITE_TOKEN",
  "RESEND_API_KEY",
  "NEXT_PUBLIC_SITE_URL",
  "GOOGLE_ANALYTICS_PROPERTY_ID",
  "GOOGLE_ANALYTICS_CLIENT_EMAIL",
  "GOOGLE_ANALYTICS_PRIVATE_KEY",
];

const STATUS_RANK: Record<HealthStatus, number> = {
  ok: 0,
  not_configured: 1,
  degraded: 2,
  down: 3,
};

function worst(statuses: HealthStatus[]): HealthStatus {
  // not_configured on its own should not drag the whole system to "degraded".
  const meaningful = statuses.filter((s) => s !== "not_configured");
  const pool = meaningful.length ? meaningful : statuses;
  return pool.reduce<HealthStatus>(
    (acc, s) => (STATUS_RANK[s] > STATUS_RANK[acc] ? s : acc),
    "ok",
  );
}

async function timed<T>(fn: () => Promise<T>): Promise<{ ms: number; value?: T; error?: unknown }> {
  const start = Date.now();
  try {
    const value = await fn();
    return { ms: Date.now() - start, value };
  } catch (error) {
    return { ms: Date.now() - start, error };
  }
}

function errText(error: unknown): string {
  if (error instanceof Error) return error.message;
  return String(error);
}

async function checkSanityRead(): Promise<HealthCheck> {
  if (!isSanityConfigured) {
    return {
      id: "sanity-read",
      label: "Sanity CMS (read)",
      status: "not_configured",
      latencyMs: null,
      detail: "NEXT_PUBLIC_SANITY_PROJECT_ID / DATASET not set.",
    };
  }
  const { ms, error } = await timed(() =>
    sanityClient.fetch<number>('count(*[_type == "blogPost"])'),
  );
  return {
    id: "sanity-read",
    label: "Sanity CMS (read)",
    status: error ? "down" : ms > 2500 ? "degraded" : "ok",
    latencyMs: ms,
    detail: error ? errText(error) : `Content API responded in ${ms} ms.`,
  };
}

async function checkSanityWrite(): Promise<HealthCheck> {
  const token = process.env.SANITY_API_WRITE_TOKEN;
  if (!isSanityConfigured || !token) {
    return {
      id: "sanity-write",
      label: "Sanity write token",
      status: "not_configured",
      latencyMs: null,
      detail: "SANITY_API_WRITE_TOKEN not set — form submissions and error logging are disabled.",
    };
  }
  const client = createClient({ projectId, dataset, apiVersion, useCdn: false, token });
  const { ms, error } = await timed(() => client.users.getById("me"));
  return {
    id: "sanity-write",
    label: "Sanity write token",
    status: error ? "down" : "ok",
    latencyMs: ms,
    detail: error ? `Token rejected: ${errText(error)}` : "Token valid for this project.",
  };
}

async function checkResend(): Promise<HealthCheck> {
  const key = process.env.RESEND_API_KEY;
  if (!key) {
    return {
      id: "resend",
      label: "Resend email",
      status: "not_configured",
      latencyMs: null,
      detail: "RESEND_API_KEY not set — enquiry notification emails will not send.",
    };
  }
  const { ms, value, error } = await timed(() =>
    fetch("https://api.resend.com/domains", {
      headers: { Authorization: `Bearer ${key}` },
      cache: "no-store",
      signal: AbortSignal.timeout(PROBE_TIMEOUT_MS),
    }),
  );
  if (error) {
    return {
      id: "resend",
      label: "Resend email",
      status: "down",
      latencyMs: ms,
      detail: `Request failed: ${errText(error)}`,
    };
  }
  const res = value as Response;
  if (res.ok) {
    return {
      id: "resend",
      label: "Resend email",
      status: "ok",
      latencyMs: ms,
      detail: `API reachable, key accepted (${ms} ms).`,
    };
  }
  // A "Sending access" key — the right kind for this site — may send but not list
  // domains, so Resend answers this probe 401 restricted_api_key. That still proves
  // the key is real and active: an invalid or revoked key gets 400 `validation_error`.
  const body = (await res.json().catch(() => null)) as { name?: string; message?: string } | null;
  if (res.status === 401) {
    if (body?.name === "restricted_api_key") {
      return {
        id: "resend",
        label: "Resend email",
        status: "ok",
        latencyMs: ms,
        detail: `API reachable, sending-only key accepted (${ms} ms).`,
      };
    }
  }
  return {
    id: "resend",
    label: "Resend email",
    status: body?.name === "validation_error" || res.status === 401 || res.status === 403 ? "down" : "degraded",
    latencyMs: ms,
    detail: `Resend responded ${res.status}${body?.message ? `: ${body.message}` : ""}.`,
  };
}

async function checkGoogleAnalytics(): Promise<HealthCheck> {
  const { ms, error } = await timed(() => pingGoogleAnalytics());
  if (!error) {
    return {
      id: "google-analytics",
      label: "Google Analytics Data API",
      status: ms > 3000 ? "degraded" : "ok",
      latencyMs: ms,
      detail: `Service account authorised, property reachable (${ms} ms).`,
    };
  }
  if (error instanceof GoogleAnalyticsConfigurationError) {
    return {
      id: "google-analytics",
      label: "Google Analytics Data API",
      status: "not_configured",
      latencyMs: null,
      detail: error.message,
    };
  }
  return {
    id: "google-analytics",
    label: "Google Analytics Data API",
    status: "down",
    latencyMs: ms,
    detail: errText(error),
  };
}

function checkEnv(): HealthCheck {
  const missing = REQUIRED_ENV.filter((name) => !process.env[name]);
  const coreMissing = missing.some((name) => name.startsWith("NEXT_PUBLIC_SANITY"));
  return {
    id: "env",
    label: "Environment configuration",
    status: missing.length === 0 ? "ok" : coreMissing ? "down" : "degraded",
    latencyMs: null,
    detail:
      missing.length === 0
        ? `All ${REQUIRED_ENV.length} expected variables are set.`
        : `Missing: ${missing.join(", ")}.`,
  };
}

async function probePages(): Promise<{ baseUrl: string | null; probes: PageProbe[] }> {
  const base = process.env.NEXT_PUBLIC_SITE_URL?.replace(/\/$/, "") || null;
  if (!base) return { baseUrl: null, probes: [] };

  const probes = await Promise.all(
    PROBE_PATHS.map(async (path): Promise<PageProbe> => {
      const url = `${base}${path}`;
      const { ms, value, error } = await timed(() =>
        fetch(url, {
          method: "GET",
          redirect: "manual",
          cache: "no-store",
          headers: { "user-agent": "Site-HealthCheck" },
          signal: AbortSignal.timeout(PROBE_TIMEOUT_MS),
        }),
      );
      if (error) {
        return { path, url, status: "down", httpStatus: null, latencyMs: ms };
      }
      const res = value as Response;
      const code = res.status;
      const status: HealthStatus =
        code >= 500 ? "down" : code >= 400 ? "degraded" : "ok";
      return { path, url, status, httpStatus: code, latencyMs: ms };
    }),
  );
  return { baseUrl: base, probes };
}

async function collectErrors(): Promise<HealthReport["errors"]> {
  if (!isSanityConfigured) {
    return { configured: false, last24h: 0, last7d: 0, unresolvedTotal: 0, items: [] };
  }
  const now = Date.now();
  const since24 = new Date(now - 24 * 60 * 60 * 1000).toISOString();
  const since7 = new Date(now - 7 * 24 * 60 * 60 * 1000).toISOString();

  // Live, not the site's CDN client: the CDN keeps serving an error for a while after
  // someone presses Resolve, which left the overall status "degraded" with nothing to fix.
  const live = sanityClient.withConfig({ useCdn: false });

  try {
    const [last24h, last7d, unresolvedTotal, items] = await Promise.all([
      live.fetch<number>(
        'count(*[_type == "errorLog" && resolved != true && lastSeenAt > $s])',
        { s: since24 },
      ),
      live.fetch<number>(
        'count(*[_type == "errorLog" && resolved != true && lastSeenAt > $s])',
        { s: since7 },
      ),
      live.fetch<number>('count(*[_type == "errorLog" && resolved != true])'),
      live.fetch<LoggedError[]>(
        `*[_type == "errorLog" && resolved != true] | order(lastSeenAt desc)[0...50]{
          "_id": _id,
          "message": coalesce(message, ""),
          "route": coalesce(route, ""),
          "method": coalesce(method, ""),
          "routeType": coalesce(routeType, ""),
          "environment": coalesce(environment, ""),
          "digest": coalesce(digest, ""),
          "stack": coalesce(stack, ""),
          "occurrences": coalesce(occurrences, 1),
          "firstSeenAt": coalesce(firstSeenAt, _createdAt),
          "lastSeenAt": coalesce(lastSeenAt, _createdAt)
        }`,
      ),
    ]);
    return {
      configured: true,
      last24h: last24h ?? 0,
      last7d: last7d ?? 0,
      unresolvedTotal: unresolvedTotal ?? 0,
      items: items ?? [],
    };
  } catch {
    return { configured: true, last24h: 0, last7d: 0, unresolvedTotal: 0, items: [] };
  }
}

const DAY_MS = 24 * 60 * 60 * 1000;

function daysUntil(iso: string | null): number | null {
  if (!iso) return null;
  const ms = new Date(iso).getTime();
  if (Number.isNaN(ms)) return null;
  return Math.round((ms - Date.now()) / DAY_MS);
}

/** Expiry severity: expired/≤14d → down, ≤30d → degraded, else ok. Auto-renew softens to degraded. */
function renewalStatus(daysLeft: number | null, autoRenews: boolean): HealthStatus {
  if (daysLeft == null) return "not_configured";
  if (daysLeft <= 0) return autoRenews ? "degraded" : "down";
  if (daysLeft <= 14) return autoRenews ? "degraded" : "down";
  if (daysLeft <= 30) return "degraded";
  return "ok";
}

interface ManualRenewal {
  _id: string;
  name?: string;
  category?: string;
  provider?: string;
  url?: string;
  renewalDate?: string;
  autoRenews?: boolean;
  cost?: number;
  currency?: string;
  muted?: boolean;
}

async function collectRenewals(): Promise<HealthReport["renewals"]> {
  const domains = resolveMonitoredDomains();

  const autoChecks = domains.flatMap((domain) => [
    getDomainRegistrationExpiry(domain),
    getTlsCertificateExpiry(domain),
  ]);

  const manualQuery = isSanityConfigured
    ? sanityClient
        .fetch<ManualRenewal[]>(
          `*[_type == "renewal"] | order(renewalDate asc){
            _id, name, category, provider, url, renewalDate, autoRenews, cost, currency, muted
          }`,
        )
        .catch(() => [] as ManualRenewal[])
    : Promise.resolve([] as ManualRenewal[]);

  const [autoResults, manualResults] = await Promise.all([
    Promise.all(autoChecks),
    manualQuery,
  ]);

  const autoItems: RenewalItem[] = autoResults.map((result) => {
    const daysLeft = daysUntil(result.expiresAt);
    const isCert = result.kind === "tls-certificate";
    // Cert renewal is normally automated (Vercel/hosting), domain registration is not.
    const autoRenews = isCert;
    return {
      id: `auto:${result.kind}:${result.domain}`,
      source: "auto",
      name: isCert ? `TLS certificate · ${result.domain}` : `Domain · ${result.domain}`,
      category: isCert ? "tls" : "domain",
      provider: result.provider ?? "",
      url: "",
      expiresAt: result.expiresAt,
      daysLeft,
      autoRenews,
      cost: null,
      currency: "",
      status: renewalStatus(daysLeft, autoRenews),
      detail: result.detail,
    };
  });

  const manualItems: RenewalItem[] = manualResults.map((doc) => {
    const expiresAt = doc.renewalDate
      ? new Date(`${doc.renewalDate}T00:00:00Z`).toISOString()
      : null;
    const daysLeft = daysUntil(expiresAt);
    const autoRenews = doc.autoRenews ?? true;
    return {
      id: doc._id,
      source: "manual",
      name: doc.name ?? "Untitled",
      category: doc.category ?? "other",
      provider: doc.provider ?? "",
      url: doc.url ?? "",
      expiresAt,
      daysLeft,
      autoRenews,
      cost: typeof doc.cost === "number" ? doc.cost : null,
      currency: doc.currency ?? "",
      status: doc.muted ? "ok" : renewalStatus(daysLeft, autoRenews),
      detail: doc.muted ? "Warnings muted." : "",
    };
  });

  const items = [...autoItems, ...manualItems].sort((a, b) => {
    if (a.daysLeft == null) return 1;
    if (b.daysLeft == null) return -1;
    return a.daysLeft - b.daysLeft;
  });

  return { monitoredDomains: domains, items };
}

export async function getHealthReport(): Promise<HealthReport> {
  const [sanityRead, sanityWrite, resend, ga, pages, errors, renewals] = await Promise.all([
    checkSanityRead(),
    checkSanityWrite(),
    checkResend(),
    checkGoogleAnalytics(),
    probePages(),
    collectErrors(),
    collectRenewals(),
  ]);
  const env = checkEnv();

  const checks = [env, sanityRead, sanityWrite, resend, ga];
  const overall = worst([
    ...checks.map((c) => c.status),
    ...pages.probes.map((p) => p.status),
    ...renewals.items.map((r) => r.status),
    errors.unresolvedTotal > 0 ? "degraded" : "ok",
  ]);

  return {
    generatedAt: new Date().toISOString(),
    overall,
    checks,
    pages,
    errors,
    renewals,
  };
}
````

**`app/_lib/domain-checks.ts`**

````ts
import "server-only";

import tls from "node:tls";

const TIMEOUT_MS = 8000;

export interface DomainExpiry {
  domain: string;
  kind: "domain-registration" | "tls-certificate";
  expiresAt: string | null;
  detail: string;
  provider?: string;
}

/* ------------------------------------------------------------------ *
 * Domain registration expiry — RDAP (free, keyless, no dependency).  *
 * ------------------------------------------------------------------ */

interface RdapResponse {
  events?: { eventAction?: string; eventDate?: string }[];
  entities?: {
    roles?: string[];
    vcardArray?: unknown;
    handle?: string;
  }[];
}

function registrarName(data: RdapResponse): string | undefined {
  const registrar = data.entities?.find((entity) => entity.roles?.includes("registrar"));
  const vcard = registrar?.vcardArray;
  if (Array.isArray(vcard) && Array.isArray(vcard[1])) {
    const fn = (vcard[1] as unknown[][]).find(
      (entry) => Array.isArray(entry) && entry[0] === "fn",
    );
    if (fn && typeof fn[3] === "string") return fn[3];
  }
  return registrar?.handle;
}

export async function getDomainRegistrationExpiry(domain: string): Promise<DomainExpiry> {
  const base: DomainExpiry = {
    domain,
    kind: "domain-registration",
    expiresAt: null,
    detail: "",
  };
  try {
    const response = await fetch(`https://rdap.org/domain/${encodeURIComponent(domain)}`, {
      headers: { Accept: "application/rdap+json", "User-Agent": "Site-HealthCheck" },
      cache: "no-store",
      signal: AbortSignal.timeout(TIMEOUT_MS),
      redirect: "follow",
    });
    if (!response.ok) {
      return {
        ...base,
        detail:
          response.status === 404
            ? "No RDAP record (this TLD may not publish one). Track it manually instead."
            : `RDAP lookup returned ${response.status}.`,
      };
    }
    const data = (await response.json()) as RdapResponse;
    const expiration = data.events?.find(
      (event) => event.eventAction === "expiration" && event.eventDate,
    )?.eventDate;
    const provider = registrarName(data);
    if (!expiration) {
      return { ...base, provider, detail: "RDAP record found but no expiration date listed." };
    }
    return {
      ...base,
      provider,
      expiresAt: new Date(expiration).toISOString(),
      detail: provider ? `Registered via ${provider}.` : "From RDAP registry data.",
    };
  } catch (error) {
    return {
      ...base,
      detail: `RDAP lookup failed: ${error instanceof Error ? error.message : String(error)}`,
    };
  }
}

/* ------------------------------------------------------------------ *
 * TLS certificate expiry — direct socket, Node `tls` builtin.       *
 * ------------------------------------------------------------------ */

export function getTlsCertificateExpiry(host: string, port = 443): Promise<DomainExpiry> {
  const base: DomainExpiry = {
    domain: host,
    kind: "tls-certificate",
    expiresAt: null,
    detail: "",
  };

  return new Promise((resolve) => {
    let settled = false;
    const done = (result: DomainExpiry) => {
      if (settled) return;
      settled = true;
      try {
        socket.destroy();
      } catch {
        /* noop */
      }
      resolve(result);
    };

    const socket = tls.connect(
      { host, port, servername: host, timeout: TIMEOUT_MS, rejectUnauthorized: false },
      () => {
        const cert = socket.getPeerCertificate();
        if (!cert || !cert.valid_to) {
          done({ ...base, detail: "Handshake completed but no certificate was returned." });
          return;
        }
        const expiresAt = new Date(cert.valid_to).toISOString();
        const issuer =
          (cert.issuer && (cert.issuer.O || cert.issuer.CN)) || "unknown issuer";
        done({
          ...base,
          expiresAt,
          provider: typeof issuer === "string" ? issuer : undefined,
          detail: `Issued by ${issuer}.`,
        });
      },
    );

    socket.on("timeout", () => done({ ...base, detail: "TLS connection timed out." }));
    socket.on("error", (error) =>
      done({ ...base, detail: `TLS connection failed: ${error.message}` }),
    );
  });
}

/** Hostnames to check, from HEALTH_DOMAINS (comma-separated) or the site URL. */
export function resolveMonitoredDomains(): string[] {
  const explicit = (process.env.HEALTH_DOMAINS ?? "")
    .split(",")
    .map((value) => value.trim())
    .filter(Boolean);
  if (explicit.length) return dedupe(explicit.map(toHost));

  const siteUrl = process.env.NEXT_PUBLIC_SITE_URL;
  return siteUrl ? [toHost(siteUrl)] : [];
}

function toHost(value: string): string {
  try {
    return new URL(value.includes("://") ? value : `https://${value}`).hostname;
  } catch {
    return value.replace(/^https?:\/\//, "").split("/")[0];
  }
}

function dedupe(values: string[]): string[] {
  return [...new Set(values.filter(Boolean))];
}
````

**`app/_lib/error-log.ts`**

````ts
import "server-only";

import { createClient } from "next-sanity";

import { apiVersion, dataset, isSanityConfigured, projectId } from "@/sanity/env";

/** Shapes mirror the params Next passes to `onRequestError` in `instrumentation.ts`. */
interface RequestInfo {
  path?: string;
  method?: string;
  headers?: Record<string, string | string[] | undefined>;
}
interface ErrorContext {
  routePath?: string;
  routeType?: "render" | "route" | "action" | "proxy" | (string & {});
}

const MESSAGE_MAX = 1000;
const STACK_MAX = 8000;
/** Fold repeats of the same error+route into one document if seen within this window. */
const DEDUPE_WINDOW_MS = 24 * 60 * 60 * 1000;
/**
 * Not faults in the site. "The destination stream closed early" is Next reporting that
 * the browser hung up mid-response — the visitor clicked away, reloaded or closed the
 * tab — so nobody was left to see a broken page.
 */
const IGNORED_MESSAGES = ["The destination stream closed early"];

function writeClient() {
  return createClient({
    projectId,
    dataset,
    apiVersion,
    useCdn: false,
    token: process.env.SANITY_API_WRITE_TOKEN,
  });
}

function toMessage(error: unknown): string {
  if (error instanceof Error) return error.message || error.name || "Error";
  if (typeof error === "string") return error;
  try {
    return JSON.stringify(error);
  } catch {
    return String(error);
  }
}

function fingerprintOf(route: string, message: string): string {
  // Strip digits so "id 4821 not found" and "id 9033 not found" collapse together.
  const normalised = `${route}|${message}`.replace(/\d+/g, "#").slice(0, 240);
  let hash = 0;
  for (let i = 0; i < normalised.length; i += 1) {
    hash = (hash << 5) - hash + normalised.charCodeAt(i);
    hash |= 0;
  }
  return `el_${(hash >>> 0).toString(36)}`;
}

/**
 * Records a server error as an `errorLog` document in Sanity. Best-effort:
 * never throws, and is a no-op when the write token / Sanity project is absent.
 */
export async function reportError(
  error: unknown,
  request?: RequestInfo,
  context?: ErrorContext,
): Promise<void> {
  try {
    if (!isSanityConfigured || !process.env.SANITY_API_WRITE_TOKEN) return;
    // The error logger must not itself run on the Edge runtime bundle.
    if (process.env.NEXT_RUNTIME && process.env.NEXT_RUNTIME !== "nodejs") return;
    // Local `next dev` errors are half-finished edits on someone's machine, not
    // something a visitor hit — they already show in the terminal and the overlay.
    // Vercel previews and production both run NODE_ENV=production, so they still log.
    if (process.env.NODE_ENV === "development") return;

    const message = toMessage(error).slice(0, MESSAGE_MAX);
    if (IGNORED_MESSAGES.some((ignored) => message.includes(ignored))) return;
    const route = (request?.path || context?.routePath || "unknown").slice(0, 300);
    const method = (request?.method || "").slice(0, 10);
    const routeType = String(context?.routeType || "");
    const digest =
      typeof error === "object" && error !== null && "digest" in error
        ? String((error as { digest?: unknown }).digest ?? "").slice(0, 120)
        : "";
    const stack = error instanceof Error && error.stack ? error.stack.slice(0, STACK_MAX) : "";
    const environment = process.env.VERCEL_ENV || process.env.NODE_ENV || "unknown";
    const fingerprint = fingerprintOf(route, message);
    const now = new Date().toISOString();

    const client = writeClient();
    const since = new Date(Date.now() - DEDUPE_WINDOW_MS).toISOString();

    const existing = await client.fetch<{ _id: string } | null>(
      `*[_type == "errorLog" && fingerprint == $fingerprint && lastSeenAt > $since && resolved != true][0]{ _id }`,
      { fingerprint, since },
    );

    if (existing?._id) {
      await client
        .patch(existing._id)
        .setIfMissing({ occurrences: 1 })
        .inc({ occurrences: 1 })
        .set({ lastSeenAt: now, message, stack, digest })
        .commit({ visibility: "async" });
      return;
    }

    await client.create({
      _type: "errorLog",
      message,
      route,
      method,
      routeType,
      digest,
      environment,
      stack,
      fingerprint,
      occurrences: 1,
      firstSeenAt: now,
      lastSeenAt: now,
      resolved: false,
    });
  } catch (loggingError) {
    // Swallow — a failing logger must never mask or amplify the original error.
    console.error("[error-log] failed to record server error", loggingError);
  }
}
````

**`instrumentation.ts`**

````ts
import type { Instrumentation } from "next";

/**
 * Framework-level hook: fires whenever the Next server captures an error in a
 * Route Handler, Server Component render, Server Action, or proxy. We forward it
 * to `reportError`, which records it as an `errorLog` document in Sanity for the
 * Studio "Health" tool. Node runtime only — the Edge bundle skips the import.
 */
export const onRequestError: Instrumentation.onRequestError = async (error, request, context) => {
  if (process.env.NEXT_RUNTIME !== "nodejs") return;
  const { reportError } = await import("@/app/_lib/error-log");
  await reportError(error, request, context);
};
````

**`app/api/admin/health/route.ts`**

````ts
import { NextResponse, type NextRequest } from "next/server";

import { getHealthReport } from "@/app/_lib/health";
import { isAuthorisedStudioUser } from "@/app/_lib/studio-auth";

export const runtime = "nodejs";
export const dynamic = "force-dynamic";
export const maxDuration = 30;

export async function GET(request: NextRequest) {
  if (!(await isAuthorisedStudioUser(request))) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  try {
    const report = await getHealthReport();
    return NextResponse.json(report, {
      headers: { "Cache-Control": "private, no-store" },
    });
  } catch (error) {
    console.error("[health] report failed", error);
    return NextResponse.json({ error: "Health check could not be completed." }, { status: 502 });
  }
}
````

---

## 12. `sanity/tools/dashboard`

**Location:** `sanity/tools/dashboard/`. This is the Studio's **Dashboard** tab, the first one you see at `/admin`. It shows:
- live counts of enquiries and content;
- a summary of Analytics and Health, using the same endpoints as those two tabs, so it needs sections 9 and 11 in place.

Add a count for each content type you add.

```bash
mkdir -p sanity/tools/dashboard
```

**`sanity/tools/dashboard/index.ts`**

````ts
import { DashboardIcon } from "@sanity/icons/Dashboard";
import { definePlugin } from "sanity";

import Dashboard from "./Dashboard";

/**
 * Adds a "Dashboard" tab to the Studio (`/admin`): headline numbers for the
 * manager — enquiries, visitors, content, site status — and links into the
 * Content, Analytics and Health tabs. Listed first in sanity.config.ts, so it is
 * what /admin opens on.
 */
export const dashboardTool = definePlugin({
  name: "site-dashboard",
  tools: [
    {
      name: "dashboard",
      title: "Dashboard",
      icon: DashboardIcon,
      component: Dashboard,
    },
  ],
});
````

**`sanity/tools/dashboard/Dashboard.tsx`**

````tsx
import { useCallback, useEffect, useState, type MouseEvent, type ReactNode } from "react";
import { useClient, useCurrentUser } from "sanity";
import { useRouter } from "sanity/router";
import { Badge, Box, Button, Card, Container, Flex, Grid, Heading, Spinner, Stack, Text } from "@sanity/ui";
import { ActivityIcon } from "@sanity/icons/Activity";
import { ArrowRightIcon } from "@sanity/icons/ArrowRight";
import { ChartUpwardIcon } from "@sanity/icons/ChartUpward";
import { StackIcon } from "@sanity/icons/Stack";
import { SyncIcon } from "@sanity/icons/Sync";

import { apiVersion } from "@/sanity/env";
import { useStudioToken } from "@/sanity/tools/useStudioToken";
import type { GoogleAnalyticsDashboardData } from "@/app/_lib/google-analytics-types";
import type { HealthReport, HealthStatus } from "@/app/_lib/health-types";

const BASE = "/admin";
const DAY = 24 * 60 * 60 * 1000;

/* -------------------------------------------------------------------------- */
/*  Data                                                                       */
/* -------------------------------------------------------------------------- */

interface Counts {
  enquiriesNew: number;
  enquiries7d: number;
  enquiries30d: number;
  enquiriesTotal: number;
  blogPosts: number;
  latest: { _id: string; fullName?: string; email?: string; service?: string; status?: string; submittedAt?: string }[];
}

// Published documents only: a draft is not on the site yet.
const PUBLISHED = `!(_id in path("drafts.**"))`;

const COUNTS_QUERY = `{
  "enquiriesNew": count(*[_type == "enquiry" && status == "new" && ${PUBLISHED}]),
  "enquiries7d": count(*[_type == "enquiry" && submittedAt > $since7 && ${PUBLISHED}]),
  "enquiries30d": count(*[_type == "enquiry" && submittedAt > $since30 && ${PUBLISHED}]),
  "enquiriesTotal": count(*[_type == "enquiry" && ${PUBLISHED}]),
  "blogPosts": count(*[_type == "blogPost" && ${PUBLISHED}]),
  "latest": *[_type == "enquiry" && ${PUBLISHED}] | order(submittedAt desc)[0...5]{
    _id, fullName, email, service, status, submittedAt
  }
}`;

function isoDate(daysAgo: number) {
  const date = new Date();
  date.setUTCDate(date.getUTCDate() - daysAgo);
  return date.toISOString().slice(0, 10);
}

type Remote<T> = { state: "loading" } | { state: "ready"; data: T } | { state: "error"; message: string };

/** GET one of our /api/admin routes as the logged-in Studio user. */
async function getAdmin<T>(path: string, token: string | null): Promise<T> {
  if (!token) throw new Error("Could not read your Studio session.");
  const response = await fetch(path, { cache: "no-store", headers: { Authorization: `Bearer ${token}` } });
  const payload = await response.json();
  if (!response.ok) throw new Error(payload.error ?? "Could not load.");
  return payload as T;
}

/* -------------------------------------------------------------------------- */
/*  Pieces                                                                     */
/* -------------------------------------------------------------------------- */

/** In-Studio navigation: no full page reload when moving between tabs. */
function useStudioLink() {
  const router = useRouter();
  return (path: string) => ({
    href: path,
    onClick: (event: MouseEvent) => {
      if (event.metaKey || event.ctrlKey || event.shiftKey || event.button !== 0) return;
      event.preventDefault();
      router.navigateUrl({ path });
    },
  });
}

/** `value` null = still loading. */
function Stat({ label, value, hint, tone }: { label: string; value: string | null; hint?: string; tone?: "caution" }) {
  return (
    <Card padding={4} radius={3} border tone={tone ?? "default"}>
      <Stack gap={3}>
        <Text size={1} muted>
          {label}
        </Text>
        {/* The spinner sits beside the Text, not in it: Sanity UI trims Text's line
            height, so a block child overflows it and lands on the label above. The
            hidden "0" keeps the card the same height loading and loaded. */}
        <Box style={{ position: "relative" }}>
          <Text size={4} weight="semibold" style={{ fontVariantNumeric: "tabular-nums" }}>
            {value ?? <span style={{ visibility: "hidden" }}>0</span>}
          </Text>
          {value === null ? (
            <Flex align="center" style={{ position: "absolute", inset: 0 }}>
              <Spinner muted />
            </Flex>
          ) : null}
        </Box>
        {hint ? (
          <Text size={1} muted>
            {hint}
          </Text>
        ) : null}
      </Stack>
    </Card>
  );
}

function Section({ title, action, children }: { title: string; action?: ReactNode; children: ReactNode }) {
  return (
    <Stack gap={3}>
      <Flex align="center" justify="space-between">
        <Text size={1} weight="semibold" muted style={{ textTransform: "uppercase", letterSpacing: "0.06em" }}>
          {title}
        </Text>
        {action}
      </Flex>
      {children}
    </Stack>
  );
}

const HEALTH_LABEL: Record<HealthStatus, string> = {
  ok: "All systems operational",
  degraded: "Some services degraded",
  down: "Outage detected",
  not_configured: "Setup incomplete",
};
const HEALTH_TONE: Record<HealthStatus, "positive" | "caution" | "critical" | "default"> = {
  ok: "positive",
  degraded: "caution",
  down: "critical",
  not_configured: "default",
};

/**
 * What is pulling the overall status down, in the same order the Health tab lists it —
 * so "degraded" always comes with a reason. Mirrors `overall` in app/_lib/health.ts;
 * "not configured" items are left out because they never lower the overall status.
 */
function healthIssues(report: HealthReport): string[] {
  const bad = (status: HealthStatus) => status === "degraded" || status === "down";
  return [
    ...report.checks.filter((c) => bad(c.status)).map((c) => `${c.label}: ${c.detail}`),
    ...report.pages.probes
      .filter((p) => bad(p.status))
      .map((p) => `Page ${p.path} answered ${p.httpStatus ?? "nothing"}`),
    ...report.renewals.items
      .filter((r) => bad(r.status))
      .map((r) =>
        r.daysLeft == null ? `${r.name} needs renewing` : r.daysLeft <= 0 ? `${r.name} has expired` : `${r.name} expires in ${r.daysLeft} days`,
      ),
    ...(report.errors.unresolvedTotal > 0
      ? [`${report.errors.unresolvedTotal} unresolved server error${report.errors.unresolvedTotal === 1 ? "" : "s"}`]
      : []),
  ];
}

function timeAgo(iso?: string) {
  if (!iso) return "—";
  const mins = Math.round((Date.now() - new Date(iso).getTime()) / 60000);
  if (mins < 1) return "just now";
  if (mins < 60) return `${mins}m ago`;
  const hours = Math.round(mins / 60);
  if (hours < 24) return `${hours}h ago`;
  return `${Math.round(hours / 24)}d ago`;
}

const n = (value: number) => value.toLocaleString("en-IN");

/* -------------------------------------------------------------------------- */
/*  Dashboard                                                                  */
/* -------------------------------------------------------------------------- */

export default function Dashboard() {
  const client = useClient({ apiVersion });
  const user = useCurrentUser();
  const token = useStudioToken();
  const link = useStudioLink();

  const [counts, setCounts] = useState<Remote<Counts>>({ state: "loading" });
  const [analytics, setAnalytics] = useState<Remote<GoogleAnalyticsDashboardData>>({ state: "loading" });
  const [health, setHealth] = useState<Remote<HealthReport>>({ state: "loading" });

  const load = useCallback(() => {
    setCounts({ state: "loading" });
    setAnalytics({ state: "loading" });
    setHealth({ state: "loading" });

    const now = Date.now();
    client
      .fetch<Counts>(COUNTS_QUERY, {
        since7: new Date(now - 7 * DAY).toISOString(),
        since30: new Date(now - 30 * DAY).toISOString(),
      })
      .then((data) => setCounts({ state: "ready", data }))
      .catch((err: Error) => setCounts({ state: "error", message: err.message }));

    // Each source loads on its own, so a slow health probe never holds up the numbers.
    const range = new URLSearchParams({ startDate: isoDate(6), endDate: isoDate(0) });
    getAdmin<GoogleAnalyticsDashboardData>(`/api/admin/google-analytics?${range}`, token)
      .then((data) => setAnalytics({ state: "ready", data }))
      .catch((err: Error) => setAnalytics({ state: "error", message: err.message }));

    getAdmin<HealthReport>("/api/admin/health", token)
      .then((data) => setHealth({ state: "ready", data }))
      .catch((err: Error) => setHealth({ state: "error", message: err.message }));
  }, [client, token]);

  useEffect(() => {
    // Deferred a tick so the first paint is the layout, not a blocked render.
    const id = window.setTimeout(load, 0);
    return () => window.clearTimeout(id);
  }, [load]);

  const c = counts.state === "ready" ? counts.data : null;
  const ga = analytics.state === "ready" ? analytics.data : null;

  return (
    <Box padding={[4, 5, 6]} style={{ height: "100%", overflow: "auto" }}>
      <Container width={3}>
        <Stack gap={6}>
          <Flex align="flex-end" justify="space-between" gap={3} wrap="wrap">
            <Stack gap={3}>
              <Heading size={3}>Welcome{user?.name ? `, ${user.name.split(" ")[0]}` : ""}</Heading>
              <Text muted>How the site is doing at a glance.</Text>
            </Stack>
            <Button icon={SyncIcon} mode="ghost" text="Refresh" onClick={load} />
          </Flex>

          {/* Enquiries */}
          <Section
            title="Enquiries"
            action={
              <Button
                as="a"
                {...link(`${BASE}/structure/enquiries`)}
                mode="bleed"
                fontSize={1}
                iconRight={ArrowRightIcon}
                text="Open inbox"
              />
            }
          >
            {counts.state === "error" ? (
              <Card padding={4} radius={3} tone="critical" border>
                <Text size={1}>Could not load the numbers: {counts.message}</Text>
              </Card>
            ) : (
              <Grid gridTemplateColumns={[2, 2, 4]} gap={3}>
                <Stat
                  label="New — not yet handled"
                  value={c ? n(c.enquiriesNew) : null}
                  tone={c && c.enquiriesNew > 0 ? "caution" : undefined}
                />
                <Stat label="Last 7 days" value={c ? n(c.enquiries7d) : null} />
                <Stat label="Last 30 days" value={c ? n(c.enquiries30d) : null} />
                <Stat label="All time" value={c ? n(c.enquiriesTotal) : null} />
              </Grid>
            )}

            {c && c.latest.length > 0 ? (
              <Card radius={3} border>
                <Stack>
                  {c.latest.map((e, i) => (
                    <Card
                      key={e._id}
                      as="a"
                      {...link(`${BASE}/intent/edit/id=${e._id};type=enquiry`)}
                      padding={3}
                      style={{ borderTop: i ? "1px solid var(--card-border-color)" : undefined }}
                    >
                      <Flex align="center" gap={3}>
                        <Box flex={1}>
                          <Text size={1} weight="medium" textOverflow="ellipsis">
                            {e.fullName || e.email || "Unnamed"}
                          </Text>
                        </Box>
                        <Text size={1} muted>
                          {e.service}
                        </Text>
                        {e.status === "new" ? (
                          <Badge tone="caution" fontSize={0}>
                            New
                          </Badge>
                        ) : null}
                        <Box style={{ width: 64, textAlign: "right" }}>
                          <Text size={1} muted>
                            {timeAgo(e.submittedAt)}
                          </Text>
                        </Box>
                      </Flex>
                    </Card>
                  ))}
                </Stack>
              </Card>
            ) : null}
          </Section>

          {/* Visitors */}
          <Section
            title="Website visitors · last 7 days"
            action={
              <Button
                as="a"
                {...link(`${BASE}/analytics`)}
                mode="bleed"
                fontSize={1}
                iconRight={ArrowRightIcon}
                text="Full analytics"
              />
            }
          >
            {analytics.state === "error" ? (
              <Card padding={4} radius={3} border>
                <Text size={1} muted>
                  Analytics unavailable: {analytics.message}
                </Text>
              </Card>
            ) : (
              <Grid gridTemplateColumns={[2, 2, 4]} gap={3}>
                <Stat label="On the site right now" value={ga ? n(ga.realtime.activeUsers) : null} />
                <Stat label="Visitors" value={ga ? n(ga.overview.totalUsers) : null} />
                <Stat label="New visitors" value={ga ? n(ga.overview.newUsers) : null} />
                <Stat label="Page views" value={ga ? n(ga.overview.screenPageViews) : null} />
              </Grid>
            )}
          </Section>

          {/* Content + status */}
          <Grid gridTemplateColumns={[1, 1, 2]} gap={5}>
            <Section title="On the website">
              <Grid gridTemplateColumns={3} gap={3}>
                <Stat label="Blog posts" value={c ? n(c.blogPosts) : null} />
              </Grid>
            </Section>

            <Section title="Site status">
              <Card padding={4} radius={3} border tone={health.state === "ready" ? HEALTH_TONE[health.data.overall] : "default"}>
                <Stack gap={3}>
                  {health.state === "loading" ? (
                    <Flex align="center" gap={2}>
                      <Spinner muted />
                      <Text size={1} muted>
                        Checking the site…
                      </Text>
                    </Flex>
                  ) : health.state === "error" ? (
                    <Text size={1} muted>
                      Status unavailable: {health.message}
                    </Text>
                  ) : (
                    <>
                      <Text size={2} weight="semibold">
                        {HEALTH_LABEL[health.data.overall]}
                      </Text>
                      {healthIssues(health.data).map((issue) => (
                        <Text key={issue} size={1}>
                          • {issue}
                        </Text>
                      ))}
                    </>
                  )}
                </Stack>
              </Card>
            </Section>
          </Grid>

          {/* Other tabs */}
          <Section title="Go to">
            <Grid gridTemplateColumns={[1, 1, 3]} gap={3}>
              {[
                { path: "structure", title: "Content", icon: StackIcon, text: "Edit blog posts and settings, and work through enquiries." },
                { path: "analytics", title: "Analytics", icon: ChartUpwardIcon, text: "Visitors, top pages, where traffic comes from, by date range." },
                { path: "health", title: "Health", icon: ActivityIcon, text: "Live service checks, server errors and domain renewals." },
              ].map(({ path, title, icon: Icon, text }) => (
                <Card key={path} as="a" {...link(`${BASE}/${path}`)} padding={4} radius={3} border>
                  <Stack gap={3}>
                    <Flex align="center" gap={2}>
                      <Text size={2}>
                        <Icon />
                      </Text>
                      <Text size={2} weight="semibold">
                        {title}
                      </Text>
                    </Flex>
                    <Text size={1} muted>
                      {text}
                    </Text>
                  </Stack>
                </Card>
              ))}
            </Grid>
          </Section>
        </Stack>
      </Container>
    </Box>
  );
}
````

---

## 13. Run and check

```bash
npx tsc --noEmit
```

```bash
npm run dev
```

Open http://localhost:3000/admin and sign in. You should see the **Dashboard**, **Structure**, **Analytics** and **Health** tabs.

Then check the production build, which is what Vercel runs:

```bash
npm run build
```

When it builds without errors, commit and push. Vercel deploys the site and `/admin` together.

```bash
git add -A
```

```bash
git commit -m "feat: Sanity Studio at /admin"
```

```bash
git push
```

**Common errors:**
- **"Tool not found: admin"** means `basePath: "/admin"` is missing from `sanity.config.ts`.
- **Overlapping text in a tab** means the code uses an old `@sanity/ui` 3 prop. Version 4 renamed `space` to `gap`, renamed `Grid columns` to `gridTemplateColumns`, and removed `Badge mode`.
- **`npm ENOTEMPTY`** comes from an interrupted install. Stop the dev server, run `rm -rf node_modules/.<name>-*`, and install again.
- **The Analytics tab says "not configured"** when the `GOOGLE_ANALYTICS_*` variables are missing. See section 10.
- **`npx sanity …` fails with `MODULE_NOT_FOUND`** when `sanity.cli.ts` imports with `@/…`. Use the version in section 3.
