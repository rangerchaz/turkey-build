# QA Agent

**Role:** Quality Assurance Turkey 🦃
**Phase:** 4/5 - Review (after initial build)

## Purpose

Validate that everything works correctly. The QA Agent ensures the code does what the spec said it would do.

## Responsibilities

1. **Unit Tests** - Test individual functions and methods
2. **Integration Tests** - Test API endpoints
3. **Feature Validation** - Verify acceptance criteria
4. **Edge Case Testing** - What happens when things go wrong?
5. **Regression Prevention** - Ensure fixes don't break other things
6. **Schema Sync Verification** - Ensure backend schemas match frontend types

## Testing Strategy

### 1. Unit Tests

Test business logic in isolation:

```ruby
# spec/services/session_detection_service_spec.rb
RSpec.describe SessionDetectionService do
  let(:user) { create(:user) }
  let(:service) { described_class.new(user) }
  
  describe '#process_heartbeat' do
    context 'with no existing session' do
      it 'creates a new session' do
        expect {
          service.process_heartbeat(
            project_name: 'my-project',
            branch: 'main',
            file_type: '.rb'
          )
        }.to change(Session, :count).by(1)
      end
      
      it 'creates a heartbeat' do
        expect {
          service.process_heartbeat(
            project_name: 'my-project',
            branch: 'main',
            file_type: '.rb'
          )
        }.to change(Heartbeat, :count).by(1)
      end
    end
    
    context 'with existing active session' do
      let!(:session) { create(:session, user: user, project_name: 'my-project') }
      
      before do
        create(:heartbeat, session: session, timestamp: 5.minutes.ago)
      end
      
      it 'joins existing session' do
        expect {
          service.process_heartbeat(
            project_name: 'my-project',
            branch: 'main',
            file_type: '.rb'
          )
        }.not_to change(Session, :count)
      end
    end
    
    context 'after 30 minute gap' do
      let!(:session) { create(:session, user: user, project_name: 'my-project') }
      
      before do
        create(:heartbeat, session: session, timestamp: 35.minutes.ago)
        session.update!(updated_at: 35.minutes.ago)
      end
      
      it 'creates new session' do
        expect {
          service.process_heartbeat(
            project_name: 'my-project',
            branch: 'main',
            file_type: '.rb'
          )
        }.to change(Session, :count).by(1)
      end
      
      it 'closes previous session' do
        service.process_heartbeat(
          project_name: 'my-project',
          branch: 'main',
          file_type: '.rb'
        )
        
        expect(session.reload.end_time).to be_present
      end
    end
  end
end
```

### 2. Integration Tests (API)

Test endpoints end-to-end:

```ruby
# spec/requests/api/heartbeats_spec.rb
RSpec.describe 'API Heartbeats', type: :request do
  let(:user) { create(:user) }
  let(:auth_headers) { auth_headers_for(user) }
  
  describe 'POST /api/heartbeats' do
    let(:valid_params) do
      {
        heartbeat: {
          project_name: 'my-project',
          branch: 'feature/new-thing',
          file_type: '.rb'
        }
      }
    end
    
    context 'with valid auth' do
      it 'returns 201' do
        post '/api/heartbeats', 
             params: valid_params, 
             headers: auth_headers
        
        expect(response).to have_http_status(:created)
      end
      
      it 'returns session info' do
        post '/api/heartbeats', 
             params: valid_params, 
             headers: auth_headers
        
        json = JSON.parse(response.body)
        expect(json['session_id']).to be_present
      end
    end
    
    context 'without auth' do
      it 'returns 401' do
        post '/api/heartbeats', params: valid_params
        
        expect(response).to have_http_status(:unauthorized)
      end
    end
    
    context 'with missing params' do
      it 'returns 422' do
        post '/api/heartbeats', 
             params: { heartbeat: { branch: 'main' } },
             headers: auth_headers
        
        expect(response).to have_http_status(:unprocessable_entity)
      end
    end
  end
  
  describe 'GET /api/current_session' do
    context 'with active session' do
      before do
        create(:session, user: user, end_time: nil)
      end
      
      it 'returns the session' do
        get '/api/current_session', headers: auth_headers
        
        expect(response).to have_http_status(:ok)
        json = JSON.parse(response.body)
        expect(json['id']).to be_present
      end
    end
    
    context 'without active session' do
      it 'returns 204' do
        get '/api/current_session', headers: auth_headers
        
        expect(response).to have_http_status(:no_content)
      end
    end
  end
end
```

### 3. Feature Validation

Map acceptance criteria to tests:

```ruby
# From spec: "30 minute gap creates new session"
RSpec.describe 'Session boundary detection' do
  it 'creates new session after 30 minute gap' do
    # Test the specific acceptance criterion
  end
end

# From spec: "Track git branch per heartbeat"
RSpec.describe 'Branch tracking' do
  it 'stores branch with each heartbeat' do
    heartbeat = service.process_heartbeat(branch: 'feature/x')
    expect(heartbeat[:heartbeat].branch).to eq('feature/x')
  end
end
```

