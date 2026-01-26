# Project Discussion Pack – Full Stack AI Short Video Ads Generator

## 1. PROJECT OVERVIEW
- **What**: A full-stack web app that turns user-uploaded product/person images into AI-generated photo composites and short product promo videos.
- **Purpose**: Let marketers and founders create studio-quality vertical ads without a production crew.
- **Problem solved**: Reduces cost/time of photo/video shoots by automating image compositing and video generation with GenAI, plus simple sharing/publishing.

## 2. TECH STACK & TOOLS
- **Frontend**: React + Vite (fast dev/build), TypeScript (type safety), React Router (multi-page flow), Tailwind v4 (utility styling), Framer Motion (micro-interactions), Clerk React (auth UI), React Hot Toast (UX feedback), Lenis (smooth scroll).
- **Backend**: Express 5 (HTTP APIs), TypeScript + tsx (modern TS runtime), Clerk Express (authn), Multer (file uploads), Axios (server-side fetch), Sentry (error tracing), CORS middleware.
- **AI/Media**: Google GenAI (`gemini-3-pro-image-preview` for images, `veo-3.1-generate-preview` for video), Cloudinary (image/video storage + CDN).
- **Database & storage**: PostgreSQL via Prisma ORM; Prisma PG adapter; Cloudinary buckets for generated media.
- **Why these choices**: Vite/React/TS maximize iteration speed; Clerk offloads auth; Prisma gives typed DB and migrations; Cloudinary simplifies media CDN; Google GenAI chosen for multimodal image+video generation; Express is lightweight and easy to extend; Sentry for observability. Trade-offs: GenAI latency vs local control; Cloudinary cost vs DIY storage; Prisma convenience vs raw SQL control.

## 3. ARCHITECTURE & DESIGN
- **High-level flow**: React SPA -> Axios calls -> Express API -> Prisma -> Postgres. Media files upload to Cloudinary; AI calls to Google GenAI; generated media URLs persisted in DB.
- **Diagram – High level**:
```
Browser (React SPA)
  |
  | HTTPS (Axios)
  v
Express API (TypeScript, Clerk middleware)
  |  \-- Multer (uploads) --> Temp FS
  |  \-- Sentry (observability)
  v
Prisma ORM -----------------------> PostgreSQL (users, projects)
  |
  |-- Cloudinary (media CDN: uploads + URLs)
  |-- Google GenAI (image/video generation)
```
- **Request/data flow**:
  - User signs in via Clerk; JWT propagated in `Authorization` headers and validated by Clerk middleware + `protect` guard.
  - Image generation: upload 2 images via `/api/project/create` (Multer saves temp files) -> Cloudinary upload -> GenAI image generation -> Cloudinary store result -> Prisma updates `Project` with URLs and flags.
  - Video generation: `/api/project/video` uses stored generated image -> GenAI video op -> poll until done -> download temp MP4 -> Cloudinary upload -> DB update.
  - Reads: `/api/user/projects`, `/api/user/projects/:id`, `/api/project/published` serve project listings.
- **Diagram – Request lifecycle (image -> video)**:
```
1) Client uploads 2 images + prompt
  -> Express /api/project/create (auth via Clerk)
  -> Multer saves temp files
  -> Cloudinary upload (source images)
  -> Google GenAI (Gemini) generates composite
  -> Cloudinary upload (generated PNG)
  -> Prisma saves Project {uploadedImages, generatedImage, isGenerating:false}

2) Client requests video
  -> Express /api/project/video (auth)
  -> Load generatedImage from Cloudinary
  -> Google GenAI (VEO) long-running op
  -> Poll until done; download temp MP4
  -> Cloudinary upload (final MP4)
  -> Prisma updates Project {generatedVideo, isGenerating:false}

3) Client fetches projects/community
  -> Express /api/user/projects or /api/project/published
  -> Prisma read -> send JSON; media served via Cloudinary CDN
```
- **Folder structure (frontend + backend)**: see `client/` (React app; components, pages, configs) and `server/` (Express app; controllers, routes, configs, prisma schema/migrations, middleware).
- **Database schema (PostgreSQL via Prisma)**:
  - `User`: `id`, `email`, `name`, `image`, `credits` (default 20), timestamps.
  - `Project`: `id` (uuid), `name`, `userId` FK, `productName`, `productDescription`, `userPrompt`, `aspectRatio`, `targetLength`, `uploadedImages` (string[]), `generatedImage`, `generatedVideo`, `isGenerating`, `isPublished`, `error`, timestamps.
  - Example `Project` row: `{ id: "uuid", userId: "clerk_user_id", productName: "Coffee Maker", userPrompt: "studio lighting, lifestyle", uploadedImages: ["https://...person.jpg", "https://...product.jpg"], generatedImage: "https://cloudinary/...png", generatedVideo: "https://cloudinary/...mp4", isGenerating: false, isPublished: true }`.

