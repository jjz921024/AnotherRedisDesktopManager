# TairHash UI Display Support Design

## Overview

Add support for displaying TairHash data structure in the Redis Desktop Manager UI.

## Background

[TairHash](https://github.com/tair-opensource/TairHash) is a Redis module that extends the native hash data structure with:
- Field-level expiration (TTL)
- Field-level versioning (for optimistic locking)
- Active and passive expiration mechanisms

## Requirements

### Functional Requirements
- **Read-only display**: View TairHash data without editing capability
- **Display fields**: Field, Value, Expire, Version
- **Type detection**: Redis `TYPE` command returns `tairhash-`

### Non-Functional Requirements
- Performance: Use parallel requests for TTL and version data
- Consistency: Match existing component styling

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        KeyDetail.vue                        │
│  getComponentNameByType(keyType) {                          │
│    'tairhash-': 'KeyContentTairHash',  // NEW               │
│    ...                                                      │
│  }                                                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  KeyContentTairHash.vue (NEW)               │
│  vxe-table with 4 columns:                                  │
│  ┌──────────┬──────────┬──────────┬──────────────┐         │
│  │  Field   │  Value   │  Expire  │   Version    │         │
│  └──────────┴──────────┴──────────┴──────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

## Data Fetching Strategy

### Commands Used
| Command | Purpose | Return Value |
|---------|---------|--------------|
| `EXHGETALL key` | Get all fields and values | `[field1, value1, field2, value2, ...]` |
| `EXHPTTL key field` | Get field TTL in milliseconds | `-1` = never expire, `-2` = expired |
| `EXHVER key field` | Get field version | Integer |

### Flow
1. Call `EXHGETALL` to get field-value pairs
2. Parse into `[{field, value}, ...]`
3. Use `Promise.all` to parallelly fetch TTL and version for each field
4. Merge into final data structure

## UI Design

### Table Columns
| Column | Width | Alignment | Format |
|--------|-------|-----------|--------|
| Field | Auto | Left | Plain text |
| Value | Auto | Left | FormatViewer (JSON/Hex/etc) |
| Expire | 120px | Center | Formatted time |
| Version | 80px | Center | Number |

### Expire Time Format
```
-1        → "永不过期" / "Never"
-2        → "已过期" / "Expired"
3600000   → "1小时" / "1h"
60000     → "1分钟" / "1m"
1000      → "1秒" / "1s"
```

## Implementation Steps

### Step 1: Modify KeyDetail.vue
Add type mapping in `getComponentNameByType()`:
```javascript
'tairhash-': 'KeyContentTairHash',
```

### Step 2: Create KeyContentTairHash.vue
- Based on `KeyContentHash.vue` structure
- Add Expire and Version columns
- Implement `loadData()` with TairHash commands
- Use `FormatViewer` for value display

### Step 3: Add i18n Support
- `src/i18n/en.js`
- `src/i18n/zh.js`

### Step 4: Testing
- Connect to TairHash Redis instance
- Verify data display
- Verify search and refresh functionality

## Files to Modify

1. `src/components/KeyDetail.vue` - Add type mapping
2. `src/components/contents/KeyContentTairHash.vue` - New component
3. `src/i18n/en.js` - English translations
4. `src/i18n/zh.js` - Chinese translations

## Estimated Effort

1-2 hours

## References

- [TairHash GitHub](https://github.com/tair-opensource/TairHash)
- Existing `KeyContentHash.vue` component
