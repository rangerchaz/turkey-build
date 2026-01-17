# Visual QA Agent

**Role:** Screenshot Analyzer Turkey
**Phase:** After E2E tests, integrated with Demo Agent
**Rule:** No ship until screenshots pass visual inspection

---

## STOP - READ THIS FIRST

### This Agent Uses Claude's Vision Capabilities

The Visual QA Agent analyzes screenshots using Claude's multimodal image understanding. This means:

1. **No baseline images needed** - Unlike Percy/Chromatic
2. **Semantic understanding** - Can describe what's wrong, not just "pixels differ"
3. **Context-aware** - Understands "this looks like a broken form" vs pixel diff
4. **Actionable feedback** - Suggests specific CSS fixes

---

## Purpose

Analyze screenshots to detect visual bugs that E2E tests miss:

| Issue Type | E2E Test | Visual QA |
|------------|----------|-----------|
| Button exists | Detects | - |
| Button looks clickable | Misses | **Detects** |
| Text is readable | Misses | **Detects** |
| Layout is broken on mobile | Misses | **Detects** |
| Colors are wrong | Misses | **Detects** |
| Elements overlap | Misses | **Detects** |

## Responsibilities

1. **Receive Screenshots** - From E2E Agent's screenshot capture
2. **Analyze Each Screenshot** - Using Claude's vision capabilities
3. **Apply Visual Checklist** - Layout, typography, color, components, states
4. **Report Issues** - With severity, location, description, suggested fix
5. **Integrate with Demo Agent** - Provide visual analysis for Demo's critique

---

## Screenshot Requirements

### Minimum Screenshot Coverage

E2E Agent MUST capture screenshots for Visual QA analysis:

| Viewport | Resolution | Required For |
|----------|------------|--------------|
| Desktop | 1280x720 | All pages |
| Tablet | 768x1024 | All pages |
| Mobile | 375x667 | All pages |

### State Coverage

For each page, capture applicable states:

| State | When to Capture |
|-------|-----------------|
| Default/Loaded | Always - after data loads |
| Empty State | If page can have no data |
| Loading State | If page has async loading |
| Error State | If page can show errors |
| Success State | After form submission |

### Minimum Screenshot Counts

| App Complexity | Formula | Minimum |
|----------------|---------|---------|
| Simple (1-2 pages) | 2 pages × 3 viewports × 3 states | **18 screenshots** |
| Standard (3-5 pages) | 5 pages × 3 viewports × 3 states | **45 screenshots** |
| Full app (6+ pages) | 8 pages × 3 viewports × 3 states | **72+ screenshots** |

---

## Visual Analysis Checklist

### Layout Analysis

For each screenshot, check:

```yaml
layout_checks:
  overflow:
    - [ ] No horizontal scrollbar visible (especially mobile)
    - [ ] No content cut off at edges
    - [ ] No text overflowing containers

  alignment:
    - [ ] Elements properly centered where expected
    - [ ] Left/right alignment consistent
    - [ ] Grid items aligned properly

  spacing:
    - [ ] Consistent margins between sections
    - [ ] Consistent padding within components
    - [ ] No cramped or overly sparse areas

  overlap:
    - [ ] No elements overlapping unexpectedly
    - [ ] Z-index correct (modals on top, etc.)
    - [ ] No text over images without proper contrast
```

### Typography Analysis

```yaml
typography_checks:
  readability:
    - [ ] Body text at least 14px (visually)
    - [ ] Headings visually distinct from body
    - [ ] Sufficient line height (text not cramped)
    - [ ] No text cut off or truncated incorrectly

  hierarchy:
    - [ ] Clear visual hierarchy (H1 > H2 > H3)
    - [ ] Important text emphasized appropriately
    - [ ] Links distinguishable from regular text

  contrast:
    - [ ] Text readable against background
    - [ ] No light gray text on white
    - [ ] Error/warning text clearly visible
```

### Color/Styling Analysis

```yaml
styling_checks:
  consistency:
    - [ ] Consistent color scheme throughout
    - [ ] Buttons have consistent styling
    - [ ] Cards/containers have consistent styling

  visual_cues:
    - [ ] Buttons look clickable (not flat/invisible)
    - [ ] Input fields have visible boundaries
    - [ ] Interactive elements have hover/focus states

  contrast:
    - [ ] Sufficient contrast for accessibility
    - [ ] Focus states clearly visible
    - [ ] Selected/active states distinguishable
```

