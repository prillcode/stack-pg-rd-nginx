# Usage Examples

This document provides detailed examples for using the Docker Dev Stack with your Angular and React projects.

## Table of Contents
- [PostgreSQL Examples](#postgresql-examples)
- [Redis Examples](#redis-examples)
- [Nginx Examples](#nginx-examples)

---

## PostgreSQL Examples

### Connecting from Node.js/Express Backend

Install the PostgreSQL client:
```bash
npm install pg
```

**Basic connection:**
```javascript
const { Pool } = require('pg');

const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'devdb',
  user: 'postgres',
  password: 'dev'
});

// Query example
const result = await pool.query('SELECT * FROM users WHERE id = $1', [userId]);
```

**Using environment variables (.env):**
```env
DATABASE_URL=postgresql://postgres:dev@localhost:5432/devdb
```

```javascript
const { Pool } = require('pg');
const pool = new Pool({
  connectionString: process.env.DATABASE_URL
});
```

### Connecting from NestJS

```typescript
// app.module.ts
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: 'localhost',
      port: 5432,
      username: 'postgres',
      password: 'dev',
      database: 'devdb',
      entities: [__dirname + '/**/*.entity{.ts,.js}'],
      synchronize: true, // Dev only!
    }),
  ],
})
export class AppModule {}
```

### Creating Tables and Seeding Data

Connect to the database:
```bash
docker exec -it local-postgres psql -U postgres -d devdb
```

Create a table:
```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO users (email, name) VALUES
  ('alice@example.com', 'Alice'),
  ('bob@example.com', 'Bob');
```

---

## Redis Examples

### Connecting from Node.js/Express Backend

Install the Redis client:
```bash
npm install redis
```

**Basic connection:**
```javascript
const redis = require('redis');
const client = redis.createClient({
  url: 'redis://localhost:6379'
});

await client.connect();

// Set and get
await client.set('key', 'value');
const value = await client.get('key');

// Set with expiration (10 seconds)
await client.setEx('session:123', 10, JSON.stringify({ userId: 1 }));
```

### Use Case 1: API Response Caching

```javascript
const express = require('express');
const redis = require('redis');
const client = redis.createClient({ url: 'redis://localhost:6379' });

await client.connect();

app.get('/api/users/:id', async (req, res) => {
  const { id } = req.params;
  const cacheKey = `user:${id}`;

  // Check cache first
  const cached = await client.get(cacheKey);
  if (cached) {
    return res.json(JSON.parse(cached));
  }

  // Cache miss - query database
  const user = await db.query('SELECT * FROM users WHERE id = $1', [id]);

  // Cache for 5 minutes
  await client.setEx(cacheKey, 300, JSON.stringify(user));

  res.json(user);
});
```

### Use Case 2: Session Storage (Express)

```bash
npm install express-session connect-redis
```

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const redis = require('redis');

const redisClient = redis.createClient({ url: 'redis://localhost:6379' });
await redisClient.connect();

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: 'your-secret-key',
  resave: false,
  saveUninitialized: false,
  cookie: { maxAge: 24 * 60 * 60 * 1000 } // 24 hours
}));
```

### Use Case 3: Rate Limiting

```javascript
async function checkRateLimit(userId, maxRequests = 10, windowSeconds = 60) {
  const key = `rate_limit:${userId}`;
  const current = await client.incr(key);

  if (current === 1) {
    await client.expire(key, windowSeconds);
  }

  return current <= maxRequests;
}

// Middleware
app.use(async (req, res, next) => {
  const allowed = await checkRateLimit(req.user.id);
  if (!allowed) {
    return res.status(429).json({ error: 'Too many requests' });
  }
  next();
});
```

### Use Case 4: Background Job Queue (Bull)

```bash
npm install bull
```

```javascript
const Queue = require('bull');

const emailQueue = new Queue('email', {
  redis: { host: 'localhost', port: 6379 }
});

// Add job to queue
app.post('/api/register', async (req, res) => {
  const user = await createUser(req.body);

  // Queue welcome email
  await emailQueue.add({ userId: user.id, type: 'welcome' });

  res.json({ success: true });
});

// Process jobs
emailQueue.process(async (job) => {
  const { userId, type } = job.data;
  await sendEmail(userId, type);
});
```

---

## Nginx Examples

### Example 1: Testing Angular Production Build

**From your Angular project:**

1. Build your app:
```bash
ng build --configuration production
```

2. Start nginx (Option 1 - from project directory):
```bash
cd /path/to/your/app
docker compose -f /path/to/stack-pg-rd-nginx/docker-compose.yml --profile production-test up nginx
```

3. Visit http://localhost:8080

### Example 2: Testing React Production Build

**From your React project:**

1. Build your app:
```bash
npm run build
```

2. Start nginx (Option 2 - using environment variable):
```bash
PROJECT_DIST=/path/to/your/react-app/build docker compose -f /path/to/stack-pg-rd-nginx/docker-compose.yml --profile production-test up nginx
```

3. Visit http://localhost:8080

### Example 3: Custom Nginx Configuration

If you need custom nginx config (e.g., for SPA routing), uncomment the config line in docker-compose.yml:

```yaml
nginx:
  volumes:
    - ./dist:/usr/share/nginx/html:ro
    - ./nginx.conf:/etc/nginx/nginx.conf:ro  # Uncomment this
```

Create `nginx.conf` in your project:
```nginx
events {
  worker_connections 1024;
}

http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # SPA routing - serve index.html for all routes
    location / {
      try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
      expires 1y;
      add_header Cache-Control "public, immutable";
    }
  }
}
```

---

## Full Workflow Example

**Scenario: React app with Express backend using PostgreSQL and Redis**

**Terminal 1 - Start dev stack:**
```bash
cd /path/to/stack-pg-rd-nginx
docker compose up -d
```

**Terminal 2 - Backend development:**
```bash
cd /path/to/my-app/backend
npm install
npm run dev  # Express server connects to localhost:5432 and localhost:6379
```

**Terminal 3 - Frontend development:**
```bash
cd /path/to/my-app/frontend
npm install
npm start  # React dev server, calls backend API
```

**Terminal 4 - Test production build (when ready):**
```bash
cd /path/to/my-app/frontend
npm run build
docker compose -f /home/prill/code/stack-pg-rd-nginx/docker-compose.yml --profile production-test up nginx
# Visit http://localhost:8080
```

---

## Troubleshooting

### PostgreSQL connection refused
- Ensure services are running: `docker compose ps`
- Check port isn't in use: `lsof -i :5432`

### Redis connection refused
- Ensure services are running: `docker compose ps`
- Check port isn't in use: `lsof -i :6379`

### Nginx shows empty directory
- Verify build output exists in your project's dist/build folder
- Check the path mapping is correct
- Ensure you're using the correct `-f` flag or `PROJECT_DIST` variable

### Data persistence
- Data is stored in Docker volumes and persists between restarts
- To reset: `docker compose down -v` (warning: deletes all data)
