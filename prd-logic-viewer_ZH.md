---
name: prd-logic-viewer
description: 解析PRD文档（支持Word/Markdown/飞书文档），提取功能模块与逻辑描述，生成带右键"查看逻辑"功能的交互原型HTML；也支持往已有原型HTML注入右键菜单功能。当用户需要从PRD生成可交互原型、给原型添加逻辑查看功能、PRD转demo、产品文档可视化等场景时使用。
---

# PRD逻辑穿透

从产品文档到可交互原型的桥梁——右键即可查看每个功能模块背后的PRD逻辑。

## 两种工作模式

### 模式A：从PRD生成原型
输入PRD文档 → AI解析提取模块逻辑 → 生成带右键"查看逻辑"功能的完整交互原型HTML

### 模式B：往已有原型注入右键功能
输入已有原型HTML + PRD文档 → AI匹配功能元素与PRD逻辑 → 注入右键菜单功能

## 工作流程

### 第1步：获取PRD内容

根据输入来源读取PRD文档：
- **Markdown文件**：直接用 read_file 读取
- **Word文档(.docx)**：用 parse_file 解析提取文本
- **飞书文档**：用 lark_cli 技能读取文档内容
- **用户直接粘贴**：直接使用粘贴内容

如果用户提供了PRD文件路径，先判断文件类型再选择对应解析方式。

### 第2步：解析PRD，提取模块与逻辑

仔细阅读PRD全文，提取出所有可交互的功能模块。对每个模块记录：

```
模块ID: 唯一标识（如 btn_submit, module_search）
模块名称: 界面上显示的名称
模块类型: button | input | form | tab | card | list | dropdown | switch | modal | table | chart | filter | nav
逻辑描述: 该模块在PRD中的完整逻辑说明（包括校验规则、触发条件、数据流向、异常处理等）
```

提取原则：
- 以PRD中的功能点为单位，不要拆得太碎也不要太粗
- 逻辑描述要保留原文关键信息，不要过度精简
- 如果PRD中有流程图/状态机描述，转化为文字描述记录在逻辑中
- 一个模块可能对应PRD中多个段落的描述，都要收录

### 第3步：生成原型（模式A）

根据PRD描述生成完整的交互原型HTML页面，严格使用下方「完整HTML模板」生成。生成时：

1. 替换 `{{PAGE_TITLE}}` 为PRD中的系统/页面名称
2. 替换 `{{PROTOTYPE_CONTENT}}` 为根据PRD生成的UI组件HTML
3. 替换 `{{LOGIC_DATA_JSON}}` 为JSON格式的逻辑数据对象

为每个功能元素添加 `data-logic-id="模块ID"` 属性，在 logicData 中注册对应逻辑文本。

### 第4步：注入已有原型（模式B）

1. 读取已有HTML原型文件
2. 识别其中的交互元素（button、input、a、select等带文字或语义的元素）
3. 将PRD提取的模块与原型中的元素进行匹配
4. 为匹配到的元素添加 `data-logic-id` 属性
5. 在HTML的 `</body>` 前注入右键菜单的CSS和JS代码（见下方「注入代码片段」）
6. 注入 logicData 对象

匹配策略：优先按元素文字内容与模块名称匹配，其次按元素ID/class语义匹配。

### 第5步：输出产物

- 生成的HTML文件保存到用户指定路径，默认保存为 `prd-prototype-{timestamp}.html`
- 告知用户：用浏览器打开即可使用，右键点击任意功能元素查看PRD逻辑

---

## 完整HTML模板

模式A直接基于此模板生成，替换三个占位符即可：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>PRD逻辑穿透 - {{PAGE_TITLE}}</title>
<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, "PingFang SC", "Microsoft YaHei", sans-serif; background: #f5f7fa; color: #333; line-height: 1.6; }

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

#logic-context-menu { position: fixed; z-index: 10000; background: #fff; border-radius: 8px; box-shadow: 0 6px 20px rgba(0,0,0,0.15); padding: 6px 0; min-width: 160px; display: none; }
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
  <span class="badge">PRD逻辑穿透</span>
</div>

<div class="prototype-container">
  {{PROTOTYPE_CONTENT}}
</div>

