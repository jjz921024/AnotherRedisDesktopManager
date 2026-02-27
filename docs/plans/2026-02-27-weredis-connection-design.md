# WERedis Connection Interface Design

**Date**: 2026-02-27
**Status**: Approved
**Approach**: Minimal Modification

## Overview

This document describes the design for customizing the connection creation interface to support WERedis connections, which dynamically fetch Redis proxy addresses from HTTP APIs instead of using static host/port configuration.

## Requirements Summary

1. **Fixed connection method**: Always use standalone Redis connection (hide Cluster/Sentinel options)
2. **Dynamic address resolution**: Redis address fetched from HTTP API based on ClusterName
3. **New required fields**: ClusterName (dropdown), UM Account, UM Password
4. **Hardcoded password**: `wb6Cluster`
5. **Minimal code changes**: Modify only essential files

## HTTP API Endpoints

### Get Cluster Names
- **URL**: `GET http://127.0.0.1:8080/api/weredis/getAllClusterNames`
- **Response**:
```json
{
  "code": "0",
  "msg": "请求处理成功",
  "resultData": [
    "GNS_GENERAL_PRESSURE_REDIS_CLUSTER_DATASTORE",
    "RPD_GENERAL_REDIS_CLUSTER_CACHE",
    "test-cluster"
  ]
}
```

### Get Proxy Address
- **URL**: `GET http://127.0.0.1:19091/redis_observer/proxy_online_list?clusterName={clusterName}`
- **Response**:
```json
{
  "code": "0",
  "msg": "请求处理成功",
  "result": [{"host": "xx", "port": 6379}]
}
```

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Connect flow | Manual | User clicks connect after selecting ClusterName |
| UM credentials usage | Local auth only | Stored but not sent in HTTP requests |
| Error handling | Disable connect | Disable button when API fails |
| API configuration | Hardcoded | Endpoints fixed as specified |
| Storage format | Minimal | Store only clusterName + UM creds, fetch address fresh |

## UI Changes

### NewConnectionDialog.vue

**Elements to Hide**:
- Host input field
- Port input field
- Auth input field
- Username input field
- Cluster checkbox
- Sentinel checkbox
- Sentinel options section

**Elements to Add**:
- **ClusterName dropdown** (required)
  - Label: "Cluster Name"
  - Options fetched from HTTP API on dialog open
  - Shows loading state during fetch
  - Shows error message if fetch fails
  - Disabled state when API unavailable

- **UM Account input** (required)
  - Label: "UM Account"
  - Type: text

- **UM Password input** (required)
  - Label: "UM Password"
  - Type: password

**Button States**:
- "Save" button: Disabled until all required fields filled AND cluster names loaded

## Connection Flow

```
User opens connection
        |
        v
Check connection.weredis === true?
        |
   +----+----+
   |         |
  No        Yes
   |         |
   v         v
Existing   Fetch proxy list from
flow       http://127.0.0.1:19091/redis_observer/proxy_online_list?clusterName={name}
             |
             v
         API success?
             |
        +----+----+
        |         |
       No        Yes
        |         |
        v         v
     Show      Extract host/port
     error     from result[0]
                  |
                  v
              Call redisClient.createConnection(
                host, port, 'wb6Cluster', {}
              )
```

## Data Storage Format

### WERedis Connection Object
```javascript
{
  key: '1709000000000_abc123',
  name: 'My WERedis Connection',
  weredis: true,
  clusterName: 'GNS_GENERAL_PRESSURE_REDIS_CLUSTER_DATASTORE',
  umAccount: 'user123',
  umPassword: 'password123',
  order: 1
}
```

### Not Stored (fetched at runtime)
- `host`, `port` - fetched from proxy API
- `auth` - hardcoded as `wb6Cluster`

## File Changes

| File | Change Type | Description |
|------|-------------|-------------|
| `src/components/NewConnectionDialog.vue` | Modify | Add WERedis form fields, hide existing fields, add HTTP fetch logic |
| `src/components/ConnectionWrapper.vue` | Modify | Add WERedis connection handling with dynamic address fetch |

### No Changes Required
- `src/redisClient.js` - Uses existing standalone connection
- `src/storage.js` - Existing structure sufficient
- `src/components/Connections.vue` - Connection list works as-is

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Cluster names API (8080) fails | Show error message in dropdown area, disable Save button |
| Proxy list API (19091) fails | Show error toast notification, do not connect |
| Proxy list returns empty array | Show "No available proxies" error, do not connect |
| Network timeout (5s) | Show "Connection timeout" error |

## Backward Compatibility

- Existing non-WERedis connections continue to work unchanged
- The `weredis` flag defaults to `false`/`undefined` for existing connections
- No migration required for existing connection data

## Implementation Notes

1. Use axios or native fetch for HTTP requests (project uses axios)
2. Add 5-second timeout for HTTP requests
3. Handle JSON parsing errors gracefully
4. Log errors to console for debugging
