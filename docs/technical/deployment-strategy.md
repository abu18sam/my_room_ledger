# Living Deployment Strategy & Production Readiness Plan

**Project:** Room & Rent Ledger Management System (*My Room Ledger*)  
**Document Status:** Living Specification & Multi-Stage Deployment Plan ✅  
**Date:** 2026-07-27  
**Traceability Reference:** `MASTER.md` $\rightarrow$ `docs/stages/01-requirement-analysis.md` $\rightarrow$ `docs/00-engineering-workflow.md`

---

## 1. Executive Summary & Strategy Goals

This living deployment document ensures that **My Room Ledger** is designed from day one to be easily deployable, cost-effective, containerized, and scale-ready. Although production rollout will occur after local development is completed, all architectural decisions align with the following deployment principles:

1. **Zero Deployment Headache**: Standardized containerization (`Dockerfile` + `docker-compose.yml`) ensuring 100% environment parity between local development and cloud production.
2. **Cost Optimization**: Leveraging free-tier and low-cost infrastructure (targeting **$0 to $5/month** for initial rollout).
3. **Stateless Scalability**: JWT-based stateless backend architecture allowing zero-downtime rolling container restarts.
4. **Pluggable File Storage**: Abstracted storage driver supporting local filesystem in development and S3-compatible cloud storage (Cloudflare R2 / AWS S3) in production for supplier electricity bills.

---

## 2. Hosting Options Analysis & Cost Comparison

Below is a detailed comparison of production hosting alternatives evaluated for initial launch and future scaling:

### 📊 Hosting Options Comparison Matrix

| Option / Provider | Stack Breakdown | Estimated Monthly Cost | Memory / CPU Limits | Setup Ease | Scalability & Pros | Tradeoffs & Cons |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Option A: PaaS + Managed DB (Recommended Initial)** | **Frontend**: Cloudflare Pages / Vercel<br>**Backend**: Render / Railway<br>**DB**: Neon / Supabase PostgreSQL<br>**Storage**: Cloudflare R2 | **$0 – $5 / month** (Free Tiers available) | • FE: Unlimited CDN<br>• BE: 512MB RAM<br>• DB: 0.5GB – 1GB storage<br>• R2: 10GB free | ⭐⭐⭐⭐⭐ (Easiest) | • Zero DevOps overhead<br>• Automatic SSL/TLS<br>• Managed DB backups | • Free tier web services sleep on idle (Render ~50s cold start; mitigated by Railway or cron ping) |
| **Option B: Low-Cost VPS (Recommended Low-Latency)** | **Frontend**: Next.js PWA on Vercel<br>**Backend**: Dockerized Node.js API<br>**DB**: Dockerized PostgreSQL<br>**Storage**: Cloudflare R2 | **$4 – $6 / month** (Hetzner / DigitalOcean / Linode) | • 2GB RAM / 1 vCPU<br>• 20GB – 40GB SSD | ⭐⭐⭐ (Moderate) | • Full server control<br>• Zero cold starts<br>• High CPU/RAM performance | • Requires manual Nginx, SSL (Certbot), and DB backup cron configuration |
| **Option C: Enterprise Cloud (Future Scale)** | **Frontend**: Vercel Enterprise / CloudFront<br>**Backend**: AWS ECS / Fargate<br>**DB**: AWS RDS PostgreSQL<br>**Storage**: Cloudflare R2 / S3 | **$25 – $50+ / month** | • Dynamic auto-scaling | ⭐⭐ (Complex) | • Enterprise SLA<br>• Unlimited horizontal scaling | • High monthly fixed costs<br>• Over-engineered for initial launch phase |

---

## 3. Recommended Infrastructure Stack for Launch

For initial production rollout, we adopt **Security-First PaaS Stack**:

```
                               +----------------------------------+
                               |     CLIENT DEVICES (PWA UI)      |
                               | (Desktop, Tablet, Mobile PWA)    |
                               +----------------+-----------------+
                                                |
                                                v HTTPS / JSON REST APIs
                               +----------------+-----------------+
                               |             Vercel               |
                               |  (Next.js TypeScript PWA + CDN)  |
                               +----------------+-----------------+
                                                |
                                                v Encrypted JSON REST Requests + Auth
                               +----------------+-----------------+
                               |   Backend Service (Docker)       |
                               | (Node.js/NestJS TypeScript API)  |
                               +-------+----------------+---------+
                                       |                |
                        Prisma ORM SQL |                | Encrypted Blob Stream
                       (TLS encrypted) |                | (AES-256 GCM + Signed URLs)
                                       v                v
                   +-------------------+---+        +---+-------------------+
                   | Managed PostgreSQL    |        | Cloudflare R2         |
                   | (Neon / Supabase DB)  |        | (Encrypted Vault)     |
                   +-----------------------+        +-----------------------+
```

