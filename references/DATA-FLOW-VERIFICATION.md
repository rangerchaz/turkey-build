# Data Flow Verification

**When:** After Runtime Verification, before E2E Testing
**Goal:** Prove data flows correctly from database through API to UI
**Rule:** No "Unknown" or placeholder data in production

## Philosophy

Runtime verification proves the server starts. E2E proves buttons work. But neither proves that **actual data** flows through the system. The most insidious bugs are data flow breaks:

- Backend schemas silently drop fields not in the model
- Frontend types don't match API responses
- IDs are stored but never resolved back to full objects
- Placeholder data ("Unknown", "TODO") reaches the UI

**The only truth: trace a piece of data from source to screen.**

## What This Phase Catches

| Bug Type | Symptom | Root Cause |
|----------|---------|------------|
| Schema omission | API returns data, frontend shows nothing | Schema missing field |
| Type mismatch | Frontend expects array, gets null | Optional field not handled |
| Placeholder leak | UI shows "Unknown" or "TODO" | ID not resolved to full data |
| Silent drop | Some items missing from list | Data filtered or lost in transit |
| Serialization gap | Complex objects become {} | Custom serialization missing |

## Phase 1: Identify Data Paths

For every feature, map the complete data journey:

```yaml
data_paths:
  # Example: E-commerce product display
  - name: "Product listing"
    source: "Database (products table)"
    journey:
      1. products table → ORM query
      2. ORM model → serializer/schema
      3. schema → JSON response
      4. JSON → Frontend API client
      5. API response → component state
      6. state → UI render
    critical_fields:
      - name (must not be "Unknown")
      - price (must be number, not null)
      - image_url (must be valid URL)
      - category (must be populated)

  # Example: User profile
  - name: "User profile display"
    source: "Database (users table)"
    journey:
      1. users → ORM model
      2. model → UserResponse schema
      3. schema → JSON
      4. JSON → Frontend hook
      5. hook → Profile component
    critical_fields:
      - email
      - name
      - preferences (nested object)

  # Example: Related data (foreign keys)
  - name: "Order with items"
    source: "Database (orders + order_items)"
    journey:
      1. orders JOIN order_items → query
      2. results → nested serialization
      3. nested data → JSON response
      4. JSON → Frontend state
      5. state → OrderDetail component
    critical_fields:
      - order.items[] (must be populated array)
      - items[].product_name (must not be "Unknown")
      - items[].quantity (must be number > 0)
```

## Phase 2: Schema Sync Check

**CRITICAL:** Backend response schemas MUST match frontend type definitions.

### Python (Pydantic) ↔ TypeScript

```
procedure verify_schema_sync_pydantic():

  # 1. Extract Pydantic schema fields
  for each backend schema file (schemas/*.py):
    parse_pydantic_models()
    extract:
      - field names
      - field types
      - Optional vs required
      - nested models

  # 2. Extract TypeScript interface fields
  for each frontend type file (types/*.ts, api.ts):
    parse_typescript_interfaces()
    extract:
      - property names
      - property types
      - optional (?) markers
      - nested types

  # 3. Compare matching types
  for each (backend_model, frontend_interface) pair:
    compare_and_report()
```

### Django (DRF Serializers) ↔ TypeScript

```
procedure verify_schema_sync_django():

  # 1. Extract serializer fields
  for each serializer in serializers/*.py:
    parse_serializer_fields()
    check Meta.fields vs actual fields

  # 2. Compare with frontend types
  # Same process as above
```

### Go (structs) ↔ TypeScript

```
procedure verify_schema_sync_go():

  # 1. Extract struct fields with json tags
  for each struct with json tags:
    parse_json_field_names()
    check omitempty behavior

  # 2. Compare with frontend types
```

### Common Sync Issues

```yaml
sync_issues:
  - issue: "Field in frontend, not in backend"
    symptom: "Frontend property always undefined"
    fix: "Add field to backend schema/serializer"

  - issue: "Field in backend, not in frontend"
    symptom: "Data sent but never used"
    severity: "Low (wasteful but not broken)"

  - issue: "Optional in backend, required in frontend"
    symptom: "Crash when field is null"
    fix: "Align optionality or add null handling"

  - issue: "Different field names"
    symptom: "Data present but not mapped"
    fix: "Align names or add field mapping"

  - issue: "Nested object vs flat"
    symptom: "Cannot access nested properties"
    fix: "Align structure between backend and frontend"
```

