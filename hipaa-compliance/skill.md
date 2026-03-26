[hipaa-compliance-SKILL.md](https://github.com/user-attachments/files/26286548/hipaa-compliance-SKILL.md)
---
name: hipaa-compliance
description: HIPAA technical safeguards checklist and PHI handling patterns for healthcare applications. TRIGGER when building or auditing any healthcare app, patient management system, or any app that handles Protected Health Information (PHI). Covers encryption, access controls, audit logging, session management, and breach prevention.
---

# HIPAA Compliance Skill

You are a HIPAA security expert. When this skill is active, scan all code for PHI handling violations and enforce technical safeguards.

## What is PHI — Always Protect These Fields

Protected Health Information includes ANY data that could identify a patient:
- Names, addresses, phone numbers, email addresses
- Dates (DOB, admission, discharge, death)
- Social Security Numbers
- Medical record numbers, health plan numbers
- Account numbers, certificate numbers
- Device identifiers, web URLs, IP addresses (when linked to health data)
- Photos, fingerprints, voice recordings
- Any unique identifying number or code
- Diagnosis codes, procedure codes, treatment notes

## Technical Safeguard Checklist

### Access Controls ✓
- [ ] Every PHI endpoint requires authentication
- [ ] Role-based access — staff see only what their role allows
- [ ] Admin-only routes protected with `requireAdmin` middleware
- [ ] Session timeout after 30 minutes of inactivity (HIPAA minimum)
- [ ] Account lockout after failed login attempts
- [ ] Unique usernames — no shared accounts

### Audit Controls ✓
- [ ] Log every PHI access: who, what, when, from where
- [ ] Audit log includes: staff_id, staff_name, action, resource, resource_id, ip_address, timestamp
- [ ] Audit logs are immutable — never allow UPDATE or DELETE on audit_log table
- [ ] Audit logs retained for minimum 6 years
- [ ] Failed login attempts logged

### Transmission Security ✓
- [ ] HTTPS enforced in production (Render handles this automatically)
- [ ] Secure session cookies: `httpOnly: true`, `secure: true` in production
- [ ] `sameSite: 'strict'` on session cookies
- [ ] No PHI in URL parameters (use POST body instead)
- [ ] No PHI in console.log statements

### Storage Security ✓
- [ ] Passwords hashed with bcrypt (minimum 10 rounds)
- [ ] Database credentials in environment variables only
- [ ] No PHI in log files
- [ ] Database backups encrypted
- [ ] No PHI in error messages returned to client

### PHI Scanning — Look For These Violations

#### Console.log with PHI (VIOLATION)
```js
// BAD — logs PHI
console.log('Patient:', patient.first_name, patient.ssn);
console.log('Login:', username, password);

// GOOD — log IDs only
console.log('Patient accessed:', patient.id);
console.log('Login attempt for user ID:', userId);
```

#### PHI in URL parameters (VIOLATION)
```js
// BAD — PHI in URL
app.get('/api/patients?name=JohnDoe&dob=1990-01-01', ...)
res.redirect(`/patient?ssn=${ssn}`);

// GOOD — use IDs in URL, PHI in body
app.get('/api/patients/:id', ...)
app.post('/api/patients/search', ...)
```

#### Missing auth on PHI endpoint (VIOLATION)
```js
// BAD — no auth check
app.get('/api/patients', async (req, res) => { ... });

// GOOD — auth required
app.get('/api/patients', requireAuth, async (req, res) => { ... });
```

#### Missing audit log (VIOLATION)
```js
// BAD — no audit trail
app.get('/api/patients/:id', requireAuth, async (req, res) => {
  const patient = await getPatient(req.params.id);
  res.json(patient);
});

// GOOD — audit every access
app.get('/api/patients/:id', requireAuth, async (req, res) => {
  await auditLog(req, 'VIEW', 'patient', req.params.id);
  const patient = await getPatient(req.params.id);
  res.json(patient);
});
```

#### Weak password hashing (VIOLATION)
```js
// BAD — never use these for passwords
const hash = crypto.createHash('sha256').update(password).digest('hex');
const hash = md5(password);
const stored = password; // plaintext

// GOOD
const hash = await bcrypt.hash(password, 10);
```

#### PHI in error responses (VIOLATION)
```js
// BAD — exposes internal data
res.status(500).json({ error: err.message, stack: err.stack, query: sql });

// GOOD
console.error('[Error]', err); // log internally
res.status(500).json({ error: 'Internal server error' }); // generic to client
```

## Audit Log Table (Required)

```sql
CREATE TABLE IF NOT EXISTS audit_log (
  id          SERIAL PRIMARY KEY,
  staff_id    INTEGER,
  staff_name  TEXT,
  action      TEXT NOT NULL,  -- VIEW, CREATE, UPDATE, DELETE, LOGIN, LOGOUT, EXPORT
  resource    TEXT,           -- patient, appointment, soap_note, etc.
  resource_id INTEGER,
  ip_address  TEXT,
  user_agent  TEXT,
  details     TEXT,           -- optional JSON with non-PHI context
  created_at  TIMESTAMP DEFAULT NOW()
);

-- NEVER add UPDATE or DELETE permissions on this table
-- Add index for reporting
CREATE INDEX IF NOT EXISTS idx_audit_log_staff ON audit_log(staff_id);
CREATE INDEX IF NOT EXISTS idx_audit_log_resource ON audit_log(resource, resource_id);
CREATE INDEX IF NOT EXISTS idx_audit_log_created ON audit_log(created_at);
```

## Session Security Implementation

```js
app.use(session({
  store: new PgSession({ pool, createTableIfMissing: true }),
  secret: process.env.SESSION_SECRET,  // min 32 random chars
  resave: false,
  saveUninitialized: false,
  name: 'sid',  // don't use default 'connect.sid'
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 30 * 60 * 1000,  // 30 minutes
    sameSite: 'strict'
  }
}));

// Session activity tracker
app.use((req, res, next) => {
  if (req.session?.staff) {
    const now = Date.now();
    if (req.session.lastActivity && now - req.session.lastActivity > 30 * 60 * 1000) {
      auditLog(req, 'SESSION_TIMEOUT', 'auth');
      return req.session.destroy(() => {
        res.status(401).json({ error: 'Session expired', redirect: '/login?timeout=1' });
      });
    }
    req.session.lastActivity = now;
  }
  next();
});
```

## Backup Requirements

- Daily backups of all patient data
- Backups encrypted at rest
- Backups retained minimum 6 years (7 years recommended)
- Test restoration quarterly
- Log all backup operations

## Business Associate Agreements (BAAs) Required

These vendors require signed BAAs before storing PHI:
- [ ] Render.com (hosting) — request via support
- [ ] SendGrid (email) — available in dashboard
- [ ] Twilio (SMS) — available in dashboard
- [ ] Stripe (payments) — available in dashboard
- [ ] AWS S3 (storage) — via AWS console
- [ ] Any cloud database provider

**Technical safeguards alone do NOT make an app HIPAA compliant.**
Full compliance also requires: BAAs, Privacy Policies, workforce training, physical safeguards, and incident response procedures.

## Security Headers (Always Apply)

```js
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:'],
    }
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true
  }
}));
```

## Minimum Necessary Standard

Only expose the minimum PHI needed for each function:

```js
// BAD — returns entire patient record to everyone
app.get('/api/appointments', requireAuth, async (req, res) => {
  const result = await pool.query('SELECT * FROM appointments JOIN patients ON ...');
  res.json(result.rows); // includes SSN, full DOB, insurance details
});

// GOOD — return only what's needed for the appointment list view
app.get('/api/appointments', requireAuth, async (req, res) => {
  const result = await pool.query(`
    SELECT a.id, a.date, a.time, a.status,
           p.first_name, p.last_name, p.phone
    FROM appointments a
    JOIN patients p ON p.id = a.patient_id
  `);
  res.json(result.rows);
});
```
