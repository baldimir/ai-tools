# Alert Selection Guide

This guide provides patterns and strategies for user interaction when selecting which Dependabot alerts to fix.

## Overview

After fetching alerts, present the user with a clear summary and multiple selection options. This guide covers how to structure the interaction for optimal user experience.

## Alert Summary Presentation

### Format

```
Found X Dependabot alerts in this repository:

By Severity:
- Critical: X alerts
- High: X alerts
- Medium: X alerts
- Low: X alerts

Grouped by Library:
1. library-a (3 alerts - 2 critical, 1 high)
2. library-b (1 alert - high)
3. library-c (2 alerts - medium)
4. [more libraries...]
```

### Example

```
Found 12 Dependabot alerts in this repository:

By Severity:
- Critical: 3 alerts
- High: 5 alerts
- Medium: 3 alerts
- Low: 1 alert

Grouped by Library:
1. library-a (3 alerts - 2 critical, 1 high)
2. library-b (2 alerts - 1 high, 1 medium)
3. library-c (2 alerts - 1 high, 1 medium)
4. library-d (1 alert - high)
5. library-e (2 alerts - 1 medium, 1 low)
6. library-f (2 alerts - 1 high, 1 medium)
```

---

## Selection Options

### Option 1: Fix Specific Library

**Prompt:**
```
Which library would you like to fix?
```

**Suggested Answers:**
- library-a
- library-b
- library-c
- [list top 5-10 libraries]

**User Input Examples:**
- "Fix library-name"
- "library-name"
- "Fix all alerts for library-name"

**Processing:**
1. Match user input to library name (case-insensitive)
2. Select all alerts for that library
3. Confirm selection with user

---

### Option 2: Fix by Severity

**Prompt:**
```
Which severity level(s) would you like to fix?
```

**Suggested Answers:**
- Critical only
- High and Critical
- Medium and above
- All severities

**User Input Examples:**
- "Fix critical alerts"
- "Fix high and critical"
- "All critical and high severity"

**Processing:**
1. Parse severity level(s) from input
2. Filter alerts by severity
3. Group by library
4. Confirm selection with user

**Severity Mapping:**
```javascript
const severityLevels = {
  'critical': 4,
  'high': 3,
  'medium': 2,
  'low': 1
};

// "High and Critical" means severity >= 3
// "Medium and above" means severity >= 2
```

---

### Option 3: Fix Top N Alerts

**Prompt:**
```
How many alerts would you like to fix?
```

**Suggested Answers:**
- Top 5 most critical
- Top 10 most critical
- First 3 alerts

**User Input Examples:**
- "Fix top 5 alerts"
- "Fix the 3 most critical"
- "Top 10"

**Processing:**
1. Parse number from input
2. Sort alerts by severity (critical > high > medium > low)
3. Take top N alerts
4. Group by library
5. Confirm selection with user

**Sorting Logic:**
```javascript
// Sort by severity score (descending), then by CVSS score
alerts.sort((a, b) => {
  if (a.severity_score !== b.severity_score) {
    return b.severity_score - a.severity_score;
  }
  return b.cvss_score - a.cvss_score;
});
```

---

### Option 4: Fix All Alerts

**Prompt:**
```
Would you like to fix all alerts?
```

**Suggested Answers:**
- Yes, fix all alerts
- Fix everything

**User Input Examples:**
- "Fix all alerts"
- "Fix everything"
- "All"
- "Yes"

**Processing:**
1. Select all open alerts
2. Group by library
3. Confirm selection with user

**Warning:**
If there are many alerts (>10), warn the user:
```
⚠️ This will process 15 libraries with 23 total alerts.
This may take 30-60 minutes.
Are you sure you want to continue?
```

---

## Confirmation Step

### Display Selected Alerts

After user makes a selection, display what will be fixed:

```
You have selected to fix:

1. library-a (3 alerts)
   - Alert #1: CVE-2021-12345 (critical)
   - Alert #2: CVE-2020-67890 (high)
   - Alert #3: CVE-2019-11111 (critical)
   
2. library-b (2 alerts)
   - Alert #4: CVE-2023-22222 (high)
   - Alert #5: CVE-2023-33333 (medium)

Total: 2 libraries, 5 alerts

Estimated time: 10-20 minutes

Proceed with these fixes? (yes/no)
```

### Confirmation Prompt

```
Proceed with these fixes?
```

**Suggested Answers:**
- Yes
- No, let me select again
- Cancel

**User Input Examples:**
- "Yes"
- "Proceed"
- "Go ahead"
- "No" (return to selection)
- "Cancel" (exit skill)

---

## User Interaction Patterns

### Pattern 1: Direct Selection

```
User: "Fix all critical Dependabot alerts"

Bot: [Fetches alerts]
     Found 12 alerts, 3 are critical.
     
     Selected alerts:
     1. library-a (2 critical alerts)
     2. library-b (1 critical alert)
     
     Proceed? (yes/no)

User: "Yes"

Bot: [Processes alerts]
```

### Pattern 2: Interactive Selection

```
User: "Fix Dependabot alerts"

Bot: [Fetches alerts]
     Found 12 alerts.
     
     [Shows summary]
     
     How would you like to proceed?
     1. Fix specific library
     2. Fix by severity
     3. Fix top N alerts
     4. Fix all alerts

User: "Fix by severity"

Bot: Which severity level(s)?
     - Critical only
     - High and Critical
     - Medium and above
     - All severities

User: "High and Critical"

Bot: Selected 8 alerts (5 high, 3 critical) across 4 libraries.
     Proceed? (yes/no)

User: "Yes"

Bot: [Processes alerts]
```