## Phase 3: Data Lookup Pattern Check

Many bugs come from storing IDs but not having a way to resolve them back to data.

```
procedure verify_data_lookups():

  # Find all places where IDs are stored without full data
  id_storage_patterns:
    - "List[str]" or "string[]" of IDs
    - "List[int]" or "number[]" of IDs
    - "*_id" or "*Id" fields (foreign keys)
    - References stored as IDs only

  for each id_storage:
    # Check: Is there a lookup mechanism?
    has_lookup = find_one_of:
      - data_lookup: Dict[id, FullData]
      - .get(id) or [id] access to data dict
      - JOIN or eager load in database query
      - Separate API call to resolve
      - Inline expansion in serialization

    if not has_lookup:
      ERROR: "IDs stored in {location} but no way to resolve to full data"
      EXAMPLE: "deck: ['card-1', 'card-2'] but no card data available"

    # Check: Is the lookup populated?
    if has_lookup:
      verify:
        - Lookup is populated BEFORE IDs are used
        - Lookup includes ALL IDs that will be accessed
        - Lookup data includes all required display fields

common_lookup_bugs:
  - "IDs stored, lookup dict empty"
  - "Lookup populated but not serialized"
  - "Lookup not restored after deserialization"
  - "Lookup missing some IDs"
  - "Lookup has IDs but missing required fields"
```

## Phase 4: Placeholder Detection

Scan codebase for placeholder patterns that indicate incomplete data flow.

```
procedure detect_placeholders():

  placeholder_patterns:
    # Generic placeholders
    - '"Unknown"'
    - '"unknown"'
    - '"TODO"'
    - '"PLACEHOLDER"'
    - '"N/A"'
    - '"TBD"'

    # Language-specific defaults
    - 'name="Unknown"'          # Python
    - 'name: "Unknown"'         # TypeScript/JSON
    - '"name": "Unknown"'       # JSON
    - 'Name: "Unknown"'         # Go

    # Fallback patterns
    - '?? "Unknown"'            # TypeScript nullish
    - '|| "Unknown"'            # JavaScript OR
    - '.get("name", "Unknown")' # Python dict
    - 'or "Unknown"'            # Python
    - 'getOrDefault("Unknown")' # Java/Kotlin

  for each file in codebase:
    matches = search(file, placeholder_patterns)

    for match in matches:
      context = get_surrounding_code(match)

      if is_data_creation(context):
        ERROR: "Placeholder data created at {location}"
        ACTION: "Ensure actual data is available"

      elif is_display_fallback(context):
        WARN: "Placeholder used as UI fallback"
        ACTION: "Verify data is always populated upstream"

  # Also detect empty object patterns
  empty_patterns:
    - '= {}'                    # Empty object assignment
    - '= []'                    # Empty array where data expected
    - '= null'                  # Null where object expected
    - '= None'                  # Python None
    - '= nil'                   # Go/Ruby nil
```

## Phase 5: End-to-End Data Trace

Actually trace data from creation to display.

```
procedure trace_data_e2e():

  # 1. Create test data with KNOWN, UNIQUE values
  test_item = create_test_data({
    name: "Test-{uuid}",        # Unique, searchable
    description: "TraceTest",   # Identifiable
    price: 99.99,               # Specific number
    # ... other fields with known values
  })

  # 2. Call API endpoint that returns this data
  response = GET /api/items/{id}  # or list endpoint

  # 3. Verify data survives the journey
  verify_in_response(response):
    - item.name == "Test-{uuid}" (NOT "Unknown")
    - item.description == "TraceTest"
    - item.price == 99.99
    - all required fields present
    - nested objects populated

  # 4. If data is missing or wrong, trace backwards
  if data_missing_or_wrong:
    trace_backwards:
      step_1: Check database - is data stored correctly?
      step_2: Check ORM query - is data fetched?
      step_3: Check serializer - is field included?
      step_4: Check JSON output - is data in response?
      step_5: Check frontend parsing - is data extracted?

    identify: "Where in the chain did data get lost?"

  # 5. Verify in UI (if headless browser available)
  page = load_page_with_test_data()

  verify_in_dom:
    - element shows "Test-{uuid}" (NOT "Unknown")
    - price displays correctly
    - images load (if applicable)
```

## Phase 6: Response Structure Verification

Hit actual endpoints and verify response structure.

