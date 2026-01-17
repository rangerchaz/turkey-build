# DevOps Agent

**Role:** Infrastructure & Deployment Turkey 🦃
**Phase:** 3+ (parallel with build, finalizes after QA)

## Purpose

Handle deployment, containerization, CI/CD, and infrastructure. The DevOps Agent ensures the code doesn't just run locally—it runs in production.

## Responsibilities

1. **Containerization** - Dockerfile, docker-compose
2. **CI/CD Pipeline** - GitHub Actions, automated testing
3. **Environment Config** - Environment variables, secrets management
4. **Deployment Config** - Platform-specific (Heroku, AWS, Vercel, etc.)
5. **Infrastructure** - Database setup, Redis, file storage
6. **Monitoring** - Logging, health checks, error tracking

## Containerization

### Dockerfile (Rails Example)

```dockerfile
# Dockerfile
FROM ruby:3.2-slim

# Install dependencies
RUN apt-get update -qq && \
    apt-get install -y build-essential libpq-dev nodejs npm && \
    rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Install gems
COPY Gemfile Gemfile.lock ./
RUN bundle config set --local deployment 'true' && \
    bundle config set --local without 'development test' && \
    bundle install

# Copy application
COPY . .

# Precompile assets
RUN SECRET_KEY_BASE=dummy bundle exec rails assets:precompile

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Start server
CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]
```

### Dockerfile (Node Example)

```dockerfile
# Dockerfile
FROM node:20-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy application
COPY . .

# Build if needed
RUN npm run build --if-present

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/app_production
      - REDIS_URL=redis://redis:6379/0
      - RAILS_ENV=production
      - SECRET_KEY_BASE=${SECRET_KEY_BASE}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=app_production
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

## CI/CD Pipeline

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
          
      - name: Setup database
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
          RAILS_ENV: test
        run: |
          bundle exec rails db:create
          bundle exec rails db:schema:load
          
      - name: Run tests
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
          RAILS_ENV: test
        run: bundle exec rspec
        
      - name: Run linter
        run: bundle exec rubocop

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: docker build -t app:${{ github.sha }} .
        
      - name: Push to registry
        run: |
          echo ${{ secrets.REGISTRY_PASSWORD }} | docker login -u ${{ secrets.REGISTRY_USERNAME }} --password-stdin
          docker tag app:${{ github.sha }} registry.example.com/app:${{ github.sha }}
          docker push registry.example.com/app:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Deploy to production
        run: |
          # Platform-specific deployment
          curl -X POST ${{ secrets.DEPLOY_WEBHOOK }}
```

## Environment Configuration

### Environment Variables Template

```bash
# .env.example (NEVER commit actual .env)

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/app_development

# Redis (for caching, sessions, background jobs)
REDIS_URL=redis://localhost:6379/0

# Rails
SECRET_KEY_BASE=generate-with-rails-secret
RAILS_ENV=development

# Authentication
JWT_SECRET=generate-a-secure-secret

# External Services
# STRIPE_API_KEY=sk_test_...
# SENDGRID_API_KEY=SG...
# AWS_ACCESS_KEY_ID=...
# AWS_SECRET_ACCESS_KEY=...
# AWS_BUCKET=...

# Monitoring
# SENTRY_DSN=https://...
# NEW_RELIC_LICENSE_KEY=...
```

### Secrets Management

```ruby
# config/credentials.yml.enc (encrypted)
# Edit with: rails credentials:edit

production:
  secret_key_base: xxx
  database:
    password: xxx
  stripe:
    api_key: xxx
  aws:
    access_key_id: xxx
    secret_access_key: xxx
```

## Platform-Specific Configs

### Heroku

```yaml
# app.json
{
  "name": "ADHD Focus Tracker",
  "description": "Passive hyperfocus tracking",
  "buildpacks": [
    { "url": "heroku/ruby" }
  ],
  "addons": [
    "heroku-postgresql:mini",
    "heroku-redis:mini"
  ],
  "env": {
    "RAILS_ENV": "production",
    "RACK_ENV": "production",
    "SECRET_KEY_BASE": { "generator": "secret" }
  }
}
```

```
# Procfile
web: bundle exec puma -C config/puma.rb
worker: bundle exec sidekiq
release: bundle exec rails db:migrate
```

### Railway / Render

```yaml
# render.yaml
services:
  - type: web
    name: app
    env: ruby
    buildCommand: bundle install && bundle exec rails assets:precompile
    startCommand: bundle exec puma -C config/puma.rb
    envVars:
      - key: DATABASE_URL
        fromDatabase:
          name: db
          property: connectionString
      - key: RAILS_MASTER_KEY
        sync: false

databases:
  - name: db
    plan: starter
```

### Vercel (Frontend)

```json
// vercel.json
{
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "framework": "vite",
  "rewrites": [
    { "source": "/api/:path*", "destination": "https://api.example.com/:path*" }
  ]
}
```

## Health Checks

```ruby
# app/controllers/health_controller.rb
class HealthController < ApplicationController
  skip_before_action :authenticate_user!
  
  def show
    checks = {
      database: check_database,
      redis: check_redis,
      disk: check_disk_space
    }
    
    status = checks.values.all? ? :ok : :service_unavailable
    
    render json: {
      status: status == :ok ? 'healthy' : 'unhealthy',
      checks: checks,
      timestamp: Time.current.iso8601
    }, status: status
  end
  
  private
  
  def check_database
    ActiveRecord::Base.connection.execute('SELECT 1')
    true
  rescue
    false
  end
  
  def check_redis
    Redis.current.ping == 'PONG'
  rescue
    false
  end
  
  def check_disk_space
    # At least 100MB free
    `df -m /`.split("\n")[1].split[3].to_i > 100
  rescue
    false
  end
end
```

## Prompt Pattern

```
You are the DevOps Agent handling deployment and infrastructure.

Given the project, create:

1. **Containerization**
   - Dockerfile optimized for the stack
   - docker-compose for local development
   - Multi-stage builds where appropriate
   
2. **CI/CD Pipeline**
   - GitHub Actions workflow
   - Test → Build → Deploy stages
   - Environment-specific deployments
   
3. **Environment Configuration**
   - .env.example with all needed variables
   - Secrets management approach
   - Platform-specific configs
   
4. **Health Checks**
   - /health endpoint
   - Database connectivity check
   - External service checks
   
5. **Deployment Configs**
   - Platform-appropriate files (Procfile, app.json, etc.)
   - Database migration strategy
   - Zero-downtime deployment

Standards:
- Never commit secrets
- Use multi-stage Docker builds
- Include health checks
- Automate everything possible
- Document deployment process

Output complete, deployable configuration files.
```

## Output Files

```
/
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .github/
│   └── workflows/
│       └── ci.yml
├── .env.example
├── Procfile (if Heroku)
├── app.json (if Heroku)
├── render.yaml (if Render)
├── vercel.json (if Vercel)
└── DEPLOYMENT.md
```

## Success Criteria

DevOps is complete when:
- [ ] Docker builds without errors
- [ ] docker-compose up starts all services
- [ ] CI pipeline passes
- [ ] Health check endpoint works
- [ ] Deployment configs match target platform
- [ ] No secrets in code
- [ ] Deployment documented

## Anti-Patterns

**DON'T:**
- Commit .env files
- Hard-code secrets
- Skip health checks
- Use `latest` tags in production
- Forget database migrations in deploy

**DO:**
- Use .env.example
- Use secrets management
- Include health endpoints
- Pin dependency versions
- Automate migrations
