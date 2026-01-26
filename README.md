# 🎬 Full Stack AI Short Video Ads Generator

<div align="center">

[![Live Frontend](https://img.shields.io/badge/Frontend-Vercel-success?style=flat-square&logo=vercel)](https://ai-ugc-ads-client.vercel.app/)
[![Live Backend](https://img.shields.io/badge/Backend-Render-important?style=flat-square&logo=render)](https://ai-ads-ugc-server.onrender.com)
[![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](./LICENSE)
[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)

Transform product images into stunning AI-generated video ads in minutes. A full-stack application leveraging Google GenAI, Cloudinary, and modern web technologies.

[View Live](https://ai-ugc-ads-client.vercel.app/) • [API Docs](#api-endpoints) • [Tech Stack](#tech-stack) • [Setup Guide](#getting-started)

</div>

---

## 🌟 Features

✨ **AI-Powered Content Creation**
- Generate professional product composite images using Google Gemini AI
- Create short-form vertical videos from images with VEO video generation
- Multi-modal AI that understands product context and lighting

💳 **Credit-Based System**
- Users start with 20 free credits
- Image generation: 5 credits | Video generation: 10 credits
- Automatic credit refunds on failed generations

📦 **Project Management**
- Create, view, edit, and delete projects
- Track generation status in real-time
- Publish projects to community feed

🔐 **Authentication & Authorization**
- Secure auth via Clerk
- User ownership enforcement
- Role-based access control

🎨 **Modern UI/UX**
- Smooth animations with Framer Motion
- Responsive design with Tailwind CSS v4
- Toast notifications for user feedback
- Smooth scrolling with Lenis

---

## 🏗️ Architecture

### High-Level Flow

```
Browser (React SPA)
  ↓ HTTPS (Axios)
Express API + TypeScript + Clerk
  ├─ Multer: File uploads
  ├─ Sentry: Error tracking
  └─ Clerk: Authentication
  ↓
Prisma ORM ←→ PostgreSQL (Users, Projects)
  ├─ Google GenAI (Image/Video generation)
  └─ Cloudinary (Media CDN)
```

### Request Lifecycle

**Image Generation Flow:**
1. User uploads 2 images + prompt → `/api/project/create`
2. Multer saves temp files → Cloudinary uploads source images
3. Google Gemini generates composite image
4. Result stored in Cloudinary → Project saved in DB

**Video Generation Flow:**
1. User requests video → `/api/project/video`
2. Load generated image from Cloudinary
3. Google VEO generates video (long-running operation)
4. Poll until complete → Download MP4
5. Upload to Cloudinary → Update DB

---

## 🛠️ Tech Stack

### Frontend
- **React 19** + **Vite** - Fast development and build
- **TypeScript** - Type safety
- **React Router** - Multi-page navigation
- **Tailwind CSS v4** - Utility-first styling
- **Framer Motion** - Smooth animations
- **Clerk React** - Authentication UI
- **Axios** - HTTP client
- **React Hot Toast** - Toast notifications
- **Lenis** - Smooth scrolling

### Backend
- **Express 5** - HTTP API framework
- **TypeScript + tsx** - Modern TS runtime
- **Clerk Express** - Authentication middleware
- **Multer** - File upload handling
- **Axios** - Server-side HTTP requests
- **Sentry** - Error tracking and monitoring
- **CORS** - Cross-origin request handling

### AI & Media
- **Google GenAI** 
  - `gemini-3-pro-image-preview` - Image generation
  - `veo-3.1-generate-preview` - Video generation
- **Cloudinary** - Image/video storage and CDN

### Database
- **PostgreSQL** - Relational database
- **Prisma ORM** - Type-safe database access
- **Prisma PG Adapter** - PostgreSQL adapter

---

## 📋 Database Schema

### User Model
```typescript
model User {
  id        String    @id @default(cuid())
  email     String    @unique
  name      String?
  image     String?
  credits   Int       @default(20)
  projects  Project[]
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
}
```

### Project Model
```typescript
model Project {
  id                String   @id @default(uuid())
  userId            String
  user              User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  name              String
  productName       String
  productDescription String
  userPrompt        String
  aspectRatio       String
  targetLength      String
  uploadedImages    String[] // URLs to original images
  generatedImage    String?  // Generated composite image URL
  generatedVideo    String?  // Generated video URL
  isGenerating      Boolean  @default(false)
  isPublished       Boolean  @default(false)
  error             String?
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
}
```

---

## 🔌 API Endpoints

### Authentication
All endpoints except `/api/project/published` require Clerk authentication via `Authorization` header.

### Project Endpoints

| Method | Endpoint | Description | Credits |
|--------|----------|-------------|---------|
| `POST` | `/api/project/create` | Create project & generate image | -5 |
| `POST` | `/api/project/video` | Generate video from image | -10 |
| `GET` | `/api/project/published` | List published projects | - |
| `DELETE` | `/api/project/:projectId` | Delete own project | - |
| `GET` | `/api/user/publish/:projectId` | Toggle publish flag | - |

### User Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/user/credits` | Get user credits |
| `GET` | `/api/user/projects` | List user's projects |
| `GET` | `/api/user/projects/:projectId` | Get project details |

### Request/Response Examples

**Create Project**
```bash
POST /api/project/create
Content-Type: multipart/form-data
Authorization: Bearer <token>

Form Data:
- images: [File, File]
- name: "Summer Campaign"
- productName: "Coffee Maker"
- productDescription: "Premium 12-cup coffee maker"
- userPrompt: "studio lighting, lifestyle setting"
- aspectRatio: "9:16"
- targetLength: "15"

Response:
{
  "id": "uuid",
  "name": "Summer Campaign",
  "productName": "Coffee Maker",
  "generatedImage": "https://cloudinary.com/...",
  "isGenerating": false,
  "credits": 15
}
```

**Generate Video**
```bash
POST /api/project/video
Content-Type: application/json
Authorization: Bearer <token>

{
  "projectId": "uuid"
}

Response:
{
  "id": "uuid",
  "generatedVideo": "https://cloudinary.com/...mp4",
  "isGenerating": false,
  "credits": 5
}
```

---

## 🚀 Getting Started

### Prerequisites
- **Node.js** 18+ and npm/yarn
- **PostgreSQL** database (local or managed)
- **Clerk** account (free at https://clerk.com)
- **Cloudinary** account (free at https://cloudinary.com)
- **Google Cloud** account with GenAI API enabled

### Environment Setup

**Frontend** (`.env.local` in `client/`):
```env
VITE_CLERK_PUBLISHABLE_KEY=your_clerk_publishable_key
VITE_API_URL=http://localhost:5000
```

**Backend** (`.env` in `server/`):
```env
DATABASE_URL=postgresql://user:password@localhost:5432/ai-ads
CLERK_SECRET_KEY=your_clerk_secret_key
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret
GOOGLE_GENAI_API_KEY=your_google_api_key
SENTRY_DSN=your_sentry_dsn (optional)
```

### Installation

1. **Clone the repository**
```bash
git clone https://github.com/Sumit123sm/Full-Stack-AI-Short-Video-Ads-Generator.git
cd Full-Stack-AI-Short-Video-Ads-Generator
```

2. **Setup Backend**
```bash
cd server
npm install

# Setup database
npx prisma migrate dev
npx prisma generate

# Start backend
npm run dev
# Backend runs on http://localhost:5000
```

3. **Setup Frontend**
```bash
cd ../client
npm install

# Start frontend
npm run dev
# Frontend runs on http://localhost:5173
```

4. **Access the app**
- Open http://localhost:5173 in your browser
- Sign up with Clerk
- Start creating ads!

---

## 🔐 Security Features

- ✅ **Clerk Authentication** - Industry-standard JWT-based auth
- ✅ **Ownership Enforcement** - All operations scoped by `userId`
- ✅ **CORS Protection** - Configurable origin allowlist
- ✅ **Environment Secrets** - Sensitive keys never committed
- ✅ **Publish Toggle** - Only generated media can be published
- ✅ **Generation Guards** - `isGenerating` flag prevents race conditions
- ✅ **Error Logging** - Sentry integration for production monitoring

---

## 📊 Performance & Scalability

### Current Optimizations
- **Vite + React** for fast client-side rendering
- **Cloudinary CDN** for media delivery
- **Prisma indexes** on foreign keys
- **Async uploads** in parallel for multiple images
- **Caching** when media already generated

### Scalability Roadmap
- **Job Queue** (BullMQ) for long-running AI tasks
- **Webhook Integration** to replace polling
- **Multi-region Deployment** for global users
- **Connection Pooling** for database
- **Published Feed Caching** with Redis

---

## 🐛 Error Handling

- **Generation Failures** → Credit refunds applied automatically
- **Network Issues** → Retry logic with exponential backoff
- **Auth Errors** → Clear redirect to login
- **Rate Limiting** → Graceful error messages
- **Sentry Integration** → Real-time error tracking in production

---

## 🎯 Key Use Cases

1. **E-commerce Brands** - Create product showcase videos instantly
2. **Marketing Agencies** - Generate client ads at scale
3. **Social Media Creators** - Produce vertical format content
4. **Startup Founders** - DIY product promotion without crew
5. **Small Businesses** - Cost-effective professional ads

---

## 📈 Credit System

| Action | Cost | Refund on Failure |
|--------|------|-------------------|
| Image Generation | 5 credits | ✅ Yes |
| Video Generation | 10 credits | ✅ Yes |
| Delete Project | 0 credits | - |

Users start with **20 free credits**. Failed generations automatically refund credits.

---

## 🔄 Generation Status Tracking

Projects include `isGenerating` flag to track state:
- `isGenerating: true` - Currently processing
- `isGenerating: false` - Complete (check `error` field for failures)
- `error: null` - Successful generation
- `error: "message"` - Generation failed with error

---

## 🤝 Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE) file for details.

---

## 👨‍💻 Author

**Sumit Sharma**
- GitHub: [@Sumit123sm](https://github.com/Sumit123sm)
- Project Repo: [Full-Stack-AI-Short-Video-Ads-Generator](https://github.com/Sumit123sm/Full-Stack-AI-Short-Video-Ads-Generator)

---

## 📚 Resources

- [Google GenAI Documentation](https://ai.google.dev/)
- [Clerk Documentation](https://clerk.com/docs)
- [Cloudinary API Docs](https://cloudinary.com/documentation)
- [Prisma Documentation](https://www.prisma.io/docs/)
- [Express.js Guide](https://expressjs.com/)

---

## 🎉 Acknowledgments

- **Google GenAI** for powerful image and video generation
- **Clerk** for seamless authentication
- **Cloudinary** for reliable media hosting
- **Vercel** and **Render** for excellent deployment platforms
- React and TypeScript communities for amazing tools

---

<div align="center">

**[Live Demo](https://ai-ugc-ads-client.vercel.app/)** • **[API](https://ai-ads-ugc-server.onrender.com)** • **[Issues](https://github.com/Sumit123sm/Full-Stack-AI-Short-Video-Ads-Generator/issues)** • **[Discussions](https://github.com/Sumit123sm/Full-Stack-AI-Short-Video-Ads-Generator/discussions)**

Made with ❤️ by Sumit Sharma

</div>
