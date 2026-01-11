# Security Fixes and Recommendations

This document provides specific code changes and configurations to address the security issues identified in the security review.

---

## 1. Fix JWT Token Expiration

### Current Code (server/middleware/auth.js)
```javascript
const generateToken = (user) => {
  return jwt.sign(
    { 
      userId: user.id, 
      username: user.username 
    },
    JWT_SECRET
    // No expiration - token lasts forever
  );
};
```

### Recommended Fix
```javascript
const generateToken = (user) => {
  return jwt.sign(
    { 
      userId: user.id, 
      username: user.username 
    },
    JWT_SECRET,
    { expiresIn: '24h' } // Token expires in 24 hours
  );
};

// Add refresh token generation
const generateRefreshToken = (user) => {
  return jwt.sign(
    { 
      userId: user.id, 
      username: user.username,
      type: 'refresh'
    },
    JWT_SECRET,
    { expiresIn: '7d' } // Refresh token expires in 7 days
  );
};
```

---

## 2. Enforce Strong JWT Secret

### Current Code (server/middleware/auth.js)
```javascript
const JWT_SECRET = process.env.JWT_SECRET || 'claude-ui-dev-secret-change-in-production';
```

### Recommended Fix
```javascript
const JWT_SECRET = process.env.JWT_SECRET;

if (!JWT_SECRET || JWT_SECRET === 'claude-ui-dev-secret-change-in-production') {
  console.error('❌ SECURITY ERROR: JWT_SECRET must be set in production');
  console.error('Generate a strong secret with: node -e "console.log(require(\'crypto\').randomBytes(64).toString(\'hex\'))"');
  
  if (process.env.NODE_ENV === 'production') {
    process.exit(1); // Exit in production
  }
  
  console.warn('⚠️  WARNING: Using default JWT secret in development. DO NOT use in production!');
}
```

### Add to .env.example
```bash
# JWT Secret (REQUIRED for production - generate with: node -e "console.log(require('crypto').randomBytes(64).toString('hex'))")
JWT_SECRET=your-very-long-random-secret-key-here-at-least-64-characters
```

---

## 3. Fix Command Injection in Git Routes

### Current Vulnerable Code (server/routes/git.js)
```javascript
await execAsync(`git commit -m "${message.replace(/"/g, '\\"')}"`, { cwd: projectPath });
await execAsync(`git checkout "${branch}"`, { cwd: projectPath });
```

### Recommended Fix
Use Node.js child_process with array arguments instead of string interpolation:

```javascript
import { spawn } from 'child_process';
import { promisify } from 'util';

// Helper function for safe git execution
async function execGitCommand(args, cwd) {
  return new Promise((resolve, reject) => {
    const process = spawn('git', args, { cwd });
    let stdout = '';
    let stderr = '';
    
    process.stdout.on('data', (data) => stdout += data.toString());
    process.stderr.on('data', (data) => stderr += data.toString());
    
    process.on('close', (code) => {
      if (code === 0) {
        resolve({ stdout, stderr });
      } else {
        reject(new Error(stderr || stdout));
      }
    });
    
    process.on('error', reject);
  });
}

// Use like this:
await execGitCommand(['commit', '-m', message], projectPath);
await execGitCommand(['checkout', branch], projectPath);
await execGitCommand(['checkout', '-b', branch], projectPath);
```

---

## 4. Implement Path Traversal Prevention

### Current Code (server/index.js)
```javascript
if (!filePath || !path.isAbsolute(filePath)) {
  return res.status(400).json({ error: 'Invalid file path' });
}
```

### Recommended Fix
```javascript
// Helper function to validate file path
function isPathSafe(filePath, projectPath) {
  if (!filePath || !path.isAbsolute(filePath)) {
    return false;
  }
  
  // Normalize paths to resolve .. and . components
  const normalizedFilePath = path.normalize(filePath);
  const normalizedProjectPath = path.normalize(projectPath);
  
  // Ensure the file path is within the project directory
  return normalizedFilePath.startsWith(normalizedProjectPath);
}

