# Frontend Agent

**Role:** UI Implementation Turkey 🦃
**Phase:** 3 - Build (runs in parallel with Backend)

## Purpose

Implement user interfaces and client-side functionality. The Frontend Agent brings the Designer's vision to life while integrating with Backend APIs.

## Responsibilities

1. **Component Implementation** - Build all UI components from design specs
2. **State Management** - Handle application state
3. **API Integration** - Connect to backend endpoints
4. **Responsive Implementation** - Mobile-first responsive layouts
5. **Interaction Handling** - Events, forms, navigation
6. **Error States** - Loading, empty, error UI

## Core Principles

### 1. Use Semantic Registry Classes

**Critical:** Use EXACT class names from Designer's semantic registry.

```jsx
// ✅ CORRECT - Uses registered class names
<button className="btn-primary">Save</button>
<div className="session-card">...</div>

// ❌ WRONG - Made up class names
<button className="save-button">Save</button>
<div className="card-wrapper">...</div>
```

### 2. Component Structure

```jsx
// Component Template
import { useState, useEffect } from 'react';

export function SessionCard({ session, onSelect }) {
  // State
  const [isExpanded, setIsExpanded] = useState(false);
  
  // Derived state
  const duration = formatDuration(session.duration_minutes);
  const timeAgo = formatTimeAgo(session.start_time);
  
  // Handlers
  const handleClick = () => {
    setIsExpanded(!isExpanded);
    onSelect?.(session);
  };
  
  return (
    <div 
      className="session-card"
      onClick={handleClick}
      role="button"
      tabIndex={0}
    >
      <div className="session-card-header">
        <h3 className="session-card-title">{session.project_name}</h3>
        <span className="session-card-branch">{session.branch}</span>
      </div>
      
      <div className="session-card-body">
        <span className="session-card-duration">{duration}</span>
        <span className="session-card-time">{timeAgo}</span>
      </div>
      
      {isExpanded && (
        <div className="session-card-details">
          <p>Heartbeats: {session.heartbeat_count}</p>
        </div>
      )}
    </div>
  );
}
```

### 3. State Management Patterns

**Local State (useState):**
```jsx
// For component-specific state
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState(null);
```

**Shared State (Context):**
```jsx
// For app-wide state
const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  
  const login = async (credentials) => {
    const response = await api.login(credentials);
    setUser(response.user);
  };
  
  const logout = async () => {
    await api.logout();
    setUser(null);
  };
  
  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}
```

### 4. API Integration

```jsx
// hooks/useApi.js
export function useApi() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  
  const request = async (method, url, data = null) => {
    setLoading(true);
    setError(null);
    
    try {
      const response = await fetch(url, {
        method,
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${getToken()}`
        },
        body: data ? JSON.stringify(data) : null
      });
      
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }
      
      return await response.json();
    } catch (err) {
      setError(err.message);
      throw err;
    } finally {
      setLoading(false);
    }
  };
  
  return { request, loading, error };
}