### 3.1 Component Specifications

1. **Frontend Hosting**: **Vercel**
   - Next.js TypeScript application delivered as an installable Progressive Web App (`next-pwa`).
   - Tailwind CSS + ShadCN UI + Recharts components compiled to serverless edge functions & static CDN assets.
   - Cost: **$0.00 / month** (Vercel Hobby plan).

2. **Backend API**: **Node.js (TypeScript) + NestJS 10**
   - RESTful API service with Zod schema validation, JWT auth, and AES-256 GCM file encryption module.
   - Containerized via Node.js alpine Docker image deployed on Render or Railway.
   - Cost: **$0.00 – $5.00 / month**.

3. **Database Tier**: **PostgreSQL + Prisma ORM**
   - PostgreSQL hosted on Neon / Supabase with Prisma ORM managing schema migrations and type-safe queries.
   - Stores users, metadata, transaction ledgers, audit logs (NO raw file blobs).
   - Cost: **$0.00 / month**.

4. **Encrypted File Vault**: **Cloudflare R2**
   - Object storage for encrypted Government IDs, receipts, and supplier bills.
   - Files accessed ONLY via short-lived backend-generated signed URLs (`/api/v1/files/signed-url`).
   - Cost: **$0.00 / month** (10GB storage free, 0 egress cost).

---

## 4. Containerization & Local Development Parity

To ensure deployment requires zero code modification, backend services will include a multi-stage `Dockerfile`:

### 4.1 Production Multi-Stage Node.js TypeScript `Dockerfile`

```dockerfile
# Stage 1: Build TypeScript Application
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
COPY prisma ./prisma/
RUN npm ci
COPY . .
RUN npx prisma generate
RUN npm run build

# Stage 2: Minimal Production Execution Container
FROM node:18-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -g 1001 -S nodejs && adduser -u 1001 -S nodeuser -G nodejs
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/prisma ./prisma
USER nodeuser
EXPOSE 8080
ENV PORT=8080
CMD ["node", "dist/server.js"]
```

---

## 5. Step-by-Step Production Deployment Execution Checklist

When local development is complete and approved, deployment will proceed following this checklist:

### Phase 1: Environment & Secrets Setup
- [ ] Provision PostgreSQL database instance (Neon / Supabase / VPS PostgreSQL).
- [ ] Create Cloudflare R2 Bucket for encrypted document storage.
- [ ] Configure Environment Variables (`DATABASE_URL`, `JWT_SECRET`, `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, `ENCRYPTION_MASTER_KEY`, `CORS_ORIGIN`).

### Phase 2: Backend Container Rollout & Migration
- [ ] Trigger container build (`docker build -t room-ledger-backend .`).
- [ ] Execute Prisma schema deployment (`npx prisma migrate deploy`).
- [ ] Verify HTTP `/api/v1/health` endpoint returns `STATUS: UP`.

### Phase 3: Frontend Deployment
- [ ] Configure `NEXT_PUBLIC_API_BASE_URL` pointing to backend production domain.
- [ ] Execute production build & deploy on Vercel (`git push origin main`).

### Phase 4: Domain, SSL, and Security Setup
- [ ] Attach custom domain (e.g., `app.myroomledger.com`).
- [ ] Enable HTTPS / SSL via Cloudflare / Let's Encrypt.
- [ ] Verify CORS policy strictly restricts API access to frontend domain.

### Phase 5: Monitoring & Automated Backups
- [ ] Configure automated database daily snapshots.
- [ ] Set up basic uptime monitoring (UptimeRobot / Better Stack free tier).

---

## 6. Document Version History

| Version | Date | Description of Changes | Author |
| :--- | :--- | :--- | :--- |
| **v1.0** | 2026-07-27 | Initial living deployment strategy document created during Stage 01 Requirement Analysis. Established containerization requirements, low-cost hosting comparison, and rollout checklist. | Lead Software Architect |
