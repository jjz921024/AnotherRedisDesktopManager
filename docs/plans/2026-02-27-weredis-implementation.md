# WERedis Connection Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add WERedis connection support that dynamically fetches Redis proxy addresses via HTTP APIs instead of static host/port configuration.

**Architecture:** Minimal modification approach - add WERedis-specific fields to NewConnectionDialog.vue and handle dynamic address fetching in ConnectionWrapper.vue. Uses existing standalone Redis connection logic.

**Tech Stack:** Vue 2.6, Element UI, axios (or native fetch), ioredis

---

## Task 1: Add WERedis Fields to NewConnectionDialog.vue Data Model

**Files:**
- Modify: `src/components/NewConnectionDialog.vue:194-234` (data() section)

**Step 1: Add WERedis data properties**

In the `data()` function, add new properties after line 232 (after `sentinelOptionsShow: false`):

```javascript
// WERedis specific fields
weredis: false,
clusterNames: [],        // list of cluster names from API
clusterNameLoading: false,
clusterNameError: '',
```

And add to the `connection` object (after line 207, before `sshOptions`):

```javascript
weredis: false,
clusterName: '',
umAccount: '',
umPassword: '',
```

**Step 2: Update connectionEmpty backup**

The `connectionEmpty` is already backed up from `connection` in `mounted()`, so no additional changes needed there.

**Step 3: Verify no syntax errors**

Run: `npm run lint -- --fix src/components/NewConnectionDialog.vue 2>&1 | head -20`
Expected: No errors or auto-fixed issues

**Step 4: Commit**

```bash
git add src/components/NewConnectionDialog.vue
git commit -m "feat(weredis): add WERedis data properties to NewConnectionDialog

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 2: Add WERedis HTTP API Utility Functions

**Files:**
- Modify: `src/components/NewConnectionDialog.vue:250-318` (methods section)

**Step 1: Add HTTP fetch methods**

Add these methods to the `methods` object (after `editConnection()` method):

```javascript
// WERedis: Fetch cluster names from API
async fetchClusterNames() {
  this.clusterNameLoading = true;
  this.clusterNameError = '';
  this.clusterNames = [];

  try {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 5000);

    const response = await fetch('http://127.0.0.1:8080/api/weredis/getAllClusterNames', {
      signal: controller.signal,
    });
    clearTimeout(timeoutId);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    const data = await response.json();

    if (data.code !== '0' || !Array.isArray(data.resultData)) {
      throw new Error(data.msg || 'Invalid response format');
    }

    this.clusterNames = data.resultData;
  } catch (error) {
    console.error('WERedis: Failed to fetch cluster names:', error);
    this.clusterNameError = error.name === 'AbortError'
      ? 'Connection timeout'
      : `Failed to fetch cluster names: ${error.message}`;
    this.clusterNames = [];
  } finally {
    this.clusterNameLoading = false;
  }
},
```

**Step 2: Verify method is added correctly**

Run: `grep -n "fetchClusterNames" src/components/NewConnectionDialog.vue`
Expected: Shows the method definition line

**Step 3: Commit**

```bash
git add src/components/NewConnectionDialog.vue
git commit -m "feat(weredis): add fetchClusterNames HTTP method

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 3: Add WERedis UI Elements to Template

**Files:**
- Modify: `src/components/NewConnectionDialog.vue:1-186` (template section)

**Step 1: Add WERedis form fields at the top of the form**

Insert after line 4 (`<el-form :label-position...>`) and before the existing `<el-row>`:

