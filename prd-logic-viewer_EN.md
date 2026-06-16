---
name: prd-logic-viewer
description: Parse PRD documents (Word/Markdown/Feishu Docs), extract functional modules and logic descriptions, generate interactive prototype HTML with right-click "View Logic" feature; also supports injecting right-click context menu into existing prototype HTML. Use when users need to generate interactive prototypes from PRD, add logic viewing to prototypes, convert PRD to demos, or visualize product documents.
---

# PRD Logic Viewer

Bridge the gap between product documents and interactive prototypes — right-click any element to view its underlying PRD logic.

## Two Working Modes

### Mode A: Generate Prototype from PRD
Input PRD document → AI parses and extracts module logic → Generate interactive prototype HTML with right-click "View Logic" feature

### Mode B: Inject Right-Click Feature into Existing Prototype
Input existing prototype HTML + PRD document → AI matches functional elements with PRD logic → Inject right-click context menu

## Workflow

### Step 1: Retrieve PRD Content

Read the PRD document based on the input source:
- **Markdown file**: Read directly with read_file
- **Word document (.docx)**: Parse and extract text with parse_file
- **Feishu document**: Read document content using lark_cli skill
- **User pasted content**: Use directly

If the user provides a PRD file path, determine the file type first, then choose the appropriate parsing method.

### Step 2: Parse PRD, Extract Modules and Logic

Carefully read the entire PRD and extract all interactive functional modules. For each module, record:

```
Module ID: Unique identifier (e.g., btn_submit, module_search)
Module Name: Display name on the interface
Module Type: button | input | form | tab | card | list | dropdown | switch | modal | table | chart | filter | nav
Logic Description: Complete logic specification from the PRD (including validation rules, trigger conditions, data flow, error handling, etc.)
```

Extraction principles:
- Organize by functional points in the PRD — not too granular, not too coarse
- Preserve key information from the original text in logic descriptions — do not oversimplify
- If the PRD contains flowchart/state machine descriptions, convert them to text descriptions in the logic
- A single module may correspond to multiple paragraphs in the PRD — include all of them

### Step 3: Generate Prototype (Mode A)

Generate a complete interactive prototype HTML page based on the PRD description, strictly using the "Complete HTML Template" below. During generation:

1. Replace `{{PAGE_TITLE}}` with the system/page name from the PRD
2. Replace `{{PROTOTYPE_CONTENT}}` with the UI component HTML generated from the PRD
3. Replace `{{LOGIC_DATA_JSON}}` with the logic data object in JSON format

Add a `data-logic-id="module_id"` attribute to each functional element, and register the corresponding logic text in logicData.

### Step 4: Inject into Existing Prototype (Mode B)

1. Read the existing HTML prototype file
2. Identify interactive elements (button, input, a, select, etc. with text or semantic meaning)
3. Match the modules extracted from the PRD with elements in the prototype
4. Add `data-logic-id` attributes to matched elements
5. Inject the right-click menu CSS and JS code before `</body>` in the HTML (see "Injection Code Snippets" below)
6. Inject the logicData object

Matching strategy: Prioritize matching by element text content with module name, then by element ID/class semantics.

### Step 5: Output

- Save the generated HTML file to the user-specified path, default: `prd-prototype-{timestamp}.html`
- Inform the user: Open in browser to use, right-click any functional element to view PRD logic

---

## Complete HTML Template

Mode A generates directly from this template — just replace the three placeholders:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>PRD Logic Viewer - {{PAGE_TITLE}}</title>
<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif; background: #f5f7fa; color: #333; line-height: 1.6; }

