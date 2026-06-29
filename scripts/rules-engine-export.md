---
name: Rules Engine Export Script
description: Browser console script that exports a Gainsight Rules Engine rule (metadata, actions, and bionic task configs) to a JSON file named Rule_{RuleName}_{RuleID}_{DateTime}.json
type: project
---
Pulls rule data from the Gainsight Rules Engine API and downloads it as a JSON export. Run in the browser console while on the rule's page.

> **Setup:** Replace `YOUR_TENANT` in the `BASE` constant at the top of the script with your Gainsight tenant subdomain (e.g., if your Gainsight URL is `acme.gainsightcloud.com`, use `acme`).

**API base:** `https://YOUR_TENANT.gainsightcloud.com`

**Endpoints used:**
- Rule metadata: `GET /v1/rulesengine/v2/{ruleId}`
- Task list (bionic only): `GET /v1/bionicreporting/config-ui/{ddConfigId}/tasks`
- Individual task config: `GET /v1/bionicreporting/config-ui/{ddConfigId}/task/{taskId}`

**Headers required:**
- `x-br-escid`: ruleId
- `x-br-system`: RULES
- `x-gs-host`: GAINSIGHT

**Output filename format:** `Rule_{sanitizedRuleName}_{ruleId}_{YYYYMMDD_HHmmss}.json`

**What's exported:** `results.rule`, `results.actions`, `results.tasks` (keyed by task ID)

**Full script:**

```javascript
const BASE = 'https://YOUR_TENANT.gainsightcloud.com'; // Replace YOUR_TENANT with your Gainsight subdomain

function getRuleId() {
  const match = window.location.href.match(/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}/i);
  if (match) {
    console.log(`Detected rule ID from URL: ${match[0]}`);
    return match[0];
  }
  return window.prompt('No rule ID found in URL — paste it here:');
}

function makeHeaders(escId) {
  return {
    'x-br-escid': escId,
    'x-br-system': 'RULES',
    'x-gs-host': 'GAINSIGHT',
    'content-type': 'application/json',
    'accept': 'application/json, text/plain, */*'
  };
}

function sanitize(str) {
  return str.replace(/[\/\\:*?"<>|]/g, '').trim();
}

function buildFilename(ruleName, ruleId) {
  const now = new Date();
  const dt = `${now.getFullYear()}${String(now.getMonth() + 1).padStart(2, '0')}${String(now.getDate()).padStart(2, '0')}_${String(now.getHours()).padStart(2, '0')}${String(now.getMinutes()).padStart(2, '0')}${String(now.getSeconds()).padStart(2, '0')}`;
  return `Rule_${sanitize(ruleName)}_${ruleId}_${dt}`;
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

async function pullRuleData() {
  const ruleId = getRuleId();
  if (!ruleId) { console.error('No rule ID provided.'); return; }

  const results = {};

  console.log(`[1/3] Fetching rule ${ruleId}...`);
  const ruleRes = await fetch(`${BASE}/v1/rulesengine/v2/${ruleId}`, { headers: makeHeaders(ruleId) });
  const ruleData = await ruleRes.json();
  results.rule = ruleData;

  const ddConfigId = ruleData?.data?.ruleDetails?.ddConfigId;
  const ruleName = ruleData?.data?.ruleDetails?.ruleName || ruleId;
  const actions = ruleData?.data?.gsRuleMetaActionDetails || [];

  console.log(`   Rule: "${ruleName}"`);
  console.log(`   ddConfigId: ${ddConfigId}`);
  console.log(`   Actions found in rule response: ${actions.length}`);

  results.actions = actions;
  actions.forEach((a, i) => {
    console.log(`   ✓ action ${i + 1}: ${a.actionId} (${a.actionType}, taskId: ${a.taskId})`);
  });

  if (!ddConfigId) {
    console.warn('No ddConfigId — not a bionic rule. Downloading partial export.');
    downloadJSON(results, buildFilename(ruleName, ruleId));
    return;
  }

  console.log(`[2/3] Fetching task list...`);
  const taskListRes = await fetch(
    `${BASE}/v1/bionicreporting/config-ui/${ddConfigId}/tasks`,
    { headers: makeHeaders(ruleId) }
  );
  const taskListData = await taskListRes.json();
  const allTasks = taskListData?.data || [];
  console.log(`   Found ${allTasks.length} data tasks: ${allTasks.map(t => `${t.id}(${t.type})`).join(', ')}`);

  console.log(`[3/3] Fetching individual task configs...`);
  results.tasks = {};
  for (const task of allTasks) {
    const res = await fetch(
      `${BASE}/v1/bionicreporting/config-ui/${ddConfigId}/task/${task.id}`,
      { headers: makeHeaders(ruleId) }
    );
    if (res.ok) {
      results.tasks[task.id] = await res.json();
      console.log(`   ✓ ${task.id} (${task.name})`);
    } else {
      console.warn(`   ✗ ${task.id} — ${res.status}`);
      results.tasks[task.id] = { error: res.status, name: task.name, type: task.type };
    }
  }

  downloadJSON(results, buildFilename(ruleName, ruleId));
}

pullRuleData();
```
