/***** CONFIG *****/
const OT_BASE = 'https://app.onetrust.com';   // You log in at app.onetrust.com
//const OT_BASE = 'https://customer.my.onetrust.com';   // Alternative tenant base

const SHEET_NAME = 'DomainGroups';            // Exact name of your sheet
const STATUS_COL = 4;                         // Column D
const REPLACE_EXISTING = true;                // true = overwrite, false = append/merge if supported

/***** CDN READ CONFIG *****/
const CDN_CONSENT_BASE = 'https://cdn.cookielaw.org/consent';

function onOpen() {
  SpreadsheetApp.getUi()
    .createMenu('OneTrust Tools')
    .addItem('Validate Setup', 'validateSetup')
    .addItem('Create Domain Groups', 'createOneTrustDomainGroups')
    .addItem('Check Domain Group (prompt)', 'checkDomainGroupInteractive') // uses CDN now
    .addToUi();
}

/***** AUTH: get an access token (or use static if provided) *****/
function getOneTrustAccessToken() {
  const props = PropertiesService.getScriptProperties();
  const staticToken = (props.getProperty('OT_BEARER_TOKEN') || '').trim();
  if (staticToken) {
    Logger.log('Using static OT_BEARER_TOKEN from Script Properties.');
    return staticToken;
  }

  const clientId = props.getProperty('OT_CLIENT_ID');
  const clientSecret = props.getProperty('OT_CLIENT_SECRET');
  if (!clientId || !clientSecret) {
    throw new Error('Missing OT_BEARER_TOKEN or OT_CLIENT_ID / OT_CLIENT_SECRET in Script Properties.');
  }

  const tokenUrl = `${OT_BASE}/api/access/v1/oauth/token`;

  function request(payload) {
    const res = UrlFetchApp.fetch(tokenUrl, {
      method: 'post',
      contentType: 'application/x-www-form-urlencoded',
      muteHttpExceptions: true,
      payload
    });
    return { code: res.getResponseCode(), text: res.getContentText() };
  }

  // Try with scope first; if tenant rejects, retry without scope.
  let { code, text } = request({
    grant_type: 'client_credentials',
    client_id: clientId,
    client_secret: clientSecret,
    scope: 'COOKIE'
  });
  if (code === 400) {
    try {
      const j = JSON.parse(text);
      if (String(j.error || '').toLowerCase() === 'invalid_scope') {
        Logger.log('Token error invalid_scope — retrying WITHOUT scope…');
        ({ code, text } = request({
          grant_type: 'client_credentials',
          client_id: clientId,
          client_secret: clientSecret
        }));
      }
    } catch (_) {}
  }
  Logger.log(`Token response ${code}: ${text}`);
  if (code !== 200) throw new Error(`Token request failed (${code})`);

  const json = JSON.parse(text);
  if (!json.access_token) throw new Error('No access_token in token response');
  return json.access_token;
}

/***** MAIN: read rows and create/update domain groups *****/
function createOneTrustDomainGroups() {
  const token = getOneTrustAccessToken();
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAME);
  if (!sh) throw new Error(`Sheet "${SHEET_NAME}" not found.`);

  // Ensure Status header
  if (sh.getRange(1, STATUS_COL).getValue() !== 'Status') {
    sh.getRange(1, STATUS_COL).setValue('Status');
  }

  const lastRow = sh.getLastRow();
  if (lastRow < 2) { Logger.log('No data rows.'); return; }

  // Use display values (strips formulas/formatting surprises)
  const rows = sh.getRange(2, 1, lastRow - 1, Math.max(3, sh.getLastColumn())).getDisplayValues();

  const uuidRe = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
  let ok = 0, fail = 0;

  rows.forEach((r, i) => {
    const rowNum = i + 2;
    const mainDomain = clean(r[0]);   // col A hostname (optional)
    const mainId     = clean(r[1]);   // col B domainId (preferred)
    const groupRaw   = clean(r[2]);   // col C list

    // Skip blank lines
    if (!mainDomain && !mainId && !groupRaw) {
      writeStatus(sh, rowNum, 'Skipped (empty row)');
      return;
    }

    try {
      // Parse group domains: comma/semicolon/newline
      const groupDomains = parseDomainList(groupRaw);
      if (groupDomains.length === 0) {
        writeStatus(sh, rowNum, 'No group domains in column C');
        fail++; return;
      }

      // Resolve domainId: prefer the explicit B value; otherwise look up from A
      let domainId = null;
      if (mainId) {
        if (!uuidRe.test(mainId)) throw new Error(`Invalid domainId in column B: "${mainId}"`);
        domainId = mainId;
      } else {
        if (!mainDomain) throw new Error('Need either Main Domain id (B) or main domain hostname (A)');
        domainId = resolveDomainIdByHostname(token, mainDomain);
        if (!domainId) throw new Error(`Could not resolve domainId for hostname "${mainDomain}"`);
        // Write back the resolved id to column B for next time
        sh.getRange(rowNum, 2).setValue(domainId);
      }

      Logger.log(`Row ${rowNum}: Sending domainId=${domainId}, urls=${JSON.stringify(groupDomains)}`);
      const { code } = createOrUpdateDomainGroup(token, domainId, groupDomains);
      writeStatus(sh, rowNum, `OK ${code}`);
      ok++;
    } catch (e) {
      writeStatus(sh, rowNum, `ERROR: ${e.message}`);
      fail++;
    }
  });

  Logger.log(`Done. Success: ${ok}, Failed: ${fail}`);
}

