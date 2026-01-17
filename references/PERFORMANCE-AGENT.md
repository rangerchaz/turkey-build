# Performance Agent

**Role:** Optimization Turkey 🦃⚡
**Phase:** After build, during QA/Review

## Purpose

Ensure the application performs well under load and identify optimization opportunities. The Performance Agent finds slow queries, memory leaks, and bottlenecks before users do.

## Responsibilities

1. **Load Testing** - Can it handle expected traffic?
2. **Query Analysis** - N+1 queries, slow queries, missing indexes
3. **Memory Profiling** - Leaks, bloat, inefficient allocations
4. **Frontend Performance** - Bundle size, render time, Core Web Vitals
5. **API Response Times** - Endpoint latency analysis
6. **Optimization Recommendations** - Specific, actionable fixes

## Performance Standards

### Response Time Targets

| Endpoint Type | Target | Acceptable | Unacceptable |
|---------------|--------|------------|--------------|
| API Read (simple) | < 50ms | < 200ms | > 500ms |
| API Read (complex) | < 200ms | < 500ms | > 1s |
| API Write | < 100ms | < 300ms | > 1s |
| Page Load | < 1s | < 3s | > 5s |
| Time to Interactive | < 2s | < 5s | > 10s |

### Database Query Targets

| Query Type | Target | Flag If |
|------------|--------|---------|
| Simple SELECT | < 10ms | > 50ms |
| JOIN query | < 50ms | > 200ms |
| Aggregation | < 100ms | > 500ms |
| Any query | - | > 1s |

## Load Testing

### Using k6

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },   // Ramp up
    { duration: '1m', target: 20 },    // Stay at 20 users
    { duration: '30s', target: 50 },   // Ramp to 50
    { duration: '1m', target: 50 },    // Stay at 50
    { duration: '30s', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests < 500ms
    http_req_failed: ['rate<0.01'],    // < 1% error rate
  },
};

const BASE_URL = 'http://localhost:3000';

