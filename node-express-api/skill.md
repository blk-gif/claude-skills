[node-express-api-SKILL.md](https://github.com/user-attachments/files/26286536/node-express-api-SKILL.md)
---
name: node-express-api
description: Build production-grade Node.js REST APIs with Express, PostgreSQL, authentication, and HIPAA-compliant patterns. TRIGGER when building backend servers, REST APIs, Express apps, or any Node.js web service. Covers routing, middleware, auth, database queries, error handling, and security.
---

# Node.js Express API Skill

You are a senior Node.js backend engineer specializing in production-grade Express APIs with PostgreSQL, security hardening, and HIPAA compliance patterns.

## Project Structure

Always organize Express apps like this:

```
server.js          — app entry point, server startup
routes/
  auth.js          — login, logout, session
  patients.js      — patient CRUD
  appointments.js  — scheduling
  staff.js         — staff management (admin only)
middleware/
  auth.js          — requireAuth, requireAdmin
  audit.js         — HIPAA audit logging
  rateLimit.js     — rate limiters
db/
  pool.js          — pg connection pool
  migrations.js    — CREATE TABLE IF NOT EXISTS
.env.example       — env var template
```

## Express App Setup

```js
require('dotenv').config();
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');

const app = express();
const PORT = process.env.PORT || 3000;

// Security first
app.use(helmet());
app.set('trust proxy', 1);

// Body parsing
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// Static files
app.use(express.static('public'));

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/patients', require('./middleware/auth').requireAuth, require('./routes/patients'));
app.use('/api/staff', require('./middleware/auth').requireAdmin, require('./routes/staff'));

// Global error handler — always last
app.use((err, req, res, next) => {
  console.error('[Error]', err.message);
  res.status(err.status || 500).json({
    error: process.env.NODE_ENV === 'production' 
      ? 'Internal server error' 
      : err.message
  });
});
```

## Auth Middleware

```js
// middleware/auth.js
function requireAuth(req, res, next) {
  if (!req.session?.staff) {
    return res.status(401).json({ error: 'Authentication required' });
  }
  next();
}

function requireAdmin(req, res, next) {
  if (!req.session?.staff) {
    return res.status(401).json({ error: 'Authentication required' });
  }
  if (req.session.staff.role !== 'Admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
}

module.exports = { requireAuth, requireAdmin };
```

## HIPAA Audit Logging

Log every PHI access — this is non-negotiable for HIPAA compliance:

```js
// middleware/audit.js
const { pool } = require('../db/pool');

async function auditLog(req, action, resource, resourceId = null) {
  try {
    await pool.query(
      `INSERT INTO audit_log (staff_id, staff_name, action, resource, resource_id, ip_address, user_agent)
       VALUES ($1, $2, $3, $4, $5, $6, $7)`,
      [
        req.session?.staff?.id,
        req.session?.staff?.name,
        action,
        resource,
        resourceId,
        req.ip,
        req.get('user-agent')
      ]
    );
  } catch (err) {
    // Never let audit failures break the app — just log
    console.error('[Audit] Failed to log:', err.message);
  }
}

module.exports = { auditLog };
```

Use on every patient endpoint:
```js
app.get('/api/patients/:id', requireAuth, async (req, res) => {
  await auditLog(req, 'VIEW', 'patient', req.params.id);
  // ... fetch patient
});
```

## PostgreSQL Query Patterns

Always use parameterized queries — never string concatenation:

```js
// CORRECT — parameterized
const result = await pool.query(
  'SELECT * FROM patients WHERE id = $1 AND active = true',
  [req.params.id]
);

// WRONG — SQL injection risk
const result = await pool.query(
  `SELECT * FROM patients WHERE id = ${req.params.id}`  // NEVER DO THIS
);
```

Standard CRUD pattern:
```js
// GET list with search
app.get('/api/patients', requireAuth, async (req, res) => {
  try {
    const { search, page = 1, limit = 20 } = req.query;
    const offset = (page - 1) * limit;
    
    let query = 'SELECT * FROM patients WHERE active = true';
    const params = [];
    
    if (search) {
      params.push(`%${search}%`);
      query += ` AND (first_name ILIKE $${params.length} OR last_name ILIKE $${params.length} OR phone ILIKE $${params.length})`;
    }
    
    params.push(limit, offset);
    query += ` ORDER BY last_name ASC LIMIT $${params.length - 1} OFFSET $${params.length}`;
    
    const result = await pool.query(query, params);
    await auditLog(req, 'LIST', 'patients');
    res.json(result.rows);
  } catch (err) {
    next(err);
  }
});

// POST create
app.post('/api/patients', requireAuth, async (req, res, next) => {
  try {
    const { first_name, last_name, dob, phone, email } = req.body;
    
    if (!first_name || !last_name) {
      return res.status(400).json({ error: 'First and last name required' });
    }
    
    const result = await pool.query(
      `INSERT INTO patients (first_name, last_name, dob, phone, email)
       VALUES ($1, $2, $3, $4, $5) RETURNING *`,
      [first_name, last_name, dob, phone, email]
    );
    
    await auditLog(req, 'CREATE', 'patient', result.rows[0].id);
    res.status(201).json(result.rows[0]);
  } catch (err) {
    next(err);
  }
});

// PUT update
app.put('/api/patients/:id', requireAuth, async (req, res, next) => {
  try {
    const { first_name, last_name, phone, email } = req.body;
    const result = await pool.query(
      `UPDATE patients SET first_name=$1, last_name=$2, phone=$3, email=$4, updated_at=NOW()
       WHERE id=$5 RETURNING *`,
      [first_name, last_name, phone, email, req.params.id]
    );
    
    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'Patient not found' });
    }
    
    await auditLog(req, 'UPDATE', 'patient', req.params.id);
    res.json(result.rows[0]);
  } catch (err) {
    next(err);
  }
});
```

## Input Validation

Always validate before hitting the database:

```js
function validatePatient(data) {
  const errors = [];
  if (!data.first_name?.trim()) errors.push('First name is required');
  if (!data.last_name?.trim()) errors.push('Last name is required');
  if (data.email && !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data.email)) {
    errors.push('Invalid email format');
  }
  if (data.phone && !/^\+?[\d\s\-\(\)]{10,}$/.test(data.phone)) {
    errors.push('Invalid phone format');
  }
  return errors;
}

// Use in routes
const errors = validatePatient(req.body);
if (errors.length > 0) {
  return res.status(400).json({ errors });
}
```

## Session Timeout — 30 Minutes (HIPAA)

```js
app.use((req, res, next) => {
  if (req.session?.staff) {
    const now = Date.now();
    const timeout = 30 * 60 * 1000; // 30 minutes
    
    if (req.session.lastActivity && (now - req.session.lastActivity > timeout)) {
      return req.session.destroy(() => {
        res.status(401).json({ error: 'Session expired', redirect: '/login?timeout=1' });
      });
    }
    req.session.lastActivity = now;
  }
  next();
});
```

## Password Hashing — Always bcrypt

```js
const bcrypt = require('bcrypt');
const SALT_ROUNDS = 10;

// Hash on create/update
const hash = await bcrypt.hash(plainPassword, SALT_ROUNDS);

// Verify on login
const valid = await bcrypt.compare(plainPassword, storedHash);

// NEVER use MD5, SHA1, SHA256 for passwords
// NEVER store plaintext passwords
```

## Error Response Standard

Always return consistent error shapes:

```js
// Success
res.json({ success: true, data: result });

// Validation error
res.status(400).json({ error: 'Validation failed', errors: ['Field required'] });

// Auth error  
res.status(401).json({ error: 'Authentication required' });

// Permission error
res.status(403).json({ error: 'Admin access required' });

// Not found
res.status(404).json({ error: 'Resource not found' });

// Server error (never expose internals in production)
res.status(500).json({ error: 'Internal server error' });
```

## Anti-Patterns to Avoid

- ❌ Never use `SELECT *` in production — specify columns
- ❌ Never catch errors silently — always log or rethrow
- ❌ Never store secrets in code — use environment variables
- ❌ Never skip input validation before database queries
- ❌ Never use `req.query` directly in SQL — parameterize
- ❌ Never disable helmet or CORS without good reason
- ❌ Never expose stack traces to clients in production
- ❌ Never use synchronous `bcrypt` functions — use async
