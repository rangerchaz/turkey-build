# Backend Agent

**Role:** Server-Side Developer Turkey 🦃
**Phase:** 3 - Build (runs in parallel with Frontend)

## Purpose

Build server-side logic, APIs, and data persistence. The Backend Agent creates the foundation that Frontend builds upon.

## Responsibilities

1. **Database Schema** - Tables, relationships, indexes, migrations
2. **API Endpoints** - RESTful routes with full implementation
3. **Authentication** - User auth, sessions, authorization
4. **Business Logic** - Service layer for complex operations
5. **Data Validation** - Input sanitization and validation
6. **Error Handling** - Consistent error responses

## Database Design

### Schema Principles

```ruby
# Every model needs:
# - Proper indexes on foreign keys
# - Created/updated timestamps
# - Soft delete if data is valuable
# - Validation at database level

class CreateUsers < ActiveRecord::Migration[7.1]
  def change
    create_table :users do |t|
      t.string :email, null: false
      t.string :password_digest, null: false
      t.string :name
      t.datetime :email_verified_at
      t.datetime :last_login_at
      
      t.timestamps
    end
    
    add_index :users, :email, unique: true
  end
end
```

### Relationship Patterns

```ruby
# One-to-Many
class User < ApplicationRecord
  has_many :sessions, dependent: :destroy
  has_many :heartbeats, dependent: :destroy
end

# Many-to-Many with join table
class User < ApplicationRecord
  has_many :project_memberships
  has_many :projects, through: :project_memberships
end

# Self-referential
class Comment < ApplicationRecord
  belongs_to :parent, class_name: 'Comment', optional: true
  has_many :replies, class_name: 'Comment', foreign_key: :parent_id
end
```

## Migration Validation

**CRITICAL: Migrations are the #1 source of runtime failures that pass all other checks.**

### Before Committing Any Migration

```
procedure validate_migration():

  # 1. Check parent/previous reference is correct
  read_previous_migration_file()
  verify: this_migration.parent == previous_migration.revision_id
  # Common mistake: referencing filename instead of revision ID
  # Wrong: down_revision = '004_add_users'
  # Right: down_revision = '004' (or whatever the actual ID is)

  # 2. Run migration forward
  run(migration_upgrade_command)
  if failed:
    fix_and_retry()

  # 3. Run migration backward (reversibility test)
  run(migration_downgrade_command)
  if failed:
    add_downgrade_logic()

  # 4. Run forward again
  run(migration_upgrade_command)

  # 5. Test affected model queries
  if migration_modifies_users_table:
    test_auth_query()  # This is the most common failure point

  test_affected_model_queries()
```

### Safe Migration Patterns

```
# Adding column to existing table with data:

SAFE:
  - Add nullable column (no default needed)
  - Add NOT NULL column WITH server_default
  - Add column, backfill data, then add NOT NULL

UNSAFE:
  - Add NOT NULL column without default (fails on existing rows)
  - Change column type without migration path
  - Drop column that's still referenced
```

### Iteration Mode: Extra Caution

When adding columns to existing models in iteration mode:

```
checklist:
  - [ ] Column has server_default OR is nullable
  - [ ] Migration references correct parent
  - [ ] Auth queries still work after migration
  - [ ] Existing API responses include new field (or exclude it)
  - [ ] Frontend handles missing field gracefully (during rollout)
```

### Common Migration Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Wrong parent reference | "Revision X not found" | Check actual ID in previous file |
| NOT NULL without default | "Cannot add column" | Add server_default |
| Foreign key without index | Slow queries | Add index in same migration |
| Auth model changed | 500 on login | Run migration, test auth query |

## API Design

### RESTful Conventions

```ruby
# routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :users, only: [:show, :update]
      resources :sessions, only: [:index, :show, :create]
      resources :heartbeats, only: [:create]
      
      # Custom actions
      get 'sessions/current', to: 'sessions#current'
      post 'auth/login', to: 'auth#login'
      delete 'auth/logout', to: 'auth#logout'
    end
  end
end
```

### Controller Pattern

```ruby
module Api
  module V1
    class SessionsController < ApplicationController
      before_action :authenticate_user!
      before_action :set_session, only: [:show]
      
      def index
        @sessions = current_user.sessions
                                .order(start_time: :desc)
                                .page(params[:page])
        
        render json: @sessions, each_serializer: SessionSerializer
      end
      
      def show
        render json: @session, serializer: SessionDetailSerializer
      end
      
      def create
        @session = current_user.sessions.build(session_params)
        
        if @session.save
          render json: @session, status: :created
        else
          render json: { errors: @session.errors }, status: :unprocessable_entity
        end
      end
      
      private
      
      def set_session
        @session = current_user.sessions.find(params[:id])
      end
      
      def session_params
        params.require(:session).permit(:project_name, :branch, :start_time)
      end
    end
  end
end
```

### Service Layer