<div id="logic-context-menu">
  <div class="menu-item" id="menu-view-logic">
    <span class="icon">📋</span>查看逻辑
  </div>
</div>

<div id="logic-modal-overlay">
  <div id="logic-modal">
    <div id="logic-modal-header">
      <h3 id="logic-modal-title">逻辑详情</h3>
      <button id="logic-modal-close">✕</button>
    </div>
    <div id="logic-modal-body"></div>
    <div id="logic-modal-footer">
      <span class="source-link">来源：PRD文档</span>
    </div>
  </div>
</div>

<div class="help-tip">
  <span class="key">右键</span>点击功能元素查看PRD逻辑
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

## 注入代码片段（模式B使用）

往已有HTML注入时，在 `</body>` 前插入以下代码。其中 CSS 部分加入 `<style>` 标签，JS 部分加入 `<script>` 标签：

**需注入的CSS**（`[data-logic-id]` hover样式 + 右键菜单 + 弹窗样式）：

```css
[data-logic-id] { position: relative; transition: outline 0.15s; }
[data-logic-id]:hover { outline: 2px dashed rgba(22,119,255,0.35); outline-offset: 2px; border-radius: 4px; cursor: context-menu; }
#logic-context-menu { position: fixed; z-index: 10000; background: #fff; border-radius: 8px; box-shadow: 0 6px 20px rgba(0,0,0,0.15); padding: 6px 0; min-width: 160px; display: none; }
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

**需注入的HTML**（菜单 + 弹窗 + 提示，加在 `</body>` 前）：

```html
<div id="logic-context-menu">
  <div class="menu-item" id="menu-view-logic">
    <span class="icon">📋</span>查看逻辑
  </div>
</div>
<div id="logic-modal-overlay">
  <div id="logic-modal">
    <div id="logic-modal-header">
      <h3 id="logic-modal-title">逻辑详情</h3>
      <button id="logic-modal-close">✕</button>
    </div>
    <div id="logic-modal-body"></div>
    <div id="logic-modal-footer">
      <span class="source-link">来源：PRD文档</span>
    </div>
  </div>
</div>
<div class="help-tip">
  <span class="key">右键</span>点击功能元素查看PRD逻辑
</div>
```

**需注入的JS**（加在 `</body>` 前，`logicData` 内容根据PRD填充）：

```javascript
const logicData = {/* 根据PRD提取的模块逻辑，格式: { "模块ID": { name: "名称", logic: "逻辑描述" } } */};

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

## UI组件样式参考

生成原型内容时可使用以下CSS类，确保视觉统一：

| 组件 | CSS类 | 用途 |
|------|-------|------|
| 卡片 | `.proto-card` | 内容容器，带圆角阴影 |
| 卡片标题 | `.proto-card h2` | 左侧蓝色竖线标题 |
| 主按钮 | `.proto-btn.primary` | 主要操作按钮 |
| 次按钮 | `.proto-btn` | 次要操作按钮 |
| 危险按钮 | `.proto-btn.danger` | 删除等危险操作 |
| 输入框 | `.proto-input` | 文本输入 |
| 下拉框 | `.proto-select` | 选择器 |
| 表格 | `.proto-table` | 数据表格 |
| 标签页 | `.proto-tabs` + `.proto-tab` | 页签导航 |
| 开关 | `.proto-switch` | 布尔切换 |
| 标签 | `.proto-tag.blue/green/orange/red` | 状态标签 |
| 筛选栏 | `.proto-filter-bar` | 筛选条件区 |
| 统计卡 | `.proto-stat` | 数据统计展示 |

---

## 注意事项

- HTML必须自包含（单文件，内联CSS和JS），无需额外依赖即可在浏览器打开
- UI风格：简洁专业，浅色主题，适合产品演示
- 弹窗中的逻辑描述保持可读性，适当使用加粗、列表等Markdown格式
- 原型中的文字、按钮等尽量还原PRD描述的真实内容，不用占位符
- 如果PRD内容过长，优先确保核心功能模块的完整性
- 生成原型不是追求视觉还原度，而是让逻辑可查可追溯
- 如果已有原型中某些元素无法匹配到PRD逻辑，列出未匹配项让用户确认
- 飞书文档读取需要先确认 lark_cli 技能已加载且有权限

---

---