```vue
<!-- WERedis Connection Fields -->
<el-row :gutter="20" v-if="true">
  <el-col :span="12">
    <el-form-item label="Cluster Name" required>
      <el-select
        v-model="connection.clusterName"
        placeholder="Select cluster"
        :loading="clusterNameLoading"
        :disabled="clusterNameLoading || clusterNameError"
        filterable
        style="width: 100%">
        <el-option
          v-for="name in clusterNames"
          :key="name"
          :label="name"
          :value="name">
        </el-option>
      </el-select>
      <div v-if="clusterNameError" style="color: #f56c6c; font-size: 12px; margin-top: 4px;">
        {{ clusterNameError }}
      </div>
    </el-form-item>
  </el-col>
  <el-col :span="12">
    <el-form-item label="Connection Name">
      <el-input v-model="connection.name" autocomplete="off" placeholder="Optional name"></el-input>
    </el-form-item>
  </el-col>
</el-row>

<el-row :gutter="20" v-if="true">
  <el-col :span="12">
    <el-form-item label="UM Account" required>
      <el-input v-model="connection.umAccount" autocomplete="off" placeholder="UM Account"></el-input>
    </el-form-item>
  </el-col>
  <el-col :span="12">
    <el-form-item label="UM Password" required>
      <el-input v-model="connection.umPassword" type="password" autocomplete="off" placeholder="UM Password"></el-input>
    </el-form-item>
  </el-col>
</el-row>

<fieldset style="margin-bottom: 10px;">
  <legend>WERedis Connection</legend>
</fieldset>
```

**Step 2: Hide existing fields by wrapping in v-show**

Wrap the existing redis connection form (lines 5-38, the first `<el-row>` with host/port/auth) with:

```vue
<div v-show="false">
  <!-- existing host/port/auth/username/separator form -->
</div>
```

**Step 3: Hide Cluster and Sentinel checkboxes**

Find the checkbox section (lines 41-65) and change the Cluster and Sentinel checkboxes to hidden:

```vue
<el-checkbox v-model="connection.cluster" v-show="false">Cluster</el-checkbox>
<el-checkbox v-model="sentinelOptionsShow" v-show="false">Sentinel</el-checkbox>
```

Keep SSH, SSL, and Readonly checkboxes visible if needed (or hide based on requirements).

**Step 4: Verify template compiles**

Run: `npm run lint -- --fix src/components/NewConnectionDialog.vue 2>&1 | head -20`
Expected: No template errors

**Step 5: Commit**

```bash
git add src/components/NewConnectionDialog.vue
git commit -m "feat(weredis): add WERedis form fields and hide original fields

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 4: Fetch Cluster Names on Dialog Open

**Files:**
- Modify: `src/components/NewConnectionDialog.vue` (methods section)

**Step 1: Call fetchClusterNames in show() method**

Modify the `show()` method to fetch cluster names:

```javascript
show() {
  this.dialogVisible = true;
  this.resetFields();
  // Fetch cluster names when dialog opens
  this.fetchClusterNames();
},
```

**Step 2: Verify the change**

Run: `grep -A3 "show()" src/components/NewConnectionDialog.vue | head -6`
Expected: Shows the method with fetchClusterNames call

**Step 3: Commit**

```bash
git add src/components/NewConnectionDialog.vue
git commit -m "feat(weredis): fetch cluster names when dialog opens

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 5: Update editConnection to Save WERedis Fields

**Files:**
- Modify: `src/components/NewConnectionDialog.vue:273-300` (editConnection method)

**Step 1: Add WERedis validation and flag setting**

Modify the `editConnection()` method to handle WERedis:

```javascript
editConnection() {
  const config = JSON.parse(JSON.stringify(this.connection));

  // WERedis validation
  if (!config.clusterName) {
    return this.$message.error('Cluster Name is required');
  }
  if (!config.umAccount) {
    return this.$message.error('UM Account is required');
  }
  if (!config.umPassword) {
    return this.$message.error('UM Password is required');
  }

  // Mark as WERedis connection
  config.weredis = true;

  // Set defaults for hidden fields
  !config.host && (config.host = '127.0.0.1');
  !config.port && (config.port = 6379);

  // Always use standalone mode (not cluster, not sentinel)
  config.cluster = false;
  delete config.sentinelOptions;

  // Clean up SSH/SSL if not configured
  if (!this.sshOptionsShow || !config.sshOptions.host) {
    delete config.sshOptions;
  }
  if (!this.sslOptionsShow) {
    delete config.sslOptions;
  }

  const oldKey = storage.getConnectionKey(this.config);
  storage.editConnectionByKey(config, oldKey);

  this.dialogVisible = false;
  this.$emit('editConnectionFinished', config);
},
```