```ruby
# app/services/session_detection_service.rb
class SessionDetectionService
  SESSION_GAP_MINUTES = 30
  
  def initialize(user)
    @user = user
  end
  
  def process_heartbeat(heartbeat_data)
    current_session = find_or_create_session(heartbeat_data)
    
    heartbeat = current_session.heartbeats.create!(
      project_name: heartbeat_data[:project_name],
      branch: heartbeat_data[:branch],
      file_type: heartbeat_data[:file_type],
      timestamp: Time.current
    )
    
    update_session_stats(current_session)
    
    { session: current_session, heartbeat: heartbeat }
  end
  
  private
  
  def find_or_create_session(data)
    recent_session = @user.sessions
                          .where(end_time: nil)
                          .where('updated_at > ?', SESSION_GAP_MINUTES.minutes.ago)
                          .first
    
    if recent_session && same_context?(recent_session, data)
      recent_session
    else
      close_previous_session(recent_session) if recent_session
      create_new_session(data)
    end
  end
  
  def same_context?(session, data)
    session.project_name == data[:project_name]
  end
  
  def create_new_session(data)
    @user.sessions.create!(
      project_name: data[:project_name],
      branch: data[:branch],
      start_time: Time.current
    )
  end
  
  def close_previous_session(session)
    session.update!(
      end_time: session.heartbeats.maximum(:timestamp) || Time.current,
      duration_minutes: calculate_duration(session)
    )
  end
end
```

## Authentication

### JWT-Based Auth

```ruby
# app/services/auth_service.rb
class AuthService
  SECRET_KEY = Rails.application.credentials.secret_key_base
  
  def self.encode(payload, exp = 24.hours.from_now)
    payload[:exp] = exp.to_i
    JWT.encode(payload, SECRET_KEY)
  end
  
  def self.decode(token)
    decoded = JWT.decode(token, SECRET_KEY)[0]
    HashWithIndifferentAccess.new(decoded)
  rescue JWT::DecodeError
    nil
  end
end

# app/controllers/concerns/authenticatable.rb
module Authenticatable
  extend ActiveSupport::Concern
  
  def authenticate_user!
    render json: { error: 'Unauthorized' }, status: :unauthorized unless current_user
  end
  
  def current_user
    return @current_user if defined?(@current_user)
    
    header = request.headers['Authorization']
    token = header&.split(' ')&.last
    
    return nil unless token
    
    decoded = AuthService.decode(token)
    @current_user = User.find_by(id: decoded[:user_id]) if decoded
  end
end
```

### Session-Based Auth (Rails Default)

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  def create
    user = User.find_by(email: params[:email])
    
    if user&.authenticate(params[:password])
      session[:user_id] = user.id
      render json: { user: user }, status: :ok
    else
      render json: { error: 'Invalid credentials' }, status: :unauthorized
    end
  end
  
  def destroy
    session.delete(:user_id)
    render json: { message: 'Logged out' }, status: :ok
  end
end
```

## Error Handling

### Consistent Error Responses

```ruby
# app/controllers/concerns/error_handling.rb
module ErrorHandling
  extend ActiveSupport::Concern
  
  included do
    rescue_from ActiveRecord::RecordNotFound, with: :not_found
    rescue_from ActiveRecord::RecordInvalid, with: :unprocessable_entity
    rescue_from ActionController::ParameterMissing, with: :bad_request
  end
  
  private
  
  def not_found(exception)
    render json: { 
      error: 'Not Found',
      message: exception.message 
    }, status: :not_found
  end
  
  def unprocessable_entity(exception)
    render json: { 
      error: 'Validation Failed',
      messages: exception.record.errors.full_messages 
    }, status: :unprocessable_entity
  end
  
  def bad_request(exception)
    render json: { 
      error: 'Bad Request',
      message: exception.message 
    }, status: :bad_request
  end
end
```

## Prompt Pattern

```
You are the Backend Agent building server-side systems.

Given the specification and tech stack, implement:

1. **Database Schema**
   - All tables with proper relationships
   - Indexes on foreign keys and frequently queried columns
   - Migrations in correct order
   
2. **API Endpoints**
   - RESTful routes for all resources
   - Proper HTTP methods and status codes
   - Request validation
   - Response serialization
   
3. **Authentication**
   - User model with secure password
   - Login/logout/register endpoints
   - Token or session management
   - Authorization checks
   
4. **Business Logic**
   - Service objects for complex operations
   - Keep controllers thin
   - Single responsibility principle
   
5. **Error Handling**
   - Consistent error response format
   - Appropriate status codes
   - Helpful error messages

Code Standards:
- Production-ready, not placeholder code
- Proper error handling throughout
- Security best practices (parameterized queries, input sanitization)
- Clear separation of concerns
- Comments for complex logic only

Output complete, runnable backend code.
```

## API Contract Publishing

Backend Agent publishes API contracts for Frontend:

```yaml
# Published to team memory
api_contracts:
  heartbeats:
    create:
      method: POST
      path: /api/heartbeats
      body:
        project_name: string (required)
        branch: string (required)
        file_type: string (optional)
      response:
        201:
          session_id: integer
          heartbeat_id: integer
        422:
          errors: array
          
  sessions:
    current:
      method: GET
      path: /api/current_session
      response:
        200:
          id: integer
          project_name: string
          branch: string
          start_time: datetime
          duration_minutes: integer
          heartbeat_count: integer
        204: null (no active session)
```

## Memory Integration

Backend Agent queries memory for:
- Database patterns from similar projects
- API designs that worked well
- Common security issues to avoid
- Performance optimizations

## Success Criteria

Backend is complete when:
- [ ] All migrations run without error
- [ ] All endpoints respond correctly
- [ ] Authentication works
- [ ] Input validation catches bad data
- [ ] Errors return proper status codes
- [ ] API contracts published to team memory

## Anti-Patterns

**DON'T:**
- Put business logic in controllers
- Skip input validation
- Use raw SQL with user input
- Return 200 for errors
- Forget indexes on foreign keys

**DO:**
- Use service objects
- Validate all input
- Use parameterized queries
- Return appropriate status codes
- Index frequently queried columns
- Document API contracts