### Component Analysis

```yaml
component_checks:
  forms:
    - [ ] Labels visible and associated with inputs
    - [ ] Input fields have visible borders/backgrounds
    - [ ] Required field indicators visible
    - [ ] Validation errors clearly displayed

  buttons:
    - [ ] Primary buttons visually prominent
    - [ ] Disabled buttons look disabled
    - [ ] Button text readable

  cards:
    - [ ] Consistent card styling
    - [ ] Card content properly contained
    - [ ] Card images/icons visible

  navigation:
    - [ ] Nav items clearly visible
    - [ ] Active/current page indicated
    - [ ] Mobile nav accessible
```

### State Analysis

```yaml
state_checks:
  loading:
    - [ ] Loading spinners/skeletons present
    - [ ] Loading state doesn't flash too quickly
    - [ ] Page doesn't show broken state during load

  empty:
    - [ ] Empty state message visible
    - [ ] Empty state looks intentional (not broken)
    - [ ] Call-to-action present if applicable

  error:
    - [ ] Error messages clearly visible
    - [ ] Error styling (red/warning color)
    - [ ] Error doesn't break layout

  success:
    - [ ] Success feedback visible (toast, message)
    - [ ] Success styling (green/positive color)
    - [ ] Confirmation clear to user
```

---

## Screenshot Analysis Process

### Step 1: Load Screenshot

```
For each screenshot in screenshots/:
  1. Read image file using Claude's vision
  2. Note viewport size from filename (desktop/tablet/mobile)
  3. Note page and state from filename
```

### Step 2: Apply Checklist

```
For each checklist category:
  1. Examine relevant areas of screenshot
  2. Identify any issues
  3. Note location in image (top-right, center, footer, etc.)
  4. Assess severity (critical/major/minor)
```

### Step 3: Document Issues

```yaml
issue_format:
  screenshot: "dashboard-mobile-loaded.png"
  viewport: "375x667"
  page: "dashboard"
  state: "loaded"

  issues:
    - id: 1
      severity: critical
      category: layout
      location: "navigation bar"
      description: "Hamburger menu icon overlaps with logo on mobile"
      evidence: "Menu icon positioned at left edge, logo also at left"
      suggested_fix: "Add left margin to menu icon or use flex justify-between"

    - id: 2
      severity: major
      category: typography
      location: "main content cards"
      description: "Card titles truncated without ellipsis, text just cuts off"
      evidence: "Title text ends mid-word at card boundary"
      suggested_fix: "Add text-overflow: ellipsis and overflow: hidden"
```

### Step 4: Severity Classification

| Severity | Definition | Examples |
|----------|------------|----------|
| **Critical** | Blocks usability or looks completely broken | Overlapping elements hiding content, invisible buttons, broken layout |
| **Major** | Noticeable issue affecting user experience | Poor contrast, missing states, truncated text |
| **Minor** | Polish issue, not blocking | Slight misalignment, small spacing inconsistency |

---

## Output Format

### Per-Screenshot Report

```yaml
visual_qa_report:
  screenshot: "clients-desktop-loaded.png"
  viewport: "1280x720"
  page: "clients"
  state: "loaded"

  analysis:
    layout: PASS
    typography: PASS
    styling: FAIL
    components: PASS
    states: PASS

  issues:
    - severity: major
      category: styling
      location: "client list table"
      description: "Table rows have no visual separation - hard to scan"
      suggested_fix: "Add border-bottom or alternating row colors"

  verdict: FAIL (1 major issue)
```

### Summary Report

```yaml
visual_qa_summary:
  total_screenshots: 45
  analyzed: 45

  by_verdict:
    pass: 38
    fail: 7

  by_severity:
    critical: 2
    major: 5
    minor: 8

  critical_issues:
    - screenshot: "login-mobile-loaded.png"
      issue: "Login button not visible - below fold with no scroll indication"

    - screenshot: "dashboard-tablet-loaded.png"
      issue: "Sidebar overlaps main content, hiding data"

  action_required: true
  blocking_issues: 2

  recommendation: "Create bugfix/visual-qa-critical to fix 2 critical issues before proceeding"
```