**Step 2: Verify method updated**

Run: `grep -n "config.weredis = true" src/components/NewConnectionDialog.vue`
Expected: Shows the line number

**Step 3: Commit**

```bash
git add src/components/NewConnectionDialog.vue
git commit -m "feat(weredis): save WERedis fields and mark connection as weredis

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 6: Handle WERedis Edit Mode

**Files:**
- Modify: `src/components/NewConnectionDialog.vue:255-272` (resetFields method)

**Step 1: Update resetFields to restore WERedis fields**

Modify the `resetFields()` method to handle WERedis connections:

```javascript
resetFields() {
  // edit connection mode
  if (this.editMode) {
    this.sshOptionsShow = !!this.config.sshOptions;
    this.sslOptionsShow = !!this.config.sslOptions;
    this.sentinelOptionsShow = !!this.config.sentinelOptions;
    // recovery connection before edit
    const connection = Object.assign({}, this.connectionEmpty, this.config);
    this.connection = JSON.parse(JSON.stringify(connection));
  }
  // new connection mode
  else {
    this.sshOptionsShow = false;
    this.sslOptionsShow = false;
    this.sentinelOptionsShow = false;
    this.connection = JSON.parse(JSON.stringify(this.connectionEmpty));
  }

  // Reset WERedis state
  this.clusterNameError = '';
  // clusterNames and loading state managed by fetchClusterNames
},
```

**Step 2: Commit**

```bash
git add src/components/NewConnectionDialog.vue
git commit -m "feat(weredis): reset WERedis state in resetFields

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 7: Add WERedis Proxy Fetch to ConnectionWrapper

**Files:**
- Modify: `src/components/ConnectionWrapper.vue:157-194` (getRedisClient method)

**Step 1: Add fetchProxyAddress method**

Add a new method before `getRedisClient()`:

```javascript
// WERedis: Fetch proxy address from API
async fetchWeridesProxyAddress(clusterName) {
  try {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 5000);

    const response = await fetch(
      `http://127.0.0.1:19091/redis_observer/proxy_online_list?clusterName=${encodeURIComponent(clusterName)}`,
      { signal: controller.signal }
    );
    clearTimeout(timeoutId);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    const data = await response.json();

    if (!data.result || !Array.isArray(data.result) || data.result.length === 0) {
      throw new Error('No available proxies');
    }

    // Use first proxy
    const proxy = data.result[0];
    if (!proxy.host || !proxy.port) {
      throw new Error('Invalid proxy configuration');
    }

    return { host: proxy.host, port: proxy.port };
  } catch (error) {
    console.error('WERedis: Failed to fetch proxy address:', error);
    throw new Error(
      error.name === 'AbortError'
        ? 'Connection timeout when fetching proxy address'
        : `Failed to fetch proxy: ${error.message}`
    );
  }
},
```

**Step 2: Modify getRedisClient to handle WERedis**

Update the `getRedisClient()` method to check for WERedis connections:

```javascript
getRedisClient(config) {
  return new Promise(async (resolve, reject) => {
    // prevent changing back to raw config, such as config.db
    const configCopy = JSON.parse(JSON.stringify(config));
    // select db
    configCopy.db = this.lastSelectedDb;

    // WERedis connection: fetch proxy address first
    if (configCopy.weredis && configCopy.clusterName) {
      try {
        const proxy = await this.fetchWeridesProxyAddress(configCopy.clusterName);
        configCopy.host = proxy.host;
        configCopy.port = proxy.port;
        configCopy.auth = 'wb6Cluster';  // hardcoded password
        console.log(`WERedis: Using proxy ${proxy.host}:${proxy.port} for cluster ${configCopy.clusterName}`);
      } catch (error) {
        this.$message.error(error.message);
        this.$refs.operateItem.searchIcon = 'el-icon-search';
        return reject(error);
      }
    }

    // ssh client
    if (configCopy.sshOptions) {
      var clientPromise = redisClient.createSSHConnection(
        configCopy.sshOptions, configCopy.host, configCopy.port, configCopy.auth, configCopy,
      );
    }
    // normal client
    else {
      var clientPromise = redisClient.createConnection(
        configCopy.host, configCopy.port, configCopy.auth, configCopy,
      );
    }

    clientPromise.then((client) => {
      this.client = client;

      client.on('error', (error) => {
        this.$message.error({
          message: `Client On Error: ${error} Config right?`,
          duration: 3000,
          customClass: 'redis-on-error-message',
        });

        this.$bus.$emit('closeConnection');
      });
    }).catch((error) => {
      this.$message.error(error.message);
      this.$bus.$emit('closeConnection');
    });

    resolve(clientPromise);
  });
},
```

**Step 3: Verify changes**

Run: `grep -n "fetchWeridesProxyAddress\|configCopy.weredis" src/components/ConnectionWrapper.vue`
Expected: Shows the new method and WERedis check

**Step 4: Commit**

```bash
git add src/components/ConnectionWrapper.vue
git commit -m "feat(weredis): add dynamic proxy fetching for WERedis connections