.prototype-header { background: #fff; border-bottom: 1px solid #e4e7ed; padding: 16px 32px; display: flex; align-items: center; gap: 12px; }
.prototype-header h1 { font-size: 18px; font-weight: 600; color: #1a1a1a; }
.prototype-header .badge { background: #e8f4ff; color: #1677ff; font-size: 12px; padding: 2px 8px; border-radius: 4px; }
.prototype-container { max-width: 1200px; margin: 24px auto; padding: 0 24px; }

.proto-card { background: #fff; border-radius: 8px; box-shadow: 0 1px 4px rgba(0,0,0,0.08); padding: 24px; margin-bottom: 20px; }
.proto-card h2 { font-size: 16px; font-weight: 600; margin-bottom: 16px; color: #1a1a1a; border-left: 3px solid #1677ff; padding-left: 10px; }
.proto-btn { display: inline-block; padding: 8px 20px; border-radius: 6px; border: 1px solid #d9d9d9; background: #fff; cursor: pointer; font-size: 14px; color: #333; transition: all 0.2s; margin: 4px; }
.proto-btn.primary { background: #1677ff; color: #fff; border-color: #1677ff; }
.proto-btn.danger { background: #ff4d4f; color: #fff; border-color: #ff4d4f; }
.proto-btn:hover { opacity: 0.85; }
.proto-input { padding: 8px 12px; border: 1px solid #d9d9d9; border-radius: 6px; font-size: 14px; outline: none; width: 240px; }
.proto-input:focus { border-color: #1677ff; box-shadow: 0 0 0 2px rgba(22,119,255,0.1); }
.proto-select { padding: 8px 12px; border: 1px solid #d9d9d9; border-radius: 6px; font-size: 14px; outline: none; background: #fff; min-width: 160px; }
.proto-table { width: 100%; border-collapse: collapse; margin: 12px 0; }
.proto-table th, .proto-table td { padding: 10px 14px; text-align: left; border-bottom: 1px solid #f0f0f0; font-size: 14px; }
.proto-table th { background: #fafafa; font-weight: 600; color: #666; }
.proto-tabs { display: flex; border-bottom: 1px solid #e4e7ed; margin-bottom: 16px; }
.proto-tab { padding: 8px 20px; cursor: pointer; font-size: 14px; color: #666; border-bottom: 2px solid transparent; }
.proto-tab.active { color: #1677ff; border-bottom-color: #1677ff; }
.proto-switch { display: inline-flex; align-items: center; gap: 8px; font-size: 14px; cursor: pointer; }
.proto-switch .track { width: 40px; height: 22px; background: #d9d9d9; border-radius: 11px; position: relative; transition: 0.3s; }
.proto-switch .track::after { content: ''; width: 18px; height: 18px; background: #fff; border-radius: 50%; position: absolute; top: 2px; left: 2px; transition: 0.3s; }
.proto-switch.on .track { background: #1677ff; }
.proto-switch.on .track::after { left: 20px; }
.proto-tag { display: inline-block; padding: 2px 8px; border-radius: 4px; font-size: 12px; }
.proto-tag.blue { background: #e8f4ff; color: #1677ff; }
.proto-tag.green { background: #e8ffe8; color: #52c41a; }
.proto-tag.orange { background: #fff7e6; color: #fa8c16; }
.proto-tag.red { background: #fff2f0; color: #ff4d4f; }
.proto-filter-bar { display: flex; gap: 12px; align-items: center; flex-wrap: wrap; margin-bottom: 16px; }
.proto-stat { display: flex; gap: 24px; margin-bottom: 20px; }
.proto-stat-item { text-align: center; }
.proto-stat-item .num { font-size: 28px; font-weight: 700; color: #1677ff; }
.proto-stat-item .label { font-size: 12px; color: #999; }

[data-logic-id] { position: relative; transition: outline 0.15s; }
[data-logic-id]:hover { outline: 2px dashed rgba(22,119,255,0.35); outline-offset: 2px; border-radius: 4px; cursor: context-menu; }

#logic-context-menu { position: fixed; z-index: 10000; background: #fff; border-radius: 8px; box-shadow: 0 6px 20px rgba(0,0,0,0.15); padding: 6px 0; min-width: 180px; display: none; }
#logic-context-menu .menu-item { padding: 8px 16px; cursor: pointer; font-size: 14px; color: #333; display: flex; align-items: center; gap: 8px; transition: background 0.15s; }
#logic-context-menu .menu-item:hover { background: #f5f5f5; }
#logic-context-menu .menu-item .icon { font-size: 16px; }

#logic-modal-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.4); z-index: 10001; display: none; justify-content: center; align-items: center; }
#logic-modal-overlay.show { display: flex; }
#logic-modal { background: #fff; border-radius: 12px; width: 90%; max-width: 640px; max-height: 80vh; display: flex; flex-direction: column; box-shadow: 0 12px 40px rgba(0,0,0,0.2); }
#logic-modal-header { padding: 20px 24px 16px; border-bottom: 1px solid #f0f0f0; display: flex; justify-content: space-between; align-items: center; }
#logic-modal-header h3 { font-size: 16px; font-weight: 600; color: #1a1a1a; }
#logic-modal-close { width: 32px; height: 32px; border-radius: 50%; border: none; background: #f5f5f5; cursor: pointer; font-size: 18px; display: flex; align-items: center; justify-content: center; color: #999; transition: 0.2s; }
#logic-modal-close:hover { background: #e8e8e8; color: #333; }
#logic-modal-body { padding: 20px 24px; overflow-y: auto; flex: 1; }
#logic-modal-body h1,#logic-modal-body h2,#logic-modal-body h3,#logic-modal-body h4 { margin: 16px 0 8px; color: #1a1a1a; }
#logic-modal-body h1 { font-size: 18px; } #logic-modal-body h2 { font-size: 16px; } #logic-modal-body h3 { font-size: 15px; }
#logic-modal-body p { margin: 8px 0; font-size: 14px; color: #555; }
#logic-modal-body ul, #logic-modal-body ol { margin: 8px 0; padding-left: 20px; font-size: 14px; color: #555; }
#logic-modal-body li { margin: 4px 0; }
#logic-modal-body strong { color: #1a1a1a; }
#logic-modal-body code { background: #f5f5f5; padding: 2px 6px; border-radius: 3px; font-size: 13px; }
#logic-modal-footer { padding: 12px 24px; border-top: 1px solid #f0f0f0; display: flex; justify-content: flex-end; gap: 8px; }
#logic-modal-footer .source-link { font-size: 12px; color: #999; align-self: center; }

.help-tip { position: fixed; bottom: 20px; right: 20px; background: #fff; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); padding: 12px 16px; font-size: 13px; color: #666; z-index: 9999; display: flex; align-items: center; gap: 8px; }
.help-tip .key { background: #f0f0f0; padding: 2px 8px; border-radius: 4px; font-size: 12px; border: 1px solid #ddd; }
</style>
</head>
<body>

<div class="prototype-header">
  <h1>{{PAGE_TITLE}}</h1>
  <span class="badge">PRD Logic Viewer</span>
</div>

<div class="prototype-container">
  {{PROTOTYPE_CONTENT}}
</div>

<div id="logic-context-menu">
  <div class="menu-item" id="menu-view-logic">
    <span class="icon">📋</span>View Logic
  </div>
</div>

<div id="logic-modal-overlay">
  <div id="logic-modal">
    <div id="logic-modal-header">
      <h3 id="logic-modal-title">Logic Details</h3>
      <button id="logic-modal-close">✕</button>
    </div>
    <div id="logic-modal-body"></div>
    <div id="logic-modal-footer">
      <span class="source-link">Source: PRD Document</span>
    </div>
  </div>
</div>

<div class="help-tip">
  <span class="key">Right-click</span> on functional elements to view PRD logic
</div>

<script>
const logicData = {{LOGIC_DATA_JSON}};

const contextMenu = document.getElementById('logic-context-menu');
const menuViewLogic = document.getElementById('menu-view-logic');
let currentLogicId = null;

document.addEventListener('contextmenu', function(e) {
  const target = e.target.closest('[data-logic-id]');
  if (target) {
    e.preventDefault();
    currentLogicId = target.getAttribute('data-logic-id');
    if (logicData[currentLogicId]) {
      contextMenu.style.display = 'block';
      contextMenu.style.left = e.clientX + 'px';
      contextMenu.style.top = e.clientY + 'px';
      const rect = contextMenu.getBoundingClientRect();
      if (rect.right > window.innerWidth) contextMenu.style.left = (e.clientX - rect.width) + 'px';
      if (rect.bottom > window.innerHeight) contextMenu.style.top = (e.clientY - rect.height) + 'px';
    }
  } else {
    contextMenu.style.display = 'none';
  }
});

document.addEventListener('click', function() {
  contextMenu.style.display = 'none';
});

menuViewLogic.addEventListener('click', function() {
  if (!currentLogicId || !logicData[currentLogicId]) return;
  const data = logicData[currentLogicId];
  document.getElementById('logic-modal-title').textContent = data.name;
  document.getElementById('logic-modal-body').innerHTML = renderMarkdown(data.logic);
  document.getElementById('logic-modal-overlay').classList.add('show');
  contextMenu.style.display = 'none';
});

const overlay = document.getElementById('logic-modal-overlay');
document.getElementById('logic-modal-close').addEventListener('click', function() {
  overlay.classList.remove('show');
});
overlay.addEventListener('click', function(e) {
  if (e.target === overlay) overlay.classList.remove('show');
});
document.addEventListener('keydown', function(e) {
  if (e.key === 'Escape') overlay.classList.remove('show');
});

function renderMarkdown(text) {
  if (!text) return '';
  let html = text
    .replace(/^### (.+)$/gm, '<h3>$1</h3>')
    .replace(/^## (.+)$/gm, '<h2>$1</h2>')
    .replace(/^# (.+)$/gm, '<h1>$1</h1>')
    .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')
    .replace(/`(.+?)`/g, '<code>$1</code>')
    .replace(/^- (.+)$/gm, '<li>$1</li>')
    .replace(/^\d+\. (.+)$/gm, '<li>$1</li>')
    .replace(/\n\n/g, '</p><p>')
    .replace(/\n/g, '<br>');
  html = html.replace(/(<li>.*<\/li>)/gs, function(match) {
    if (!match.startsWith('<ul>')) return '<ul>' + match + '</ul>';
    return match;
  });
  return '<p>' + html + '</p>';
}
</script>

</body>
</html>
```

---

## Injection Code Snippets (for Mode B)

When injecting into an existing HTML file, insert the following code before `</body>`. Wrap CSS in `<style>` tags and JS in `<script>` tags:

**CSS to inject** (`[data-logic-id]` hover styles + context menu + modal styles):

```css
[data-logic-id] { position: relative; transition: outline 0.15s; }
[data-logic-id]:hover { outline: 2px dashed rgba(22,119,255,0.35); outline-offset: 2px; border-radius: 4px; cursor: context-menu; }
#logic-context-menu { position: fixed; z-index: 10000; background: #fff; border-radius: 8px; box-shadow: 0 6px 20px rgba(0,0,0,0.15); padding: 6px 0; min-width: 180px; display: none; }
#logic-context-menu .menu-item { padding: 8px 16px; cursor: pointer; font-size: 14px; color: #333; display: flex; align-items: center; gap: 8px; transition: background 0.15s; }
#logic-context-menu .menu-item:hover { background: #f5f5f5; }
#logic-context-menu .menu-item .icon { font-size: 16px; }
#logic-modal-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.4); z-index: 10001; display: none; justify-content: center; align-items: center; }
#logic-modal-overlay.show { display: flex; }
#logic-modal { background: #fff; border-radius: 12px; width: 90%; max-width: 640px; max-height: 80vh; display: flex; flex-direction: column; box-shadow: 0 12px 40px rgba(0,0,0,0.2); }
#logic-modal-header { padding: 20px 24px 16px; border-bottom: 1px solid #f0f0f0; display: flex; justify-content: space-between; align-items: center; }
#logic-modal-header h3 { font-size: 16px; font-weight: 600; color: #1a1a1a; }
#logic-modal-close { width: 32px; height: 32px; border-radius: 50%; border: none; background: #f5f5f5; cursor: pointer; font-size: 18px; display: flex; align-items: center; justify-content: center; color: #999; transition: 0.2s; }
#logic-modal-close:hover { background: #e8e8e8; color: #333; }
#logic-modal-body { padding: 20px 24px; overflow-y: auto; flex: 1; }
#logic-modal-body h1,#logic-modal-body h2,#logic-modal-body h3,#logic-modal-body h4 { margin: 16px 0 8px; color: #1a1a1a; }
#logic-modal-body h1 { font-size: 18px; } #logic-modal-body h2 { font-size: 16px; } #logic-modal-body h3 { font-size: 15px; }
#logic-modal-body p { margin: 8px 0; font-size: 14px; color: #555; }
#logic-modal-body ul, #logic-modal-body ol { margin: 8px 0; padding-left: 20px; font-size: 14px; color: #555; }
#logic-modal-body li { margin: 4px 0; }
#logic-modal-body strong { color: #1a1a1a; }
#logic-modal-body code { background: #f5f5f5; padding: 2px 6px; border-radius: 3px; font-size: 13px; }
#logic-modal-footer { padding: 12px 24px; border-top: 1px solid #f0f0f0; display: flex; justify-content: flex-end; gap: 8px; }
#logic-modal-footer .source-link { font-size: 12px; color: #999; align-self: center; }
.help-tip { position: fixed; bottom: 20px; right: 20px; background: #fff; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); padding: 12px 16px; font-size: 13px; color: #666; z-index: 9999; display: flex; align-items: center; gap: 8px; }
.help-tip .key { background: #f0f0f0; padding: 2px 8px; border-radius: 4px; font-size: 12px; border: 1px solid #ddd; }
```

**HTML to inject** (menu + modal + tip, added before `</body>`):

```html
<div id="logic-context-menu">
  <div class="menu-item" id="menu-view-logic">
    <span class="icon">📋</span>View Logic
  </div>
</div>
<div id="logic-modal-overlay">
  <div id="logic-modal">
    <div id="logic-modal-header">
      <h3 id="logic-modal-title">Logic Details</h3>
      <button id="logic-modal-close">✕</button>
    </div>
    <div id="logic-modal-body"></div>
    <div id="logic-modal-footer">
      <span class="source-link">Source: PRD Document</span>
    </div>
  </div>
</div>
<div class="help-tip">
  <span class="key">Right-click</span> on functional elements to view PRD logic
</div>
```

**JS to inject** (added before `</body>`, `logicData` content filled from PRD):

```javascript
const logicData = {/* Module logic extracted from PRD, format: { "module_id": { name: "Name", logic: "Logic description" } } */};

const contextMenu = document.getElementById('logic-context-menu');
const menuViewLogic = document.getElementById('menu-view-logic');
let currentLogicId = null;

document.addEventListener('contextmenu', function(e) {
  const target = e.target.closest('[data-logic-id]');
  if (target) {
    e.preventDefault();
    currentLogicId = target.getAttribute('data-logic-id');
    if (logicData[currentLogicId]) {
      contextMenu.style.display = 'block';
      contextMenu.style.left = e.clientX + 'px';
      contextMenu.style.top = e.clientY + 'px';
      const rect = contextMenu.getBoundingClientRect();
      if (rect.right > window.innerWidth) contextMenu.style.left = (e.clientX - rect.width) + 'px';
      if (rect.bottom > window.innerHeight) contextMenu.style.top = (e.clientY - rect.height) + 'px';
    }
  } else {
    contextMenu.style.display = 'none';
  }
});

document.addEventListener('click', function() { contextMenu.style.display = 'none'; });

menuViewLogic.addEventListener('click', function() {
  if (!currentLogicId || !logicData[currentLogicId]) return;
  const data = logicData[currentLogicId];
  document.getElementById('logic-modal-title').textContent = data.name;
  document.getElementById('logic-modal-body').innerHTML = renderMarkdown(data.logic);
  document.getElementById('logic-modal-overlay').classList.add('show');
  contextMenu.style.display = 'none';
});

const overlay = document.getElementById('logic-modal-overlay');
document.getElementById('logic-modal-close').addEventListener('click', function() { overlay.classList.remove('show'); });
overlay.addEventListener('click', function(e) { if (e.target === overlay) overlay.classList.remove('show'); });
document.addEventListener('keydown', function(e) { if (e.key === 'Escape') overlay.classList.remove('show'); });

function renderMarkdown(text) {
  if (!text) return '';
  let html = text
    .replace(/^### (.+)$/gm, '<h3>$1</h3>').replace(/^## (.+)$/gm, '<h2>$1</h2>').replace(/^# (.+)$/gm, '<h1>$1</h1>')
    .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>').replace(/`(.+?)`/g, '<code>$1</code>')
    .replace(/^- (.+)$/gm, '<li>$1</li>').replace(/^\d+\. (.+)$/gm, '<li>$1</li>')
    .replace(/\n\n/g, '</p><p>').replace(/\n/g, '<br>');
  html = html.replace(/(<li>.*<\/li>)/gs, function(m) { return m.startsWith('<ul>') ? m : '<ul>' + m + '</ul>'; });
  return '<p>' + html + '</p>';
}
```

---

## UI Component Style Reference

Use the following CSS classes when generating prototype content to ensure visual consistency:

| Component | CSS Class | Usage |
|-----------|-----------|-------|
| Card | `.proto-card` | Content container with rounded corners and shadow |
| Card Title | `.proto-card h2` | Left blue bar title |
| Primary Button | `.proto-btn.primary` | Primary action button |
| Default Button | `.proto-btn` | Secondary action button |
| Danger Button | `.proto-btn.danger` | Destructive action button |
| Input | `.proto-input` | Text input field |
| Select | `.proto-select` | Dropdown selector |
| Table | `.proto-table` | Data table |
| Tabs | `.proto-tabs` + `.proto-tab` | Tab navigation |
| Switch | `.proto-switch` | Boolean toggle |
| Tag | `.proto-tag.blue/green/orange/red` | Status tags |
| Filter Bar | `.proto-filter-bar` | Filter condition area |
| Stat Card | `.proto-stat` | Data statistics display |

---

## Notes

- HTML must be self-contained (single file, inline CSS and JS) — no external dependencies required to open in browser
- UI style: clean and professional, light theme, suitable for product demos
- Keep logic descriptions in modals readable — use bold, lists, and other Markdown formatting appropriately
- Use real content from the PRD for text, buttons, etc. — avoid placeholder text
- If the PRD is lengthy, prioritize completeness of core functional modules
- The goal is not pixel-perfect visual fidelity, but making logic traceable and accessible
- If some elements in an existing prototype cannot be matched to PRD logic, list unmatched items for user confirmation
- Reading Feishu documents requires the lark_cli skill to be loaded with proper permissions

---

---