// Use in endpoints:
app.get('/api/projects/:projectName/file', authenticateToken, async (req, res) => {
  try {
    const { projectName } = req.params;
    const { filePath } = req.query;
    
    // Get project directory
    const projectPath = await extractProjectDirectory(projectName);
    
    // Validate path is safe
    if (!isPathSafe(filePath, projectPath)) {
      return res.status(403).json({ 
        error: 'Access denied',
        details: 'File path is outside the project directory'
      });
    }
    
    const content = await fsPromises.readFile(filePath, 'utf8');
    res.json({ content, path: filePath });
  } catch (error) {
    // ...
  }
});
```

---

## 5. Make OpenAI Integration Optional

### Add to .env.example
```bash
# OpenAI API Key (optional - only needed for voice transcription feature)
# Leave empty to disable voice features
OPENAI_API_KEY=

# Enable/disable OpenAI features
ENABLE_VOICE_FEATURES=false
```

### Update server/index.js
```javascript
app.post('/api/transcribe', authenticateToken, async (req, res) => {
  // Check if voice features are enabled
  if (process.env.ENABLE_VOICE_FEATURES !== 'true') {
    return res.status(403).json({ 
      error: 'Voice features are disabled',
      details: 'Enable ENABLE_VOICE_FEATURES in .env to use transcription'
    });
  }
  
  const apiKey = process.env.OPENAI_API_KEY;
  if (!apiKey) {
    return res.status(500).json({ 
      error: 'OpenAI API key not configured',
      details: 'Set OPENAI_API_KEY in server environment to enable voice features'
    });
  }
  
  // ... rest of the code
});
```

### Add Privacy Notice to README
```markdown
## Privacy and Data Sharing

### Voice Transcription Feature (Optional)

When enabled, the voice transcription feature sends audio data to OpenAI's API:
- Audio recordings are uploaded to OpenAI's Whisper API for transcription
- Transcribed text may be further processed by GPT models for enhancement
- This feature is **disabled by default** and requires explicit configuration

To enable voice features:
1. Obtain an OpenAI API key
2. Set `OPENAI_API_KEY` in your `.env` file
3. Set `ENABLE_VOICE_FEATURES=true` in your `.env` file

**Important:** By enabling this feature, you acknowledge that your audio data will be sent to OpenAI and processed according to [OpenAI's Privacy Policy](https://openai.com/policies/privacy-policy).

### Local-Only Features

All other features operate locally and do not send data to external services:
- Gemini CLI integration (communicates only with Google's Gemini API as configured in Gemini CLI)
- File management and editing
- Git operations
- Session management
```

---

## 6. Add Rate Limiting for Authentication

### Install Required Package
```bash
npm install express-rate-limit
```

### Add to server/index.js
```javascript
import rateLimit from 'express-rate-limit';

// Rate limiter for authentication endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 requests per window
  message: 'Too many authentication attempts, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
});

// Rate limiter for general API endpoints
const apiLimiter = rateLimit({
  windowMs: 1 * 60 * 1000, // 1 minute
  max: 100, // 100 requests per window
  message: 'Too many requests, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
});

// Apply rate limiters
app.use('/api/auth/login', authLimiter);
app.use('/api/auth/register', authLimiter);
app.use('/api', apiLimiter);
```

---

## 7. Add Security Headers

### Install Required Package
```bash
npm install helmet
```

### Add to server/index.js
```javascript
import helmet from 'helmet';

// Apply security headers
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"], // Needed for React
      styleSrc: ["'self'", "'unsafe-inline'"], // Needed for styled components
      imgSrc: ["'self'", "data:", "blob:"],
      connectSrc: ["'self'", "ws:", "wss:"], // Allow WebSocket connections
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  },
}));
```

---

## 8. Add Audit Logging

### Create logger utility (server/utils/logger.js)
```javascript
import fs from 'fs';
import path from 'path';
import { fileURLToPath } from 'url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

const LOG_FILE = path.join(__dirname, '../logs/audit.log');