- Add fetchWeridesProxyAddress method to get proxy from HTTP API
- Modify getRedisClient to fetch proxy before connecting
- Use hardcoded password 'wb6Cluster' for WERedis

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 8: Test WERedis Connection Flow

**Files:**
- Test: Manual testing in Electron app

**Step 1: Start the development server**

Run: `npm run dev`
Expected: Electron app launches

**Step 2: Test new connection dialog**

1. Click "New Connection" button
2. Verify:
   - Cluster Name dropdown is visible
   - UM Account and Password fields are visible
   - Original host/port/auth fields are hidden
   - Loading state shows when fetching cluster names

**Step 3: Test connection creation**

1. Select a cluster name from dropdown
2. Enter UM Account and Password
3. Click Confirm
4. Verify connection is saved with `weredis: true`

**Step 4: Test connection opening**

1. Click on the saved connection
2. Verify:
   - Proxy address is fetched from API (check console log)
   - Redis connection is established
   - Keys are loaded

**Step 5: Test error handling**

1. Stop the HTTP API servers
2. Try to create/open a connection
3. Verify error messages are shown

**Step 6: Document any issues found**

If issues found, create follow-up tasks to fix them.

---

## Task 9: Final Verification and Cleanup

**Files:**
- Review all modified files

**Step 1: Run linter on all modified files**

Run: `npm run lint -- --fix src/components/NewConnectionDialog.vue src/components/ConnectionWrapper.vue`
Expected: No errors

**Step 2: Verify backward compatibility**

1. Create a standard (non-WERedis) connection
2. Open and use the connection
3. Verify existing functionality works

**Step 3: Final commit (if any fixes needed)**

```bash
git add -A
git commit -m "fix(weredis): final cleanup and fixes

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Summary

| Task | Description | File(s) |
|------|-------------|---------|
| 1 | Add WERedis data properties | NewConnectionDialog.vue |
| 2 | Add HTTP fetch method | NewConnectionDialog.vue |
| 3 | Add WERedis UI elements | NewConnectionDialog.vue |
| 4 | Fetch clusters on dialog open | NewConnectionDialog.vue |
| 5 | Save WERedis fields | NewConnectionDialog.vue |
| 6 | Handle WERedis edit mode | NewConnectionDialog.vue |
| 7 | Add proxy fetch to connection | ConnectionWrapper.vue |
| 8 | Manual testing | - |
| 9 | Final verification | - |