## 4. API & SYSTEM DESIGN
- **Key REST APIs**:
  - `POST /api/project/create` (auth, multipart `images[]`, body: `name`, `aspectRatio`, `userPrompt`, `productName`, `productDescription`, `targetLength`) → create project, generate composite image, deduct 5 credits.
  - `POST /api/project/video` (auth, body: `projectId`) → generate short video from project image, deduct 10 credits.
  - `GET /api/project/published` → list public projects.
  - `DELETE /api/project/:projectId` (auth) → delete own project.
  - `GET /api/user/credits` (auth) → return credits (creates user with defaults if missing).
  - `GET /api/user/projects` (auth) → list own projects.
  - `GET /api/user/projects/:projectId` (auth) → fetch single project.
  - `GET /api/user/publish/:projectId` (auth) → toggle publish flag (requires generated media).
- **Async/long-running**: Video generation uses Google GenAI operations polling before download/upload; image/video generation marked via `isGenerating` flag to block duplicate runs.
- **AuthZ**: `protect` middleware checks Clerk-authenticated `userId`; controllers scope queries by `userId` to enforce ownership.

## 5. KEY FEATURES & FUNCTIONALITY
- Guided creation flow: upload two images (person + product), enter prompt/ratio/length, start generation.
- Credit system: image (-5) and video (-10) generation decrements credits; refunds on failure.
- AI image generation: merges person+product with studio-light prompt; stores result in Cloudinary.
- AI video generation: uses generated image as seed to create short ad video; streams result to Cloudinary.
- Project management: list own projects, view detail, delete, toggle publish, browse published community feed.
- UX touches: smooth scrolling, animated hero/sections, toasts for success/error, Clerk-hosted auth UI.
- Error handling: Sentry capture on backend, error persisted in project row when generation fails.

## 6. MY ROLE & CONTRIBUTIONS
- **Frontend**: Built React SPA (routing, pages, components), integrated Clerk Provider, toast UX, smooth scroll, pricing/FAQ/CTA sections, upload & result flows, community feed.
- **Backend**: Designed Express APIs, integrated Clerk middleware, built Multer upload handling, Cloudinary upload pipeline, Google GenAI image/video calls with polling, Sentry error tracing.
- **Database**: Modeled users/projects with Prisma, migrations, credit accounting, publish flags, generation state/error fields.
- **Deployment/env**: .env driven (Clerk keys, DB URL, Cloudinary, Google API); start scripts (`tsx server.ts`, Vite dev/build). CORS configured; raw body route for Clerk webhooks.
- **Ownership**: Sole developer; made tech choices, schema design, error handling, UX decisions.

## 7. CHALLENGES & SOLUTIONS
- **Reliable media generation**: GenAI ops can be slow/fail → added `isGenerating` guard, polling loop, and persisted `error` field; refunds credits on failure; Cloudinary used for stable URLs.
- **Auth across tiers**: Needed consistent identity in API and DB → standardized on Clerk `userId`, added `protect` middleware, and scoped Prisma queries by `userId`.
- **Large file handling**: Image/video temp files → Multer disk storage with minimal config, immediate Cloudinary upload, and filesystem cleanup for videos after upload.
- **Credit accuracy**: Avoid double charges → deduct before work, refund on caught errors, gate duplicate requests when `isGenerating` true.