### 4. Edge Cases

Test the unhappy paths:

```ruby
RSpec.describe 'Edge cases' do
  describe 'malformed input' do
    it 'handles empty strings' do
      result = service.process_heartbeat(project_name: '', branch: 'main')
      expect(result[:error]).to be_present
    end
    
    it 'handles nil values' do
      result = service.process_heartbeat(project_name: nil, branch: nil)
      expect(result[:error]).to be_present
    end
    
    it 'handles extremely long strings' do
      long_name = 'a' * 10_000
      result = service.process_heartbeat(project_name: long_name, branch: 'main')
      expect(result[:error]).to be_present
    end
  end
  
  describe 'concurrent requests' do
    it 'handles rapid heartbeats' do
      10.times do
        service.process_heartbeat(project_name: 'proj', branch: 'main')
      end
      
      expect(Session.count).to eq(1)
      expect(Heartbeat.count).to eq(10)
    end
  end
  
  describe 'database failures' do
    it 'handles connection errors gracefully' do
      allow(Session).to receive(:create!).and_raise(ActiveRecord::ConnectionNotEstablished)
      
      expect {
        service.process_heartbeat(project_name: 'proj', branch: 'main')
      }.to raise_error(ServiceError)
    end
  end
end
```

### 5. Schema Sync Verification

**CRITICAL:** Backend schemas must match frontend type definitions. Mismatches cause silent data loss.

This applies to all stacks:
- Python/Pydantic ↔ TypeScript
- Python/Django DRF ↔ TypeScript
- Go structs ↔ TypeScript
- Node.js/Express ↔ Frontend types

```python
# Example: Python/Pydantic schema sync tests
# tests/test_schema_sync.py

class TestSchemaSync:
    """Verify backend schemas include all fields frontend expects."""

    def test_response_has_required_fields(self):
        """Frontend expects these fields - schema must include them."""
        from app.schemas.user import UserResponse

        fields = UserResponse.model_fields
        required = ['id', 'email', 'name', 'preferences']
        for field in required:
            assert field in fields, f"UserResponse missing '{field}' - frontend will break!"

    def test_nested_response_has_items(self):
        """Frontend expects nested data - verify it's included."""
        from app.schemas.order import OrderResponse

        fields = OrderResponse.model_fields
        assert 'items' in fields, "OrderResponse missing 'items' - order details won't show!"

    def test_api_response_matches_schema(self, client, test_data):
        """Verify actual API response includes all schema fields."""
        response = client.get(f"/api/items/{test_data.id}")
        data = response.json()

        # Verify required fields present
        assert 'name' in data, "API response missing 'name'!"
        assert 'details' in data, "API response missing 'details'!"

        # Verify no placeholder values
        assert data.get('name') != 'Unknown', "Name is placeholder!"
        assert data.get('name') != '', "Name is empty!"


class TestNoPlaceholderData:
    """Verify no placeholder data leaks into API responses."""

    def test_no_unknown_values_in_response(self, client, test_data):
        """Data must have real values, not placeholders."""
        response = client.get(f"/api/items")
        data = response.json()

        for item in data.get('items', []):
            assert item['name'] != 'Unknown', "Found placeholder name!"
            assert item['name'] != 'TODO', "Found TODO placeholder!"
            assert item['name'] != '', "Found empty name!"

    def test_nested_objects_populated(self, client, test_data):
        """Nested objects must have actual data, not empty dicts."""
        response = client.get(f"/api/orders/{test_data.id}")
        data = response.json()

        # Items array should be populated
        assert 'items' in data, "Missing items array!"
        assert len(data['items']) > 0, "Items array is empty!"

        for item in data['items']:
            assert item.get('product'), "Item missing product data!"
            assert item['product'] != {}, "Product data is empty!"
```

```typescript
// Example: Frontend type checking tests
// src/__tests__/types.test.ts

import { UserResponse, OrderResponse } from '../types/api';

describe('Type Compatibility', () => {
  it('UserResponse should have all required fields', () => {
    // This test verifies TypeScript types match what we expect
    const mockUser: UserResponse = {
      id: 1,
      email: 'test@example.com',
      name: 'Test User',
      preferences: { theme: 'dark' },
    };

    expect(mockUser.preferences).toBeDefined();
  });

  it('OrderResponse should have nested items', () => {
    const mockOrder: OrderResponse = {
      id: 1,
      total: 99.99,
      items: [
        { id: 1, product_name: 'Widget', quantity: 2 }
      ],
    };

    // Verify items is an array with content
    expect(mockOrder.items).toBeInstanceOf(Array);
    expect(mockOrder.items.length).toBeGreaterThan(0);
    expect(mockOrder.items[0].product_name).not.toBe('Unknown');
  });
});
```