// Ensure log directory exists
fs.mkdirSync(path.dirname(LOG_FILE), { recursive: true });

export function logAuditEvent(event) {
  const timestamp = new Date().toISOString();
  const logEntry = {
    timestamp,
    ...event
  };
  
  fs.appendFileSync(
    LOG_FILE, 
    JSON.stringify(logEntry) + '\n',
    'utf8'
  );
  
  // Also log to console in development
  if (process.env.NODE_ENV !== 'production') {
    console.log('🔒 Audit:', logEntry);
  }
}
```

### Use in authentication routes
```javascript
// In server/routes/auth.js
import { logAuditEvent } from '../utils/logger.js';

router.post('/login', async (req, res) => {
  const { username, password } = req.body;
  
  try {
    const user = userDb.getUserByUsername(username);
    
    if (!user) {
      logAuditEvent({
        event: 'login_failed',
        username,
        reason: 'user_not_found',
        ip: req.ip
      });
      return res.status(401).json({ error: 'Invalid username or password' });
    }
    
    const isValidPassword = await bcrypt.compare(password, user.password_hash);
    
    if (!isValidPassword) {
      logAuditEvent({
        event: 'login_failed',
        username,
        reason: 'invalid_password',
        ip: req.ip
      });
      return res.status(401).json({ error: 'Invalid username or password' });
    }
    
    logAuditEvent({
      event: 'login_success',
      username,
      ip: req.ip
    });
    
    // ... rest of login logic
  } catch (error) {
    // ...
  }
});
```

### Use in shell access
```javascript
// In server/index.js shell handler
function handleShellConnection(ws) {
  const userInfo = ws.upgradeReq?.user || { username: 'unknown' };
  
  logAuditEvent({
    event: 'shell_session_started',
    username: userInfo.username,
    timestamp: new Date().toISOString()
  });
  
  // ... rest of shell handler
  
  ws.on('close', () => {
    logAuditEvent({
      event: 'shell_session_ended',
      username: userInfo.username,
      timestamp: new Date().toISOString()
    });
  });
}
```

---

## 9. Update README with Security Warnings

### Add Security Section to README.md
```markdown
## 🔒 Security Considerations

### ⚠️ Important Security Notice

This application provides **authenticated shell access** and **file system operations** to users. Once authenticated, users have extensive system access equivalent to running commands in a terminal.

### Deployment Security

**For Personal/Local Use:**
- Use strong passwords for your user account
- Keep the application running on localhost
- Use a firewall to block external access

**For Remote/Production Deployments:**
- **REQUIRED:** Use HTTPS/TLS (set up a reverse proxy like Nginx or Caddy)
- **REQUIRED:** Use strong, randomly generated JWT_SECRET
- **REQUIRED:** Deploy behind VPN or SSH tunnel
- **RECOMMENDED:** Use Docker containers for isolation
- **RECOMMENDED:** Enable audit logging
- **RECOMMENDED:** Implement 2FA authentication
- **RECOMMENDED:** Regular security updates

### Network Configuration

By default, the server binds to `0.0.0.0` (all network interfaces). To restrict to localhost only:

```bash
# In .env file
BIND_ADDRESS=127.0.0.1
```

Then update `server/index.js`:
```javascript
const BIND_ADDRESS = process.env.BIND_ADDRESS || '0.0.0.0';
server.listen(PORT, BIND_ADDRESS, async () => {
  console.log(`Server running on http://${BIND_ADDRESS}:${PORT}`);
});
```

### Data Privacy

- Voice transcription features send audio data to OpenAI (disabled by default)
- All other features operate locally
- Session data is stored locally in SQLite database
- Gemini CLI communicates with Google's Gemini API according to your Gemini CLI configuration

### Regular Maintenance

1. Keep Node.js and npm packages up to date
2. Review audit logs regularly
3. Monitor for failed authentication attempts
4. Rotate JWT secrets periodically
5. Review and remove inactive user sessions
```

---

## 10. Add Security Testing Script

