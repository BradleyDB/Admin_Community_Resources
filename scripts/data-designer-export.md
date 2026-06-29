---
name: Data Designer Export Script
description: Browser console script that exports a Gainsight Data Designer config (metadata + all task configs) to a JSON file named DD_{Name}_{ID}_{DateTime}.json
type: project
---
Pulls Data Designer config from the Gainsight bionic reporting API and downloads it as a JSON export. Run in the browser console while on the Data Designer's page.

> **Setup:** Replace `YOUR_TENANT` in the `BASE` constant at the top of the script with your Gainsight tenant subdomain (e.g., if your Gainsight URL is `acme.gainsightcloud.com`, use `acme`).

**API base:** `https://YOUR_TENANT.gainsightcloud.com`

**Endpoints used:**
- DD metadata: `GET /v1/bionicreporting/config-ui/{ddId}`
- Task list: `GET /v1/bionicreporting/config-ui/{ddId}/tasks`
- Individual task config: `GET /v1/bionicreporting/config-ui/{ddId}/task/{taskId}`

**Headers required:**
- `x-br-escid`: ddId
- `x-br-system`: DATA_DESIGNER
- `x-gs-host`: GAINSIGHT

**Name field path in metadata response:** `metaData?.data?.name` (confirmed working)

**Output filename format:** `DD_{sanitizedName}_{ddId}_{YYYYMMDD_HHmmss}.json`

**What's exported:** `results.metadata`, `results.tasks` (keyed by task ID)

**Key paths when parsing a DD export for documentation:**
- Task name/type: `tasks[id].data.name`, `tasks[id].data.type`
- Extract fields: `tasks[id].data.queryInfo.show[]` → use `displayName` and `fieldAlias`
- Extract filters: `tasks[id].data.queryInfo.criteria.conditions[]` → use `leftOperand.label`, `comparisonOperator`, `filterValue`, `filterAlias`; boolean expression at `criteria.expression`
- GroupBy fields: `tasks[id].data.queryInfo.groupBy[]` → use `displayName`
- **Join conditions (merge/join tasks):** `tasks[id].data.queryInfo.joinChain[].joinOn.conditions[]`
  - Left side: `conditions[].leftOperand.displayName` + `leftOperand.objectLabel`
  - Right side: `conditions[].filterField.displayName` + `filterField.objectLabel`
  - Operator: `conditions[].comparisonOperator`
- Parent/child task links: `tasks[id].data.parents[]`, `tasks[id].data.children[]`
- freeForm task description: `tasks[id].data.description` (plain text note on what the transform does)

**Full script:**

```javascript
const BASE = 'https://YOUR_TENANT.gainsightcloud.com'; // Replace YOUR_TENANT with your Gainsight subdomain

function getDDId() {
  const match = window.location.href.match(/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}/i);
  if (match) {
    console.log(`Detected Data Designer ID from URL: ${match[0]}`);
    return match[0];
  }
  return window.prompt('No Data Designer ID found in URL — paste it here:');
}

function makeHeaders(ddId) {
  return {
    'x-br-escid': ddId,
    'x-br-system': 'DATA_DESIGNER',
    'x-gs-host': 'GAINSIGHT',
    'content-type': 'application/json',
    'accept': 'application/json, text/plain, */*'
  };
}

function sanitize(str) {
  return str.replace(/[\/\\:*?"<>|]/g, '').trim();
}

function buildFilename(ddName, ddId) {
  const now = new Date();
  const dt = `${now.getFullYear()}${String(now.getMonth() + 1).padStart(2, '0')}${String(now.getDate()).padStart(2, '0')}_${String(now.getHours()).padStart(2, '0')}${String(now.getMinutes()).padStart(2, '0')}${String(now.getSeconds()).padStart(2, '0')}`;
  return `DD_${sanitize(ddName)}_${ddId}_${dt}`;
}

function downloadJSON(data, filename) {
  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `${filename}.json`;
  a.click();
  URL.revokeObjectURL(url);
  console.log(`✅ Downloaded ${filename}.json`);
}

async function pullDDData() {
  const ddId = getDDId();
  if (!ddId) { console.error('No Data Designer ID provided.'); return; }

  const results = {};

  console.log(`[1/3] Fetching Data Designer metadata ${ddId}...`);
  const metaRes = await fetch(`${BASE}/v1/bionicreporting/config-ui/${ddId}`, { headers: makeHeaders(ddId) });
  const metaData = await metaRes.json();
  results.metadata = metaData;

  const ddName =
    metaData?.data?.name ||
    metaData?.data?.title ||
    metaData?.data?.ddName ||
    metaData?.data?.configName ||
    ddId;

  console.log(`   Data Designer: "${ddName}"`);

  console.log(`[2/3] Fetching task list...`);
  const taskListRes = await fetch(
    `${BASE}/v1/bionicreporting/config-ui/${ddId}/tasks`,
    { headers: makeHeaders(ddId) }
  );
  const taskListData = await taskListRes.json();
  const allTasks = taskListData?.data || [];
  console.log(`   Found ${allTasks.length} tasks: ${allTasks.map(t => `${t.id}(${t.type})`).join(', ')}`);

  console.log(`[3/3] Fetching individual task configs...`);
  results.tasks = {};
  for (const task of allTasks) {
    const res = await fetch(
      `${BASE}/v1/bionicreporting/config-ui/${ddId}/task/${task.id}`,
      { headers: makeHeaders(ddId) }
    );
    if (res.ok) {
      results.tasks[task.id] = await res.json();
      console.log(`   ✓ ${task.id} (${task.name})`);
    } else {
      console.warn(`   ✗ ${task.id} — ${res.status}`);
      results.tasks[task.id] = { error: res.status, name: task.name, type: task.type };
    }
  }

  downloadJSON(results, buildFilename(ddName, ddId));
}

pullDDData();
```