### Pattern 3: Refinement

```
User: "Fix all alerts"

Bot: ⚠️ This will process 15 libraries with 23 alerts.
     This may take 30-60 minutes.
     
     Would you like to:
     1. Proceed with all
     2. Filter by severity
     3. Select specific libraries
     4. Cancel

User: "Filter by severity"

Bot: [Shows severity options]
```

---

## Input Parsing

### Library Name Matching

```javascript
function matchLibrary(userInput, libraries) {
  const input = userInput.toLowerCase().trim();
  
  // Exact match
  for (const lib of libraries) {
    if (lib.name.toLowerCase() === input) {
      return lib;
    }
  }
  
  // Partial match
  for (const lib of libraries) {
    if (lib.name.toLowerCase().includes(input) || 
        input.includes(lib.name.toLowerCase())) {
      return lib;
    }
  }
  
  return null;
}
```

### Severity Parsing

```javascript
function parseSeverity(userInput) {
  const input = userInput.toLowerCase();
  
  if (input.includes('critical') && !input.includes('high')) {
    return ['critical'];
  }
  
  if (input.includes('critical') && input.includes('high')) {
    return ['critical', 'high'];
  }
  
  if (input.includes('high') && !input.includes('critical')) {
    return ['high'];
  }
  
  if (input.includes('medium') && input.includes('above')) {
    return ['critical', 'high', 'medium'];
  }
  
  if (input.includes('all')) {
    return ['critical', 'high', 'medium', 'low'];
  }
  
  return null;
}
```

### Number Parsing

```javascript
function parseNumber(userInput) {
  const input = userInput.toLowerCase();
  
  // Match "top 5", "first 10", "5 alerts", etc.
  const match = input.match(/(\d+)/);
  
  if (match) {
    return parseInt(match[1], 10);
  }
  
  // Word numbers
  const wordNumbers = {
    'one': 1, 'two': 2, 'three': 3, 'four': 4, 'five': 5,
    'six': 6, 'seven': 7, 'eight': 8, 'nine': 9, 'ten': 10
  };
  
  for (const [word, num] of Object.entries(wordNumbers)) {
    if (input.includes(word)) {
      return num;
    }
  }
  
  return null;
}
```

---

## Error Handling

### No Alerts Found

```
No open Dependabot alerts found in this repository.

Possible reasons:
- No vulnerabilities detected
- Dependabot not enabled
- All alerts already fixed

Would you like me to:
1. Check Dependabot configuration
2. Exit
```

### Invalid Selection

```
I couldn't find a library named "librar" (did you mean "library-a"?).

Available libraries:
- library-a
- library-b
- library-c
[...]

Please select from the list above.
```

### Ambiguous Input

```
Your selection matches multiple options:
1. Fix "library-name" library (3 alerts)
2. Fix all "low" severity alerts (5 alerts)

Which did you mean?
```

---

## Best Practices

1. **Always Show Summary First**
   - Give user context before asking for selection
   - Show counts by severity and ecosystem

2. **Provide Clear Options**
   - List specific choices
   - Use suggested answers
   - Make it easy to select

3. **Confirm Before Processing**
   - Show exactly what will be fixed
   - Give estimated time
   - Allow user to cancel or refine

4. **Handle Ambiguity**
   - Ask clarifying questions
   - Suggest alternatives
   - Don't assume

5. **Warn for Large Operations**
   - Alert user if many alerts selected
   - Provide time estimate
   - Offer to filter

6. **Support Natural Language**
   - Parse various input formats
   - Be flexible with wording
   - Match intent, not exact phrases

---

## Example Interaction Flow

```
User: "Fix Dependabot alerts"

Bot: Fetching Dependabot alerts...
     
     Found 12 Dependabot alerts in this repository:
     
     By Severity:
     - Critical: 3 alerts
     - High: 5 alerts
     - Medium: 3 alerts
     - Low: 1 alert
     
     Grouped by Library:
     1. library-a (3 alerts - 2 critical, 1 high)
     2. library-b (2 alerts - 1 high, 1 medium)
     3. library-c (2 alerts - 1 high, 1 medium)
     4. library-d (1 alert - high)
     5. library-e (2 alerts - 1 medium, 1 low)
     6. library-f (2 alerts - 1 high, 1 medium)
     
     How would you like to proceed?

User: "Fix critical and high severity"

Bot: Selected 8 alerts across 4 libraries:
     
     1. lodash (3 alerts - 2 critical, 1 high)
     2. axios (1 alert - high)
     3. express (1 alert - high)
     4. requests (1 alert - high)
     5. spring-core (1 alert - high)
     
     Total: 5 libraries, 8 alerts
     Estimated time: 15-25 minutes
     
     Proceed with these fixes?

User: "Yes"

Bot: Starting Dependabot auto-fix process...
     
     [1/5] Processing lodash (3 alerts)...
     [...]
```

---

## Summary

This guide covers:
- ✅ Alert summary presentation format
- ✅ Four selection options (library, severity, top N, all)
- ✅ Confirmation step with details
- ✅ User interaction patterns
- ✅ Input parsing strategies
- ✅ Error handling approaches
- ✅ Best practices for user experience

**Key Principles:**
- Clear communication
- Flexible input parsing
- Always confirm before processing
- Provide context and estimates
- Handle errors gracefully

---

*Part of Dependabot Auto-Fix Skill v1.0.0*