/***** VALIDATOR: quick pre-flight check (no mutations) *****/
function validateSetup() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAME);
  if (!sh) throw new Error(`Sheet "${SHEET_NAME}" not found.`);

  const lastRow = sh.getLastRow();
  const data = sh.getRange(1, 1, Math.max(2, lastRow), Math.max(3, sh.getLastColumn())).getDisplayValues();
  Logger.log(`"${SHEET_NAME}" rows=${data.length}, cols=${(data[0]||[]).length}`);
  if (data.length >= 2) {
    const A2 = clean(String(data[1][0] || ''));
    const B2 = clean(String(data[1][1] || ''));
    const C2 = clean(String(data[1][2] || ''));
    Logger.log(`A2 (main domain): "${A2}"`);
    Logger.log(`B2 (domainId):    "${B2}"`);
    Logger.log(`C2 (group list):  "${C2}"`);
    const parsed = parseDomainList(C2);
    Logger.log(`Parsed C2 into ${parsed.length} domain(s): ${JSON.stringify(parsed)}`);
  }

  try {
    const t = getOneTrustAccessToken();
    Logger.log(`Access token OK (${t.length} chars).`);
  } catch (e) {
    Logger.log(`❗ Token check failed: ${e.message}`);
  }
}

/***** LOOKUP: resolve domainId from hostname if B is empty *****/
function resolveDomainIdByHostname(accessToken, hostname) {
  const host = normalizeHostname(hostname);
  // Try Websites list (paged)
  const base = `${OT_BASE}/api/cookiemanager/v2/websites`;
  for (let page = 0; page < 50; page++) {
    const url = `${base}?page=${page}&size=${200}`;
    const res = UrlFetchApp.fetch(url, {
      method: 'get',
      headers: { Authorization: `Bearer ${accessToken}` },
      muteHttpExceptions: true
    });
    if (res.getResponseCode() !== 200) {
      Logger.log(`Websites list failed page=${page}: ${res.getResponseCode()} ${res.getContentText()}`);
      return null;
    }
    const j = JSON.parse(res.getContentText());
    const content = Array.isArray(j?.content) ? j.content : (Array.isArray(j) ? j : []);
    const hit = content.find(item => {
      const candidates = [
        (item.domain || ''), (item.website || ''), (item.hostname || '')
      ].map(s => normalizeHostname(String(s || ''))).filter(Boolean);
      return candidates.includes(host);
    });
    if (hit && hit.id) return hit.id;
    if (j?.last === true || content.length === 0) break;
  }
  return null;
}

/***** CALL: create/update the domain group (UPDATED) *****/
function createOrUpdateDomainGroup(accessToken, domainId, groupDomains) {
  if (!domainId || !Array.isArray(groupDomains) || groupDomains.length === 0) {
    throw new Error('Refusing to call API: missing domainId or groupDomains.');
  }
  const url = `${OT_BASE}/api/cookiemanager/v1/domains/${encodeURIComponent(domainId)}/domaingroup`;

  // New API expects { urls: [...], removeExisting: true|false }
  const payload = {
    urls: groupDomains,
    removeExisting: REPLACE_EXISTING
  };

  Logger.log(`POST ${url}`);
  Logger.log(`Payload: ${JSON.stringify(payload)}`);

  const res = UrlFetchApp.fetch(url, {
    method: 'post',
    contentType: 'application/json',
    headers: { Authorization: `Bearer ${accessToken}` },
    muteHttpExceptions: true,
    payload: JSON.stringify(payload)
  });
  const code = res.getResponseCode();
  const body = res.getContentText();
  Logger.log(`Response ${code}: ${body}`);
  if (code < 200 || code >= 300) throw new Error(`HTTP ${code}: ${body}`);
  return { code, body };
}