```go
// Example: Go struct validation tests
// handlers/handlers_test.go

func TestResponseStructure(t *testing.T) {
    // Verify JSON tags match frontend expectations
    user := User{
        ID:          1,
        Email:       "test@example.com",
        Preferences: []Preference{},
    }

    data, _ := json.Marshal(user)
    var result map[string]interface{}
    json.Unmarshal(data, &result)

    // Verify preferences is present (not omitted)
    if _, ok := result["preferences"]; !ok {
        t.Error("preferences field missing from JSON - check omitempty")
    }
}
```

### Schema Sync Checklist

Before completing QA phase:

- [ ] All API response fields are defined in backend schemas
- [ ] All frontend-expected fields exist in backend schemas
- [ ] Types match between backend and frontend
- [ ] No placeholder values in test responses ("Unknown", "TODO", etc.)
- [ ] Nested objects are fully populated
- [ ] Optional fields are marked optional in both places
- [ ] Arrays return actual items, not empty []
- [ ] Foreign key references are expanded (not just IDs)

## Test Categories by Priority

### P0 - Must Pass (Blocks Release)
- Authentication works
- Core feature happy paths
- Data integrity (no data loss)
- Security validations

### P1 - Should Pass
- Error handling
- Edge cases
- Performance within limits
- All API endpoints respond

### P2 - Nice to Pass
- UI edge cases
- Uncommon scenarios
- Performance optimization

## Prompt Pattern

```
You are the QA Agent validating the build.

Your job is to ensure everything works as specified.

Write tests for:

1. **Unit Tests**
   - All service objects
   - Model validations
   - Business logic methods
   
2. **Integration Tests**
   - Every API endpoint
   - Happy path AND error paths
   - Auth required vs public
   
3. **Feature Validation**
   - Each acceptance criterion becomes a test
   - If spec says X, there's a test proving X
   
4. **Edge Cases**
   - Empty inputs
   - Invalid data types
   - Extremely long strings
   - Concurrent operations
   - Network failures

Test Standards:
- Tests must be runnable
- Use factories/fixtures properly
- Clean up after each test
- Clear test names that describe behavior
- Test the behavior, not the implementation

Output complete test files that can run with `rspec` or equivalent.
```

## Validation Checklist

QA Agent validates against spec:

```yaml
feature: "Passive heartbeat tracking"
acceptance_criteria:
  - "Extension sends heartbeat every 2 minutes while coding"
  - "Heartbeat includes project name, branch, file type"
  - "No user interaction required"
  
tests_needed:
  - unit: HeartbeatService handles all required fields
  - integration: POST /api/heartbeats accepts valid payload
  - integration: POST /api/heartbeats rejects missing fields
  - e2e: Extension sends heartbeats without button click
```

## Memory Integration

QA Agent queries memory for:
- Common bugs in similar projects
- Test patterns that caught issues
- Edge cases that were missed before
- Flaky test patterns to avoid

## Output Files

```
spec/
├── factories/
│   ├── users.rb
│   ├── sessions.rb
│   └── heartbeats.rb
├── models/
│   ├── user_spec.rb
│   ├── session_spec.rb
│   └── heartbeat_spec.rb
├── services/
│   └── session_detection_service_spec.rb
├── requests/
│   └── api/
│       ├── heartbeats_spec.rb
│       └── sessions_spec.rb
└── spec_helper.rb
```

## Success Criteria

QA is complete when:
- [ ] All unit tests pass
- [ ] All integration tests pass
- [ ] Every acceptance criterion has a test
- [ ] Edge cases covered
- [ ] No flaky tests
- [ ] Test coverage > 80%
- [ ] **Schema sync verified (Pydantic ↔ TypeScript)**
- [ ] **No placeholder data in API responses**
- [ ] **Nested objects fully populated in tests**

## Bug Report Format

When QA finds issues:

```yaml
bug_report:
  severity: HIGH
  location: SessionDetectionService#process_heartbeat
  description: "30-minute gap logic uses updated_at instead of heartbeat timestamp"
  expected: "New session after 30 min since last heartbeat"
  actual: "New session after 30 min since session updated_at"
  test_that_fails: |
    it 'creates new session based on heartbeat timestamp, not session timestamp'
  suggested_fix: "Use session.heartbeats.maximum(:timestamp) for comparison"
```

## Anti-Patterns

**DON'T:**
- Write tests that always pass
- Test implementation details
- Leave tests that require manual setup
- Skip error path testing
- Write flaky tests

**DO:**
- Test behavior, not implementation
- Cover happy AND unhappy paths
- Use factories consistently
- Name tests descriptively
- Make tests independent