### Create tests/security-check.js
```javascript
import Database from 'better-sqlite3';
import path from 'path';
import { fileURLToPath } from 'url';
import crypto from 'crypto';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

console.log('🔒 Running Security Checks...\n');

// Check 1: JWT Secret
console.log('1. Checking JWT Secret...');
if (!process.env.JWT_SECRET) {
  console.log('   ❌ JWT_SECRET is not set');
} else if (process.env.JWT_SECRET === 'claude-ui-dev-secret-change-in-production') {
  console.log('   ❌ Using default JWT_SECRET (insecure)');
} else if (process.env.JWT_SECRET.length < 32) {
  console.log('   ⚠️  JWT_SECRET is too short (should be at least 32 characters)');
} else {
  console.log('   ✅ JWT_SECRET is configured');
}

// Check 2: Node Environment
console.log('\n2. Checking Node Environment...');
if (process.env.NODE_ENV === 'production') {
  console.log('   ✅ Running in production mode');
} else {
  console.log('   ⚠️  Running in development mode');
}

// Check 3: OpenAI Integration
console.log('\n3. Checking OpenAI Integration...');
if (process.env.ENABLE_VOICE_FEATURES === 'true') {
  console.log('   ⚠️  Voice features enabled (data sent to OpenAI)');
  if (!process.env.OPENAI_API_KEY) {
    console.log('   ❌ OPENAI_API_KEY not set but features enabled');
  }
} else {
  console.log('   ✅ Voice features disabled');
}

// Check 4: Binding Address
console.log('\n4. Checking Network Binding...');
const bindAddress = process.env.BIND_ADDRESS || '0.0.0.0';
if (bindAddress === '0.0.0.0') {
  console.log('   ⚠️  Server binds to all interfaces (0.0.0.0)');
  console.log('      Consider restricting to localhost in production');
} else if (bindAddress === '127.0.0.1' || bindAddress === 'localhost') {
  console.log('   ✅ Server restricted to localhost');
} else {
  console.log('   ℹ️  Server binds to:', bindAddress);
}

console.log('\n🔒 Security Check Complete\n');
```

### Add to package.json
```json
{
  "scripts": {
    "security-check": "node tests/security-check.js"
  }
}
```

---

## Implementation Priority

### Phase 1: Critical (Implement Immediately)
1. ✅ Document OpenAI data sharing in README
2. ✅ Enforce strong JWT_SECRET
3. ✅ Fix command injection in git routes
4. ✅ Implement path traversal prevention

### Phase 2: High (Implement Within 1 Week)
5. ✅ Add JWT token expiration
6. ✅ Add rate limiting
7. ✅ Add security headers
8. ✅ Add audit logging

### Phase 3: Medium (Implement Within 1 Month)
9. ✅ Make OpenAI integration optional with explicit consent
10. ✅ Add security documentation
11. ✅ Create security testing script
12. ✅ Add bind address configuration

### Phase 4: Future Enhancements
13. Add 2FA support
14. Implement CSRF protection
15. Add session management UI
16. Container/Docker security hardening

---

## Testing Security Fixes

After implementing fixes, test with:

```bash
# Run security check
npm run security-check

# Test authentication with wrong credentials
curl -X POST http://localhost:4008/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"wrong"}'

# Test rate limiting (should fail after 5 attempts)
for i in {1..10}; do
  curl -X POST http://localhost:4008/api/auth/login \
    -H "Content-Type: application/json" \
    -d '{"username":"test","password":"wrong"}'
  echo "Attempt $i"
done

# Test path traversal (should be blocked)
curl http://localhost:4008/api/projects/test/file?filePath=/etc/passwd \
  -H "Authorization: Bearer YOUR_TOKEN"
```

---

## Contact for Security Issues

If you discover a security vulnerability, please report it responsibly:

1. **Do not** open a public GitHub issue
2. Email security details to [maintainer email]
3. Include:
   - Description of the vulnerability
   - Steps to reproduce
   - Potential impact
   - Suggested fix (if any)

We will respond within 48 hours and provide a timeline for fixes.
