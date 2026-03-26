[render-deployment-SKILL.md](https://github.com/user-attachments/files/26284298/render-deployment-SKILL.md)
---
name: render-deployment
description: Deploy and manage applications on Render.com. Use when the user wants to deploy a web service, background worker, cron job, or PostgreSQL database to Render. Handles render.yaml Blueprint generation, environment variable configuration, database provisioning, and deployment validation. TRIGGER when user mentions Render, render.yaml, deploying to cloud, or hosting a Node.js/Express app.
---

# Render Deployment Skill

You are an expert in deploying applications to Render.com using Infrastructure-as-Code Blueprints and the Render MCP server.

## Setup — Render MCP (Use This First)

Before doing anything, check if the Render MCP server is configured:

```bash
claude mcp add --transport http render https://mcp.render.com/mcp --header "Authorization: Bearer YOUR_RENDER_API_KEY"
```

If MCP is available, prefer it over CLI for all operations. Get the API key from Render Dashboard → Account Settings → API Keys.

## render.yaml Blueprint Structure

Always generate a complete render.yaml for Node.js/Express apps with PostgreSQL:

```yaml
services:
  - type: web
    name: your-app-name
    env: node
    region: ohio  # closest to Buffalo NY
    plan: free
    buildCommand: npm install
    startCommand: node server.js
    healthCheckPath: /api/health
    envVars:
      - key: NODE_ENV
        value: production
      - key: PORT
        value: 10000
      - key: DATABASE_URL
        fromDatabase:
          name: your-app-db
          property: connectionString
      - key: SESSION_SECRET
        generateValue: true
      - key: SENDGRID_API_KEY
        sync: false  # must be set manually in dashboard
      - key: TWILIO_ACCOUNT_SID
        sync: false
      - key: TWILIO_AUTH_TOKEN
        sync: false

databases:
  - name: your-app-db
    databaseName: your_app
    user: your_app_user
    region: ohio
    plan: free
```

## Deployment Checklist

Before deploying, verify in order:

1. **Git remote exists** — render.yaml must be in a GitHub/GitLab repo
2. **Health endpoint exists** — add `/api/health` that returns 200 OK
3. **Environment variables** — all secrets marked `sync: false` must be set manually in Render dashboard
4. **Database migrations** — run `CREATE TABLE IF NOT EXISTS` on startup, never drop tables
5. **Port binding** — app must listen on `process.env.PORT || 10000`
6. **No hardcoded secrets** — all credentials via environment variables only

## Health Check Endpoint (Required)

Always add this to server.js before deploying:

```js
app.get('/api/health', async (req, res) => {
  try {
    await pool.query('SELECT 1');
    res.json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      database: 'connected',
      version: process.env.npm_package_version || '1.0.0'
    });
  } catch (err) {
    res.status(503).json({
      status: 'unhealthy',
      database: 'disconnected',
      error: err.message
    });
  }
});
```

## PostgreSQL Connection (Production-Safe)

Always use connection pooling with SSL for Render PostgreSQL:

```js
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: process.env.NODE_ENV === 'production' 
    ? { rejectUnauthorized: false } 
    : false,
  max: 10,                    // max pool size
  idleTimeoutMillis: 30000,   // close idle connections after 30s
  connectionTimeoutMillis: 2000
});

// Test connection on startup
pool.connect((err, client, done) => {
  if (err) {
    console.error('[DB] Connection failed:', err.message);
    process.exit(1);
  }
  console.log('[DB] Connected to PostgreSQL');
  done();
});
```

## Database Migrations on Startup

Run all CREATE TABLE IF NOT EXISTS statements before starting the server:

```js
async function runMigrations() {
  console.log('[DB] Running migrations...');
  const migrations = [
    `CREATE TABLE IF NOT EXISTS staff (
      id SERIAL PRIMARY KEY,
      username TEXT UNIQUE NOT NULL,
      password TEXT NOT NULL,
      role TEXT DEFAULT 'Staff',
      active BOOLEAN DEFAULT true,
      created_at TIMESTAMP DEFAULT NOW()
    )`,
    `CREATE TABLE IF NOT EXISTS settings (
      key TEXT PRIMARY KEY,
      value TEXT
    )`,
    `CREATE TABLE IF NOT EXISTS audit_log (
      id SERIAL PRIMARY KEY,
      staff_id INTEGER,
      action TEXT NOT NULL,
      resource TEXT,
      ip_address TEXT,
      created_at TIMESTAMP DEFAULT NOW()
    )`
    // add more tables here
  ];

  for (const sql of migrations) {
    await pool.query(sql);
  }
  console.log('[DB] Migrations complete');
}

// Call before starting server
runMigrations().then(() => {
  app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
}).catch(err => {
  console.error('[DB] Migration failed:', err);
  process.exit(1);
});
```

## Environment Variables — Never Commit These

Always add to `.gitignore`:
```
.env
*.env
.env.local
.env.production
```

Always create `.env.example` with placeholder values:
```
DATABASE_URL=postgresql://user:password@host:5432/dbname
SESSION_SECRET=your-secret-here
SENDGRID_API_KEY=SG.xxxxx
TWILIO_ACCOUNT_SID=ACxxxxx
TWILIO_AUTH_TOKEN=xxxxx
NODE_ENV=development
PORT=3000
```

## Common Render Deployment Errors

| Error | Fix |
|-------|-----|
| `ECONNREFUSED` on DB | Add `ssl: { rejectUnauthorized: false }` to pool config |
| Build fails | Check Node version — add `engines: { "node": "18.x" }` to package.json |
| App crashes on start | Make sure `PORT` comes from `process.env.PORT` |
| 502 Bad Gateway | Health check failing — verify `/api/health` returns 200 |
| DB connection timeout | Increase `connectionTimeoutMillis` in pool config |
| `relation does not exist` | Migrations not running — check startup order |

## Session Management for Web Apps

For HIPAA-compliant session handling on Render:

```js
const session = require('express-session');
const PgSession = require('connect-pg-simple')(session);

app.use(session({
  store: new PgSession({
    pool,
    tableName: 'session',
    createTableIfMissing: true  // auto-creates session table
  }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',  // HTTPS only in prod
    httpOnly: true,                                  // no JS access
    maxAge: 30 * 60 * 1000,                         // 30 min timeout
    sameSite: 'strict'
  }
}));
```

## Security Hardening (Always Apply)

```js
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');

// Security headers
app.use(helmet());

// Trust Render's proxy
app.set('trust proxy', 1);

// Rate limiting on auth endpoints
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 10,                     // 10 attempts
  message: { error: 'Too many login attempts. Try again in 15 minutes.' }
});
app.use('/api/auth/login', loginLimiter);
```

## Deployment Steps

1. Push code to GitHub: `git push origin main`
2. In Render dashboard → New → Blueprint → connect repo
3. Render auto-detects render.yaml and provisions services
4. Set all `sync: false` env vars manually in Render dashboard
5. Trigger manual deploy or wait for auto-deploy on push
6. Check logs: Render dashboard → your service → Logs
7. Verify health: `curl https://your-app.onrender.com/api/health`

## Free Tier Limitations

- Web services **spin down after 15 min inactivity** — add a ping cron job to keep alive
- PostgreSQL free tier: 1GB storage, 97 connection limit
- No custom domains on free tier — upgrade to Starter ($7/mo) for custom domain

## Keep-Alive for Free Tier

Add this cron job to render.yaml if using free web service:

```yaml
  - type: cron
    name: keep-alive
    env: node
    schedule: "*/14 * * * *"  # every 14 minutes
    buildCommand: ""
    startCommand: node -e "require('https').get('https://your-app.onrender.com/api/health', r => console.log(r.statusCode))"
```

## Render MCP Commands (if configured)

```
list_services()           — list all services
deploy_service(id)        — trigger a deploy
get_service_logs(id)      — view logs
get_environment_vars(id)  — list env vars
update_environment_var(id, key, value)  — set env var
```