export default function () {
  // Simulate typical user flow
  
  // 1. Get dashboard
  const dashboard = http.get(`${BASE_URL}/api/sessions`, {
    headers: { Authorization: `Bearer ${__ENV.TOKEN}` },
  });
  check(dashboard, {
    'dashboard status 200': (r) => r.status === 200,
    'dashboard < 500ms': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
  
  // 2. Send heartbeat
  const heartbeat = http.post(
    `${BASE_URL}/api/heartbeats`,
    JSON.stringify({
      project_name: 'test-project',
      branch: 'main',
      file_type: '.js',
    }),
    {
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${__ENV.TOKEN}`,
      },
    }
  );
  check(heartbeat, {
    'heartbeat status 201': (r) => r.status === 201,
    'heartbeat < 200ms': (r) => r.timings.duration < 200,
  });
  
  sleep(1);
  
  // 3. Get analytics
  const analytics = http.get(`${BASE_URL}/api/analytics`, {
    headers: { Authorization: `Bearer ${__ENV.TOKEN}` },
  });
  check(analytics, {
    'analytics status 200': (r) => r.status === 200,
    'analytics < 1s': (r) => r.timings.duration < 1000,
  });
  
  sleep(2);
}
```

Run with:
```bash
k6 run --env TOKEN=your-token load-test.js
```

### Load Test Report

```markdown
# Load Test Results

## Summary
- **Total Requests**: 12,453
- **Failed Requests**: 23 (0.18%)
- **Avg Response Time**: 187ms
- **P95 Response Time**: 423ms
- **P99 Response Time**: 892ms

## By Endpoint

| Endpoint | Requests | Avg | P95 | P99 | Errors |
|----------|----------|-----|-----|-----|--------|
| GET /api/sessions | 4,151 | 156ms | 312ms | 567ms | 0.1% |
| POST /api/heartbeats | 4,150 | 89ms | 145ms | 234ms | 0.2% |
| GET /api/analytics | 4,152 | 315ms | 678ms | 1.2s | 0.3% |

## Issues Found

1. **Analytics endpoint slow under load**
   - P99 exceeds 1s target
   - Likely cause: Complex aggregation query
   
2. **Memory growth during test**
   - Started: 256MB
   - Ended: 512MB
   - Possible memory leak

## Recommendations

1. Add caching to analytics endpoint
2. Investigate memory leak
3. Add database indexes (see Query Analysis)
```

## Query Analysis

### Finding N+1 Queries

```ruby
# config/initializers/bullet.rb (development)
if defined?(Bullet)
  Bullet.enable = true
  Bullet.rails_logger = true
  Bullet.add_footer = true
  
  Bullet.bullet_logger = true
  Bullet.console = true
end
```

### Analyzing Slow Queries

```ruby
# config/initializers/query_analyzer.rb
if Rails.env.development?
  ActiveSupport::Notifications.subscribe('sql.active_record') do |*args|
    event = ActiveSupport::Notifications::Event.new(*args)
    
    if event.duration > 100 # ms
      Rails.logger.warn "SLOW QUERY (#{event.duration.round}ms): #{event.payload[:sql]}"
    end
  end
end
```

### Query Analysis Report

```markdown
# Query Analysis

## N+1 Queries Found

### Dashboard Controller
```ruby
# Problem
@sessions = current_user.sessions
@sessions.each { |s| s.heartbeats.count }  # N+1!

# Fix
@sessions = current_user.sessions.includes(:heartbeats)
```

### Sessions Serializer
```ruby
# Problem
def project_summary
  object.heartbeats.group(:file_type).count  # Query per session!
end

# Fix: Add counter cache or eager load
```

## Missing Indexes

| Table | Column | Query Frequency | Impact |
|-------|--------|-----------------|--------|
| heartbeats | session_id | High | Critical |
| sessions | user_id + start_time | High | High |
| heartbeats | timestamp | Medium | Medium |

### Add Indexes
```ruby
class AddPerformanceIndexes < ActiveRecord::Migration[7.1]
  def change
    add_index :heartbeats, :session_id
    add_index :sessions, [:user_id, :start_time]
    add_index :heartbeats, :timestamp
  end
end
```

## Slow Queries

| Query | Avg Time | Frequency | Location |
|-------|----------|-----------|----------|
| Analytics aggregation | 450ms | 100/hour | AnalyticsController#index |
| Session history | 230ms | 500/hour | SessionsController#index |

### Analytics Query Optimization
```ruby
# Before (450ms)
Session.where(user: current_user)
       .group("DATE(start_time)")
       .sum(:duration_minutes)

# After (45ms) - Use materialized view or cache
Rails.cache.fetch("analytics:#{current_user.id}", expires_in: 5.minutes) do
  # ... query
end
```
```

## Frontend Performance

### Bundle Analysis

```bash
# For webpack
npx webpack-bundle-analyzer stats.json

# For Vite
npx vite-bundle-visualizer
```

### Core Web Vitals

```javascript
// Measure Core Web Vitals
import { getCLS, getFID, getLCP, getFCP, getTTFB } from 'web-vitals';

function sendToAnalytics(metric) {
  console.log(metric.name, metric.value);
}

getCLS(sendToAnalytics);   // Cumulative Layout Shift
getFID(sendToAnalytics);   // First Input Delay
getLCP(sendToAnalytics);   // Largest Contentful Paint
getFCP(sendToAnalytics);   // First Contentful Paint
getTTFB(sendToAnalytics);  // Time to First Byte
```

### Frontend Performance Report

```markdown
# Frontend Performance

## Bundle Size
- **Total**: 245KB (gzipped)
- **Main**: 180KB
- **Vendor**: 65KB

| Module | Size | % of Bundle |
|--------|------|-------------|
| react | 42KB | 17% |
| chart.js | 68KB | 28% |
| lodash | 24KB | 10% |

### Recommendations
1. **Use lodash-es** - Import only needed functions
2. **Lazy load charts** - Don't load until needed
3. **Tree-shake chart.js** - Only import used chart types

## Core Web Vitals

| Metric | Value | Target | Status |
|--------|-------|--------|--------|
| LCP | 1.8s | < 2.5s | ✅ Good |
| FID | 45ms | < 100ms | ✅ Good |
| CLS | 0.15 | < 0.1 | ⚠️ Needs Work |

### CLS Issue
- Layout shift when sessions load
- Fix: Add skeleton loaders with fixed heights
```

## Memory Profiling

### Rails Memory

```ruby
# Add to Gemfile
gem 'derailed_benchmarks'
gem 'memory_profiler'

# Profile memory
$ bundle exec derailed exec perf:mem
```

### Node Memory

```javascript
// Add memory tracking
const used = process.memoryUsage();
console.log({
  heapTotal: `${Math.round(used.heapTotal / 1024 / 1024)} MB`,
  heapUsed: `${Math.round(used.heapUsed / 1024 / 1024)} MB`,
  external: `${Math.round(used.external / 1024 / 1024)} MB`,
});
```

## Prompt Pattern

```
You are the Performance Agent optimizing application performance.

Your job is to:
1. Run load tests and identify breaking points
2. Find N+1 queries and slow queries
3. Analyze frontend bundle and Core Web Vitals
4. Profile memory usage
5. Provide specific, actionable optimizations

Analysis Process:
1. Establish baseline metrics
2. Identify bottlenecks
3. Prioritize by impact
4. Provide specific fixes
5. Verify improvements

Performance Standards:
- API responses < 200ms (p95)
- Page load < 3s
- No N+1 queries
- Bundle size < 300KB gzipped
- Core Web Vitals all "Good"

Output:
- Load test results
- Query analysis with fixes
- Frontend performance report
- Memory profile
- Prioritized optimization list

Be specific. Include code fixes. Measure before and after.
```

## Performance Report Template

```markdown
# Performance Analysis Report

## Executive Summary
- Overall: ⚠️ NEEDS OPTIMIZATION
- Critical Issues: 2
- High Priority: 3
- Optimizations Identified: 8

## Critical Issues

### 1. N+1 Query in Dashboard
- **Impact**: 200ms → 2s with 100 sessions
- **Location**: DashboardController#index
- **Fix**: Add `.includes(:heartbeats)`

### 2. Analytics Query Timeout
- **Impact**: Timeout at 50 concurrent users
- **Location**: AnalyticsController#index
- **Fix**: Add caching, consider materialized view

## High Priority

1. Missing index on heartbeats.session_id
2. Bundle size 450KB (target: 300KB)
3. CLS 0.15 (target: < 0.1)

## Optimization Roadmap

| Priority | Issue | Fix | Estimated Impact |
|----------|-------|-----|------------------|
| 1 | N+1 query | Add includes | -80% response time |
| 2 | Missing index | Add migration | -50% query time |
| 3 | Analytics cache | Add Redis cache | -90% for cached |
| 4 | Bundle size | Tree-shake lodash | -60KB |
| 5 | CLS | Skeleton loaders | CLS → 0.05 |

## Before/After Targets

| Metric | Before | After | Target |
|--------|--------|-------|--------|
| Dashboard load | 450ms | 90ms | < 200ms |
| Analytics load | 1.2s | 120ms | < 500ms |
| Bundle size | 450KB | 280KB | < 300KB |
| Memory (1hr) | +300MB | +50MB | < 100MB |
```

## Success Criteria

Performance is acceptable when:
- [ ] All API endpoints < 200ms (p95)
- [ ] No N+1 queries
- [ ] All indexes present
- [ ] Bundle < 300KB gzipped
- [ ] Core Web Vitals all "Good"
- [ ] No memory leaks
- [ ] Handles 50 concurrent users

## Anti-Patterns

**DON'T:**
- Optimize without measuring
- Focus on micro-optimizations
- Ignore N+1 queries
- Skip load testing
- Cache without invalidation strategy

**DO:**
- Measure before optimizing
- Focus on biggest bottlenecks
- Fix N+1 queries immediately
- Load test realistic scenarios
- Plan cache invalidation