```
procedure verify_response_structure():

  for each api_endpoint:

    # 1. Get actual response
    response = call_endpoint_with_test_data()

    # 2. Verify against expected structure
    check_response(response):

      # No placeholder values
      - No field contains "Unknown"
      - No field contains "TODO"
      - No field contains "PLACEHOLDER"

      # Required fields present
      - All documented fields exist
      - Nested objects are populated
      - Arrays have items when data exists

      # Types correct
      - Numbers are numbers (not strings)
      - Booleans are booleans
      - Arrays are arrays (not null)
      - Nested objects have structure

    # 3. Report violations
    if violations:
      for v in violations:
        ERROR: "{v.path}: expected {v.expected}, got {v.actual}"
```

## Stack-Specific Guidance

### Python/FastAPI/Pydantic

```python
# Common issue: Pydantic drops fields not in schema
class UserResponse(BaseModel):
    id: int
    email: str
    # MISSING: preferences field
    # Frontend expects it, silently dropped!

# Fix: Include all frontend-expected fields
class UserResponse(BaseModel):
    id: int
    email: str
    preferences: dict = {}  # Now included
```

### Python/Django/DRF

```python
# Common issue: Serializer doesn't include related data
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ['id', 'total']
        # MISSING: items - frontend shows empty order!

# Fix: Include nested serializer
class OrderSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True, read_only=True)

    class Meta:
        model = Order
        fields = ['id', 'total', 'items']
```

### Node.js/Express

```javascript
// Common issue: Not including all fields in response
app.get('/api/user/:id', (req, res) => {
  const user = await User.findById(id);
  res.json({
    id: user.id,
    email: user.email
    // MISSING: preferences
  });
});

// Fix: Include all required fields
res.json({
  id: user.id,
  email: user.email,
  preferences: user.preferences || {}
});
```

### Go

```go
// Common issue: omitempty hides empty values
type User struct {
    ID          int    `json:"id"`
    Preferences []Pref `json:"preferences,omitempty"`
    // Empty array becomes missing field!
}

// Fix: Don't omitempty for required fields, or ensure populated
type User struct {
    ID          int    `json:"id"`
    Preferences []Pref `json:"preferences"` // Always included
}
```

## Integration with Build Flow

```
Runtime Verification
        │
        ▼
Data Flow Verification
  ├── Schema Sync Check
  ├── Placeholder Detection
  ├── Data Lookup Verification
  └── E2E Data Trace
        │
        ▼
   If failures: bugfix/* branch
        │
        ▼
E2E Browser Testing
```

## Prompt Template

```
You are the Data Flow Verification Agent.

Your job is to prove that REAL DATA flows through the system, not placeholders.

## Your Process

1. **Schema Sync Check**
   - Compare backend schemas to frontend types
   - Find fields that exist in one but not the other
   - Flag type mismatches

2. **Data Lookup Verification**
   - Find all ID storage patterns
   - Verify each has a lookup mechanism
   - Verify lookups are populated with real data

3. **Placeholder Detection**
   - Scan for "Unknown", "TODO", "PLACEHOLDER" strings
   - Flag any placeholder data creation
   - Warn on fallback patterns that hide missing data

4. **End-to-End Trace**
   - Create test data with KNOWN values
   - Trace through DB → Service → Schema → API → Frontend
   - Verify data survives each stage
   - Flag any silent drops

5. **Response Verification**
   - Call actual API endpoints
   - Verify response structure matches expected
   - Flag any placeholder values in responses

## Critical Rules

- "Unknown" in API response = FAIL
- Empty nested objects when data expected = FAIL
- Schema field mismatch = FAIL
- Missing lookup for stored IDs = FAIL

## Success Criteria

Data Flow verification passes when:
- [ ] Backend schemas have all frontend-expected fields
- [ ] Frontend types match API response structure
- [ ] No placeholder values created in data layer
- [ ] All ID storage has corresponding lookup mechanisms
- [ ] API responses contain real data, not placeholders
- [ ] Nested objects are fully populated
- [ ] Data survives entire journey from DB to UI
```

## Anti-Patterns

**DON'T:**
- Assume "compiles" means "data flows"
- Skip schema comparison
- Ignore placeholder fallbacks
- Trust empty arrays/objects are intentional
- Assume IDs will "just work"

**DO:**
- Trace actual data through the system
- Verify schema sync before testing
- Flag all placeholder patterns
- Test with real, known data values
- Verify at each stage of the journey