## 8. CORE TECHNICAL CONCEPTS
- **Clerk JWT auth**: Clerk middleware populates `req.auth`; `protect` enforces presence; IDs persist to DB for ownership checks.
- **Prisma with Postgres**: Typed client, migrations, cascading delete on projects; arrays for uploaded images; timestamps and defaults.
- **GenAI image/video ops**: Uses multimodal prompts with base64 inline images; video via long-running operations + polling; temperature/topP tuned for quality; safety settings relaxed for ad scenarios.
- **Cloudinary CDN**: Uploads for generated PNG/MP4; returns HTTPS URLs stored in DB; reduces backend bandwidth.
- **Error observability**: Sentry capture in controllers; explicit error messages returned; `error` column for UI surfacing.

## 9. PERFORMANCE, SECURITY & SCALABILITY
- **Security**: Clerk-managed auth, route-level protection, ownership scoping, CORS allowlist (configurable), environment-based secrets, publish toggle prevents exposing non-generated assets.
- **Performance**: Vite+React for fast load; minimized server work by offloading media CDN; avoids re-generation when media exists; Prisma queries indexed by PK/FK; async uploads in parallel for images.
- **Failure handling**: Generation errors update project state and refund credits; video temp files cleaned; Sentry alerts; API returns clear messages; `isGenerating` prevents race conditions.
- **Scalability**: Stateless API; Cloudinary and Google GenAI scale externally; Postgres can be hosted (e.g., managed DB); routers modular for feature growth.

## 10. POSSIBLE IMPROVEMENTS & FUTURE ENHANCEMENTS
- **Short term**: Add client-side progress indicators for generation; retry with exponential backoff; improve validation (file size/type); configurable credit pricing.
- **Medium term**: Webhook/async callbacks from GenAI to avoid polling; queue jobs (BullMQ) for heavy workloads; role-based admin dashboard; rate limiting.
- **Long term**: Multi-region deployment; caching published feeds; pre-signed upload direct to Cloudinary; add template library and captions; analytics on conversion.

## 11. LIKELY INTERVIEW QUESTIONS & STRONG ANSWERS
- **Architecture**: How do frontend, backend, and external services interact? → SPA calls Express; Clerk auth propagates; media to Cloudinary; AI via Google GenAI; DB via Prisma/Postgres; state stored in `Project` rows.
- **Backend**: How do you prevent double generation? → `isGenerating` flag, existence checks (`generatedImage`/`generatedVideo`), and credit refund on failure.
- **Database**: Why arrays for `uploadedImages`? → Small, bounded (2 items) and simplifies ordering of inputs; alternative is join table if we supported many assets.
- **Security**: How is auth enforced? → Clerk middleware populates `req.auth`; `protect` requires `userId`; Prisma queries filter by `userId`; publish endpoint gated on generated media.
- **Scalability**: How to handle higher volume? → Move to job queue + worker for generation, webhook-based completion, horizontal scale of stateless API, connection pooling for Postgres, CDN caching for published feed.

## 12. QUICK REVISION SUMMARY
- **Key files**: Frontend shell in [client/src/App.tsx](client/src/App.tsx); auth setup in [client/src/main.tsx](client/src/main.tsx); backend entry [server/server.ts](server/server.ts); project controller [server/controllers/projectController.ts](server/controllers/projectController.ts); user controller [server/controllers/userController.ts](server/controllers/userController.ts); schema [server/prisma/schema.prisma](server/prisma/schema.prisma); auth guard [server/middlewares/auth.ts](server/middlewares/auth.ts).
- **Key technologies**: React+Vite+TS, Clerk auth, Tailwind, Express 5, Prisma/Postgres, Google GenAI (Gemini + VEO), Cloudinary CDN, Sentry.
- **Talking points**: AI image/video pipeline, credit accounting with refunds, ownership enforcement via Clerk IDs, `isGenerating` race prevention, Cloudinary for media delivery, Prisma schema design with publish flag and error field.
