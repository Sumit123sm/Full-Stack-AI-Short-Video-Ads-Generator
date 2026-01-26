# Deploying the Backend to Render

Concise steps to get the Express + Prisma backend running on Render with PostgreSQL and Cloudinary.

## 1) Prereqs
- Render account
- Cloudinary account (for media storage)
- Google GenAI API key
- Clerk Secret Key (server-side auth)
- Sentry DSN (optional, for error tracing)

## 2) Create services on Render
1. **PostgreSQL**: Create a Managed PostgreSQL instance. Copy its `External Database URL` (use as `DATABASE_URL`).
2. **Web Service**: New > Web Service > Connect your repo > pick the `server/` directory as the root.

## 3) Render service settings
- **Environment**: Node
- **Build Command**: `npm install && npm run build`
- **Start Command**: `npm start`
- **Root Directory**: `server`
- **Instance Type**: Start with the free or lowest tier; bump if GenAI polling load grows.
- **Node version**: (optional) set an env var `NODE_VERSION=20` or match your local.

## 4) Required environment variables
Set these in Render > Web Service > Environment:
- `PORT` = `10000` (Render sets `PORT`; you can also omit and let app read `PORT`.)
- `DATABASE_URL` = `<render postgres external URL>`
- `GOOGLE_CLOUD_API_KEY` = `<your Google GenAI key>`
- `CLERK_SECRET_KEY` = `<your Clerk secret key>`
- **Cloudinary** (either single URL or components):
  - Option A: `CLOUDINARY_URL=cloudinary://<api_key>:<api_secret>@<cloud_name>`
  - Option B: `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`
- Optional:
  - `SENTRY_DSN` for error tracing

## 5) Prisma configuration notes
- `DATABASE_URL` is read by `@prisma/adapter-pg` in `configs/prisma.ts`.
- Render builds on each deploy, so `npm run build` will generate the Prisma client under `server/generated/prisma`.
- If you change the schema, run `npx prisma migrate deploy` once. You can add a Render deploy hook or run it from a temporary shell: `npx prisma migrate deploy` with `DATABASE_URL` set.

## 6) Media and temp files
- Images/videos are uploaded to Cloudinary; local `videos/` is temporary. Render has an ephemeral filesystem; that’s fine because files are removed after upload.

## 7) CORS
- Update CORS allowed origins in `server.ts` if needed, or set `CORS_ORIGIN` and pass it into `cors({ origin: ... })` to restrict to your frontend domain.

## 8) Health check
- Render pings `/` by default. The root route responds with `Server is Live!`; keep it for health checks.

## 9) Deploy steps summary
1. Push code to GitHub.
2. Create Render Postgres; copy `DATABASE_URL`.
3. Create Web Service pointing to `server/` with the commands above.
4. Add env vars (`DATABASE_URL`, `GOOGLE_CLOUD_API_KEY`, `CLERK_SECRET_KEY`, Cloudinary keys, optional `SENTRY_DSN`).
5. Deploy. Verify logs show `Server is running at http://localhost:PORT`.
6. Hit `/api/user/credits` with an authenticated request to confirm Clerk + DB wiring.

## 10) Common gotchas
- **Missing env**: Clerk/Cloudinary/Google keys must be present; otherwise requests fail at runtime.
- **Webhook raw body**: `/api/clerk` uses `express.raw`; keep that route before `express.json()` (already done).
- **Long-running video jobs**: VEO polling can take time; choose an instance size that tolerates that.
- **Cold starts**: On free tier, expect sleep/resume latency.