/***** UTIL *****/
function writeStatus(sh, row, msg) { sh.getRange(row, STATUS_COL).setValue(msg); }
function parseDomainList(s) {
  if (!s) return [];
  const parts = String(s).split(/[,;\n\r]+/).map(x => clean(x)).filter(Boolean);
  const seen = new Set(), out = [];
  for (const p of parts) if (!seen.has(p)) { seen.add(p); out.push(p); }
  return out;
}
function normalizeHostname(v) {
  let s = String(v || '').trim();
  s = s.replace(/^https?:\/\//i, '').replace(/\/.*$/, '');
  s = s.replace(/^www\./i, '');
  return s.toLowerCase();
}
function clean(s) { return String(s || '').replace(/[\u200B\u202F\u00A0]/g, ' ').trim(); }

/***** Helper: UUID checker *****/
function isUUID(v) {
  return /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i.test(String(v || '').trim());
}

/***** Reporting: prompt for a domainId or hostname and write mapped domains via CDN *****/
const REPORT_SHEET = 'domainCheck';  // target worksheet name

function checkDomainGroupInteractive() {
  const ui = SpreadsheetApp.getUi();
  const resp = ui.prompt(
    'Check Domain Group (CDN)',
    'Enter a Domain ID (UUID) OR a main hostname (e.g. example.com):',
    ui.ButtonSet.OK_CANCEL
  );
  if (resp.getSelectedButton() !== ui.Button.OK) return;

  const input = (resp.getResponseText() || '').trim();
  if (!input) {
    ui.alert('Missing input', 'Please enter a Domain ID or hostname.', ui.ButtonSet.OK);
    return;
  }

  try {
    let domainId;
    if (isUUID(input)) {
      domainId = input;
    } else {
      const host = normalizeHostname(input);
      if (!host) throw new Error('That does not look like a valid hostname.');
      // Need auth only to resolve hostname -> domainId
      const token = getOneTrustAccessToken();
      domainId = resolveDomainIdByHostname(token, host);
      if (!domainId) throw new Error(`Could not resolve a Domain ID for hostname "${host}".`);
    }

    // Fetch mapped URLs from CDN and write to sheet
    const mapped = getMappedDomainsFromCDN(domainId); // array of strings
    writeReportRow(domainId, mapped);

    const details = mapped.length
      ? `\n\nMapped URLs:\n- ${mapped.join('\n- ')}`
      : '\n\nNo mapped URLs found in CDN list.';
    ui.alert('Check Complete (CDN)', `Domain ID: ${domainId}\nFound ${mapped.length} mapped URL(s).${details}`, ui.ButtonSet.OK);

  } catch (e) {
    ui.alert('Error Checking Domain Group (CDN)', String(e && e.message ? e.message : e), ui.ButtonSet.OK);
  }
}

/***** GET mapped domains from the CDN domain-list.json *****/
function getMappedDomainsFromCDN(domainId) {
  const url = `${CDN_CONSENT_BASE}/${encodeURIComponent(domainId)}/domain-list.json`;
  const res = UrlFetchApp.fetch(url, {
    method: 'get',
    muteHttpExceptions: true
  });

  const code = res.getResponseCode();
  const body = res.getContentText();
  Logger.log(`GET ${url} → ${code}; body length=${(body || '').length}`);

  if (code === 404) {
    throw new Error('CDN list not found (HTTP 404). Double-check the Domain ID or that the site is published.');
  }
  if (code < 200 || code >= 300) {
    throw new Error(`Failed to fetch CDN list: HTTP ${code} ${body}`);
  }

  let json;
  try {
    json = JSON.parse(body);
  } catch (err) {
    throw new Error(`Could not parse CDN JSON: ${err}`);
  }

  if (!Array.isArray(json)) {
    throw new Error('Unexpected CDN response (expected an array).');
  }

  // Map array entries to strings:
  //  - ["example.com", "foo.com"]
  //  - [{ url: "example.com" }, { hostname: "foo.com" }, ...]
  const out = [];
  for (const item of json) {
    if (typeof item === 'string') {
      const v = String(item || '').trim();
      if (v) out.push(v);
    } else if (item && typeof item === 'object') {
      const v = String(
        item.url ??
        item.hostname ??
        item.domain ??
        item.name ??
        ''
      ).trim();
      if (v) out.push(v);
    }
  }

  // De-dup + tidy
  const seen = new Set();
  const cleaned = [];
  for (const v of out) {
    const norm = clean(v);
    if (norm && !seen.has(norm)) {
      seen.add(norm);
      cleaned.push(norm);
    }
  }
  return cleaned;
}

// Ensure the report sheet exists with headers, then append a row
function writeReportRow(domainId, mappedDomains) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sh = ss.getSheetByName(REPORT_SHEET);
  if (!sh) {
    sh = ss.insertSheet(REPORT_SHEET);
  }

  // Ensure headers
  const headerA = sh.getRange(1, 1).getValue();
  const headerB = sh.getRange(1, 2).getValue();
  if (headerA !== 'domainID' || headerB !== 'Mapped Domains') {
    sh.getRange(1, 1).setValue('domainID');
    sh.getRange(1, 2).setValue('Mapped Domains');
  }

  const row = sh.getLastRow() + 1;
  sh.getRange(row, 1).setValue(domainId);
  sh.getRange(row, 2).setValue(mappedDomains.join(', '));
}