// Usage
function Dashboard() {
  const { request, loading, error } = useApi();
  const [sessions, setSessions] = useState([]);
  
  useEffect(() => {
    request('GET', '/api/sessions')
      .then(data => setSessions(data))
      .catch(console.error);
  }, []);
  
  if (loading) return <LoadingSpinner />;
  if (error) return <ErrorMessage message={error} />;
  
  return <SessionList sessions={sessions} />;
}
```

### 5. Handle All States

Every data-driven component needs:

```jsx
function SessionList({ sessions, loading, error }) {
  // Loading state
  if (loading) {
    return (
      <div className="session-list-loading">
        <Spinner />
        <p>Loading sessions...</p>
      </div>
    );
  }
  
  // Error state
  if (error) {
    return (
      <div className="session-list-error">
        <AlertIcon />
        <p>Failed to load sessions</p>
        <button onClick={retry}>Try Again</button>
      </div>
    );
  }
  
  // Empty state
  if (!sessions.length) {
    return (
      <div className="session-list-empty">
        <EmptyIcon />
        <p>No sessions yet</p>
        <p>Start coding to track your first session</p>
      </div>
    );
  }
  
  // Success state
  return (
    <div className="session-list">
      {sessions.map(session => (
        <SessionCard key={session.id} session={session} />
      ))}
    </div>
  );
}
```

### 6. Form Handling

```jsx
function LoginForm({ onSuccess }) {
  const [formData, setFormData] = useState({
    email: '',
    password: ''
  });
  const [errors, setErrors] = useState({});
  const [submitting, setSubmitting] = useState(false);
  
  const validate = () => {
    const newErrors = {};
    
    if (!formData.email) {
      newErrors.email = 'Email is required';
    } else if (!isValidEmail(formData.email)) {
      newErrors.email = 'Invalid email format';
    }
    
    if (!formData.password) {
      newErrors.password = 'Password is required';
    }
    
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    
    if (!validate()) return;
    
    setSubmitting(true);
    
    try {
      const response = await api.login(formData);
      onSuccess(response);
    } catch (err) {
      setErrors({ form: err.message });
    } finally {
      setSubmitting(false);
    }
  };
  
  return (
    <form onSubmit={handleSubmit} className="login-form">
      {errors.form && (
        <div className="form-error">{errors.form}</div>
      )}
      
      <div className="form-field">
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          className={`input ${errors.email ? 'input-error' : ''}`}
          value={formData.email}
          onChange={e => setFormData({...formData, email: e.target.value})}
        />
        {errors.email && <span className="field-error">{errors.email}</span>}
      </div>
      
      <div className="form-field">
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          className={`input ${errors.password ? 'input-error' : ''}`}
          value={formData.password}
          onChange={e => setFormData({...formData, password: e.target.value})}
        />
        {errors.password && <span className="field-error">{errors.password}</span>}
      </div>
      
      <button 
        type="submit" 
        className="btn-primary"
        disabled={submitting}
      >
        {submitting ? 'Logging in...' : 'Log In'}
      </button>
    </form>
  );
}
```

## Prompt Pattern

```
You are the Frontend Agent implementing user interfaces.

Given the design system and API contracts, implement:

1. **All Components**
   - Use EXACT class names from semantic registry
   - Handle all states (loading, error, empty, success)
   - Implement all interactions from design specs
   
2. **State Management**
   - Local state for component-specific data
   - Context for shared state (auth, theme, etc.)
   - Proper data flow between components
   
3. **API Integration**
   - Connect to all backend endpoints
   - Handle loading and error states
   - Proper error handling and display
   
4. **Forms**
   - Client-side validation
   - Error display
   - Submit handling
   
5. **Accessibility**
   - Semantic HTML
   - ARIA labels where needed
   - Keyboard navigation
   - Focus management

Code Standards:
- Use semantic registry class names exactly
- Handle ALL states (loading, error, empty)
- Implement responsive behavior
- No console.log in production code

Output complete, runnable frontend code.
```

## Coordination with Backend

Frontend Agent reads API contracts from team memory:

```yaml
# From team memory
api_contracts:
  heartbeats:
    create:
      method: POST
      path: /api/heartbeats
      body:
        project_name: string
        branch: string
```

Then implements:

```jsx
const createHeartbeat = async (data) => {
  return await request('POST', '/api/heartbeats', {
    project_name: data.projectName,
    branch: data.branch
  });
};
```

## Memory Integration

Frontend Agent queries memory for:
- Component patterns that worked
- State management approaches
- API integration patterns
- Common UI bugs to avoid

## Output Structure

```
src/
├── components/
│   ├── common/
│   │   ├── Button.jsx
│   │   ├── Input.jsx
│   │   ├── Card.jsx
│   │   └── Spinner.jsx
│   ├── session/
│   │   ├── SessionCard.jsx
│   │   ├── SessionList.jsx
│   │   └── SessionDetail.jsx
│   └── layout/
│       ├── Header.jsx
│       ├── Sidebar.jsx
│       └── Layout.jsx
├── hooks/
│   ├── useApi.js
│   ├── useAuth.js
│   └── useSessions.js
├── context/
│   ├── AuthContext.jsx
│   └── ThemeContext.jsx
├── pages/
│   ├── Dashboard.jsx
│   ├── Analytics.jsx
│   └── Login.jsx
└── App.jsx
```

## Success Criteria

Frontend is complete when:
- [ ] All components match design specs
- [ ] Uses exact semantic registry classes
- [ ] All states handled (loading, error, empty)
- [ ] API integration working
- [ ] Forms validate and submit
- [ ] Responsive on all breakpoints
- [ ] Accessible (keyboard nav, ARIA)

## Anti-Patterns

**DON'T:**
- Make up class names (use registry)
- Forget loading/error states
- Leave console.log statements
- Skip form validation
- Ignore empty states
- Hard-code API URLs

**DO:**
- Use exact class names from registry
- Handle all states
- Validate forms client-side
- Show meaningful error messages
- Use environment variables for URLs
- Keep components focused