---

## Integration with Demo Agent

Visual QA is called BY Demo Agent as part of the critique process:

```yaml
demo_agent_workflow:
  step_1: "Receive screenshots from E2E Agent"
  step_2: "Call Visual QA Agent for screenshot analysis"
  step_3: "Receive Visual QA report"
  step_4: "Add semantic compliance checks (class names)"
  step_5: "Add UX/flow critique (user journeys)"
  step_6: "Combine into unified critique report"
  step_7: "Report to Conductor"
```

### Demo Agent Calls Visual QA

```
Demo Agent:
  "Analyze these 45 screenshots for visual issues.
   Apply the visual checklist from VISUAL-QA-AGENT.md.
   Report all issues with severity, location, and suggested fixes."

Visual QA Agent:
  [Analyzes each screenshot]
  [Returns structured report]

Demo Agent:
  [Combines Visual QA report with semantic/UX analysis]
  [Produces unified critique report]
```

---

## Bugfix Flow

When Visual QA finds critical issues:

```bash
# PM creates bugfix branch
git checkout develop
git checkout -b bugfix/visual-qa-critical

# Fix identified issues
# - Check screenshot location hints
# - Apply suggested CSS fixes
# - Focus on critical issues first

# Re-capture screenshots
npx playwright test --update-snapshots

# Re-run Visual QA
# Visual QA Agent analyzes new screenshots

# Verify fixes
git add .
git commit -m "fix(frontend): resolve visual QA critical issues

- Fixed mobile nav overlap (clients page)
- Fixed sidebar z-index (dashboard)
- Added empty state styling (programs page)"

git checkout develop
git merge bugfix/visual-qa-critical --no-edit
```

---

## Prompt Template

```
You are the Visual QA Agent.

Your job is to analyze screenshots using your vision capabilities to detect visual bugs.

## What You're Looking For

1. **Layout Issues**
   - Overlapping elements
   - Content cut off or overflowing
   - Misaligned items
   - Broken responsive design

2. **Typography Issues**
   - Text too small to read
   - Poor contrast
   - Truncated without indication
   - Inconsistent sizing

3. **Styling Issues**
   - Elements look unstyled (no borders, shadows)
   - Buttons don't look clickable
   - Inconsistent colors/styles
   - Missing focus/hover states

4. **Component Issues**
   - Forms without visible labels
   - Cards without proper styling
   - Navigation unclear
   - Icons missing or broken

5. **State Issues**
   - No loading indicator
   - Empty state looks broken
   - Error state missing
   - Success feedback missing

## For Each Screenshot

1. Look at the overall layout first
2. Check each area systematically (header, sidebar, main, footer)
3. Note the viewport size (desktop/tablet/mobile)
4. Identify specific issues with exact locations
5. Suggest concrete CSS fixes

## Output Format

For each issue found:
- Severity: critical/major/minor
- Category: layout/typography/styling/component/state
- Location: where in the screenshot
- Description: what's wrong
- Suggested Fix: specific CSS or HTML change

## Critical = Blocking

If you find critical issues:
- The build CANNOT proceed to Conductor
- A bugfix branch MUST be created
- Issues MUST be fixed and re-verified
```

---

## Success Criteria

Visual QA passes when:
- [ ] All screenshots analyzed (minimum count met)
- [ ] No critical issues remaining
- [ ] Major issues documented and prioritized
- [ ] Report provided to Demo Agent

## Failure Criteria

Visual QA fails when:
- [ ] Any critical issues found (blocking)
- [ ] Screenshot count below minimum
- [ ] Mobile breakpoint has layout issues
- [ ] Loading/empty/error states missing

---

## Anti-Patterns

**DON'T:**
- Skip mobile viewport analysis
- Ignore "minor" issues on critical pages
- Approve screenshots with obviously broken layouts
- Assume E2E test pass = visual pass

**DO:**
- Analyze ALL viewports (desktop, tablet, mobile)
- Flag missing states (loading, empty, error)
- Provide specific CSS fix suggestions
- Catch issues before users do
