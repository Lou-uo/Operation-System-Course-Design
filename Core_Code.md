# 操作系统课设 —— 核心代码详解

> 本文档摘录 `simulator.html` 中各功能模块的核心代码，并附有详细注释。

---

## 目录

### 一、文件系统功能
1. [多级树形目录管理](#1-多级树形目录管理)
2. [文件全生命周期操作](#2-文件全生命周期操作)
3. [FCB 文件控制块管理](#3-fcb-文件控制块管理)
4. [文件保护机制](#4-文件保护机制)
5. [连续存储分配](#5-连续存储分配)

### 二、磁盘存储管理功能
6. [模拟磁盘模型](#6-模拟磁盘模型)
7. [空闲块管理（位示图+空闲盘区链）](#7-空闲块管理（位示图+空闲盘区链）)
8. [空间分配算法（First-Fit/Best-Fit）](#8-空间分配算法（First-Fit/Best-Fit）)
9. [磁盘状态可视化](#9-磁盘状态可视化)

### 三、内存缓冲区管理功能
10. [缓冲池模型](#10-缓冲池模型)
11. [LRU 页面置换算法](#11-lru-页面置换算法)
12. [脏页机制与性能统计](#12-脏页机制与性能统计)
13. [加载动画模拟磁盘 I/O](#13-加载动画模拟磁盘-io)
14. [五种页面置换算法对比](#14-五种页面置换算法对比)

### 四、进程同步与通信功能
15. [互斥同步（mutex + 等待队列）](#15-互斥同步（mutex + 等待队列）)
16. [文件访问冲突演示](#16-文件访问冲突演示)
17. [生产者-消费者模型](#17-生产者-消费者模型)
18. [消息队列通信](#18-消息队列通信)
19. [共享内存通信](#19-共享内存通信)
20. [管道通信（环形缓冲区）](#20-管道通信（环形缓冲区）)

### 五、调度与死锁算法功能
21. [四种作业调度算法](#21-四种作业调度算法)
22. [甘特图可视化与统计](#22-甘特图可视化与统计)
23. [银行家算法（安全序列检测）](#23-银行家算法（安全序列检测）)
24. [资源分配图可视化](#24-资源分配图可视化)
25. [打印作业调度](#25-打印作业调度)

### 六、交互与可视化功能
26. [Windows 风格图形界面](#26-windows-风格图形界面)
27. [命令行终端（9条命令）](#27-命令行终端（9条命令）)
28. [右键菜单与快捷键](#28-右键菜单与快捷键)
29. [一键演示模式](#29-一键演示模式)

---

## 一、文件系统功能

### 1. 多级树形目录管理

```javascript
// 全局状态：文件夹路径列表（支持无限层级嵌套）
let folders = [];
let currentPath = 'user1';
let navHistory = [], navHistoryIdx = -1;

/**
 * 递归渲染导航树
 * 支持无限层级嵌套，通过 depth 控制缩进
 */
function renderNavTree() {
  const el = $('navTree');
  let html = `<div class="tree-node${!currentPath?' selected':''}" onclick="navigateTo('')">
    <div class="tree-row" style="--depth:0">
      <span class="tree-arrow expandable expanded">▶</span>
      <span class="tree-icon">🖥️</span><span class="tree-name">此电脑</span>
    </div>
  </div>`;
  
  // 递归渲染子文件夹
  function renderSubFolders(parentName, depth) {
    const subs = folders.filter(fp => 
      fp !== parentName && 
      fp.startsWith(parentName+'/') && 
      fp.split('/').length === parentName.split('/').length + 1
    ).sort();
    subs.forEach(sub => {
      const hasSubs = folders.some(fp => 
        fp.startsWith(sub+'/') && 
        fp.split('/').length === sub.split('/').length + 1
      );
      html += `<div class="tree-node${currentPath===sub?' selected':''}" data-folder="${sub}"
        onclick="navigateTo('${sub}')">
        <div class="tree-row" style="--depth:${depth}">
          <span class="tree-arrow${hasSubs?' expandable':''}" 
            onclick="event.stopPropagation();toggleTreeNode(this)">▶</span>
          <span class="tree-icon">📂</span><span class="tree-name">${sub.split('/').pop()}</span>
        </div>`;
      if (hasSubs) {
        html += `<div class="tree-children" id="tc${sub.replace(/[\/\s]/g,'_')}">`;
        renderSubFolders(sub, depth+1);
        html += `</div>`;
      }
      html += `</div>`;
    });
  }
  el.innerHTML = html;
}

/**
 * 展开/收起目录树节点
 */
function toggleTreeNode(arrow) {
  arrow.classList.toggle('expanded');
  const node = arrow.closest('.tree-node');
  const children = node.querySelector('.tree-children');
  if (children) children.classList.toggle('collapsed');
}

/**
 * 新建文件夹（支持嵌套）
 */
function cmdNewFolder() {
  const name = prompt('请输入文件夹名称:', '新建文件夹');
  if (!name || !name.trim()) return;
  const folder = name.trim();
  const parentPath = currentPath && currentPath !== '' ? currentPath : '';
  const fullPath = parentPath ? parentPath + '/' + folder : folder;
  
  if (folders.includes(fullPath)) {
    toast(`文件夹 ${fullPath} 已存在`, '⚠️');
    return;
  }
  folders.push(fullPath);
  // 确保父路径也存在
  if (parentPath && !folders.includes(parentPath)) folders.push(parentPath);
  
  currentPath = fullPath;
  navHistory.push(fullPath);
  navHistoryIdx = navHistory.length - 1;
  updateAddressBar();
  renderNavTree();
  renderFileList();
}
```

---

### 2. 文件全生命周期操作

```javascript
/**
 * 新建文件 —— 核心流程：
 *   1. 校验文件名唯一性
 *   2. 按 64B 拆分内容块
 *   3. 通过位示图分配连续物理块
 *   4. 创建 FCB 并加入目录表
 */
function _diskCreateFile() {
  const name = $('nfName').value.trim() || ('file' + (files.length + 1));
  const owner = $('nfOwner').value.trim() || currentPath || 'user1';
  const rawContent = $('nfContent').value;
  
  // 检查同名文件
  if (files.find(f => f.name === name && f.owner === owner)) {
    toast(`${name} 已在 ${owner} 中存在`, '⚠️');
    return;
  }
  
  // 按 BLOCK_SIZE(64B) 拆分内容
  let contentBlocks = [];
  if (rawContent) {
    for (let i = 0; i < rawContent.length; i += BLOCK_SIZE) {
      contentBlocks.push(rawContent.substring(i, i + BLOCK_SIZE));
    }
  }
  
  const size = Math.max(
    parseInt($('nfSize').value) || 3,
    contentBlocks.length || 1
  );
  
  // 分配连续物理块（调用位示图分配算法）
  const start = bitmapFindFree(size);
  if (start === -1) {
    log(`错误: 无足够连续空闲空间 (需要${size}块)`, 'err');
    toast('磁盘空间不足', '❌');
    return;
  }
  
  // 写入物理块
  for (let i = start; i < start + size; i++) {
    diskBlocks[i] = {
      type: 'data',
      file: name,
      content: i - start < contentBlocks.length ? contentBlocks[i - start] : ''
    };
  }
  
  // 创建 FCB
  const now = new Date();
  const time = `${now.getFullYear()}-${(now.getMonth() + 1)
    .toString().padStart(2, '0')}-${now.getDate().toString().padStart(2, '0')}`;
  
  files.push({
    name, owner, time,
    start, size,
    perm: 'rw-r--r--',
    content: contentBlocks
  });
  
  updateFreeAreas();
  renderAll();
}

/**
 * 分块查看文件内容
 */
function diskViewFile() {
  const name = selectedFile;
  const f = files.find(x => x.name === name);
  fileInUse = name;
  
  const el = $('fileViewContent');
  let html = `<div class="mb-8">
    <b class="color-accent">${f.name}</b>
    <span>物理块 ${f.start}~${f.start+f.size-1} | ${f.owner}</span>
  </div>`;
  
  // 按块显示内容
  for (let i = 0; i < f.size; i++) {
    const c = f.content[i] || '';
    const hex = (f.start + i).toString(16).toUpperCase().padStart(4, '0');
    html += `<div style="margin:4px 0;padding:6px 10px;background:var(--bg);border-radius:4px;
      border-left:3px solid ${c ? 'var(--blue)' : 'var(--border)'}">
      <span class="text-xs color-yellow">块${f.start+i} [0x${hex}]</span><br>
      <span>${c ? c.replace(/</g,'&lt;').replace(/>/g,'&gt;') : '(空)'}</span>
    </div>`;
  }
  el.innerHTML = html;
  openModal('fileViewModal');
  
  // 触发缓冲页动画加载
  bufLoadFileAnimated(f);
}

/**
 * 分块编辑文件内容
 */
function _diskEditFile() {
  const name = selectedFile;
  const f = files.find(x => x.name === name);
  fileInUse = name;
  
  let html = `<div class="mb-8 flex-between">
    <b class="color-accent">修改 ${f.name}</b><span class="f-inuse">使用中</span>
  </div>`;
  
  // 每块一个 textarea
  for (let i = 0; i < f.size; i++) {
    const c = f.content && f.content[i] ? f.content[i] : '';
    html += `<div class="mb-8">
      <span class="text-xs color-yellow">块${f.start+i}</span>
      <textarea id="eb${i}" rows="2" style="width:100%">${c}</textarea>
    </div>`;
  }
  $('fileEditContent').innerHTML = html;
  openModal('fileEditModal');
}

/**
 * 保存编辑内容
 */
function diskSaveEdit() {
  const name = editTargetFile || selectedFile;
  const f = files.find(x => x.name === name);
  
  for (let i = 0; i < f.size; i++) {
    const ta = $('eb' + i);
    if (ta) {
      if (!f.content) f.content = [];
      f.content[i] = ta.value;
      const bi = f.start + i;
      if (diskBlocks[bi]) diskBlocks[bi].content = ta.value;
    }
  }
  closeFileView();
}

/**
 * 文件重命名
 */
function cmdRename() {
  const row = document.querySelector(`.file-row[data-file="${selectedFile}"]`);
  const nameEl = row.querySelector('.f-name-text');
  const old = nameEl.textContent;
  
  const inp = document.createElement('input');
  inp.className = 'f-name-input';
  inp.value = old;
  nameEl.replaceWith(inp);
  inp.focus();
  inp.select();
  
  inp.onblur = inp.onkeydown = function(e) {
    if (e && e.key === 'Escape') {
      inp.replaceWith(nameEl);
      return;
    }
    if (e && e.key !== 'Enter') return;
    
    const nn = inp.value.trim();
    if (nn && nn !== old) {
      const f = files.find(ff => ff.name === selectedFile);
      if (f && !files.find(ff => ff.name === nn && ff.owner === f.owner)) {
        f.name = nn;
        // 更新磁盘块中的文件名引用
        for (let i = f.start; i < f.start + f.size; i++) {
          if (diskBlocks[i]) diskBlocks[i].file = nn;
        }
        selectedFile = nn;
        renderAll();
      } else {
        toast('该目录下已存在同名文件', '⚠️');
        inp.replaceWith(nameEl);
      }
    } else {
      inp.replaceWith(nameEl);
    }
  };
}
```

---

### 3. FCB 文件控制块管理

```javascript
/**
 * FCB（文件控制块）数据结构：
 * {
 *   name:     文件名,
 *   owner:    所属目录/用户,
 *   time:     创建时间,
 *   start:    起始物理块号,
 *   size:     占用块数,
 *   perm:     文件权限 (rw-r--r--),
 *   content:  [每个块的内容字符串]
 * }
 */
let files = [];

/**
 * 显示 FCB 详情弹窗
 */
function showFCB(name) {
  const f = files.find(x => x.name === name);
  if (!f) return;
  
  const usedBytes = f.content ? f.content.reduce(
    (s, c) => s + (c ? c.length : 0), 0
  ) : 0;
  
  let html = `
    <div class="fcb-row"><span class="fcb-key">📝 文件名</span><span class="fcb-val">${f.name}</span></div>
    <div class="fcb-row"><span class="fcb-key">👤 用户</span><span class="fcb-val">${f.owner}</span></div>
    <div class="fcb-row"><span class="fcb-key">📅 创建时间</span><span class="fcb-val">${f.time}</span></div>
    <div class="fcb-row"><span class="fcb-key">🔒 权限</span><span class="fcb-val">${f.perm}</span></div>
    <div class="fcb-row"><span class="fcb-key">📍 起始块</span><span class="fcb-val">${f.start}</span></div>
    <div class="fcb-row"><span class="fcb-key">📦 占用块数</span><span class="fcb-val">${f.size}</span></div>
    <div class="fcb-row"><span class="fcb-key">💾 数据量</span><span class="fcb-val">${usedBytes}B / ${f.size * BLOCK_SIZE}B</span></div>
    <div class="fcb-row"><span class="fcb-key">📐 分配方式</span><span class="fcb-val">连续分配</span></div>
    <div class="fcb-row"><span class="fcb-key">🔄 状态</span><span class="fcb-val">${fileInUse === f.name ? '🟡 使用中' : '🟢 空闲'}</span></div>
  `;
  
  // 数据块指针列表
  html += `<div class="mt-8 text-sm color-text2">数据块指针:</div><div class="fcb-blocks">`;
  for (let i = 0; i < f.size; i++) {
    html += `<span class="fcb-block-tag">块${f.start + i}</span>`;
  }
  html += `</div>`;
  
  $('fcbContent').innerHTML = html;
  openModal('fcbModal');
}
```

---

### 4. 文件保护机制

```javascript
let fileInUse = null;  // 当前正在使用的文件名

/**
 * 删除文件 —— 含文件保护机制
 * 
 * 文件保护逻辑：
 *   变量 fileInUse 记录当前正在查看/编辑的文件名
 *   若待删文件正在使用中，阻止删除操作
 */
function _diskDeleteFile() {
  const name = selectedFile;
  if (!name) {
    toast('请先选择文件', '⚠️');
    return;
  }
  
  // === 文件保护检查 ===
  if (fileInUse === name) {
    log(`⛔ 删除被阻止: 文件 ${name} 正在使用中`, 'err');
    toast(`文件 ${name} 正在使用中，无法删除`, '🔒');
    return;
  }
  
  const idx = files.findIndex(f => f.name === name);
  if (idx === -1) return;
  
  const f = files[idx];
  
  // 释放物理块
  for (let i = f.start; i < f.start + f.size; i++) {
    diskBlocks[i] = null;
  }
  
  // 清理缓冲区中该文件的页面
  bufPages.forEach((p, i) => {
    if (p && p.owner === name) bufPages[i] = null;
  });
  
  files.splice(idx, 1);
  selectedFile = null;
  
  updateFreeAreas();
  renderAll();
}

/**
 * 关闭文件查看/编辑 —— 解除文件保护
 */
function closeFileView() {
  fileInUse = null;
  editTargetFile = null;
  closeModal('fileViewModal');
  closeModal('fileEditModal');
  renderAll();
  log('已关闭文件查看，解除文件保护', 'sys');
}
```

---

### 5. 连续存储分配

```javascript
/**
 * 连续存储分配核心逻辑：
 *   1. 在空闲盘区链中查找满足 size 的连续空闲区域
 *   2. 将文件数据写入连续的物理块
 *   3. 更新空闲盘区链（切割或删除节点）
 */
function _diskCreateFile() {
  // ... 省略前置代码 ...
  
  // 分配连续物理块
  const start = bitmapFindFree(size);
  if (start === -1) {
    log(`错误: 无足够连续空闲空间 (需要${size}块)`, 'err');
    return;
  }
  
  // 将文件数据写入连续的物理块
  for (let i = start; i < start + size; i++) {
    diskBlocks[i] = {
      type: 'data',
      file: name,
      content: i - start < contentBlocks.length ? contentBlocks[i - start] : ''
    };
  }
  
  // 更新空闲盘区链（自动合并相邻空闲区）
  updateFreeAreas();
}
```

---

## 二、磁盘存储管理功能

### 6. 模拟磁盘模型

```javascript
// 磁盘常量：1024 个盘块，每块 64 字节
const DISK_SIZE = 1024, BLOCK_SIZE = 64;

/**
 * 磁盘块数组：null 表示空闲，否则存储 {type, file, content}
 * type: 'dir'（目录区）| 'meta'（元数据区）| 'data'（文件数据区）
 */
let diskBlocks = new Array(DISK_SIZE).fill(null);

/**
 * 磁盘初始化
 * 划分为系统区（目录、元数据）与数据区
 */
function diskInit() {
  // 前 8 块 —— MFD（主文件目录）
  for (let i = 0; i < 8; i++) {
    diskBlocks[i] = { type: 'dir', file: 'MFD' };
  }
  
  // 第 8~15 块 —— UFD（用户文件目录）
  for (let i = 8; i < 16; i++) {
    diskBlocks[i] = { type: 'dir', file: 'UFD' };
  }
  
  // 第 16~23 块 —— FreeTable（空闲盘区表元数据）
  for (let i = 16; i < 24; i++) {
    diskBlocks[i] = { type: 'meta', file: 'FreeTable' };
  }
  
  files = [];
  folders = ['user1', 'user2', 'system'];
  bufPages = new Array(BUF_SIZE).fill(null);
  
  // 自动创建系统应用程序
  createExeFile('printer.exe', 'system', 'PRINTER:打印作业调度程序');
  createExeFile('paint.exe', 'system', 'PAINT:画图程序');
  createExeFile('notepad.exe', 'system', 'NOTEPAD:文本编辑器');
  createExeFile('calc.exe', 'system', 'CALC:计算器');
  
  updateFreeAreas();
  renderAll();
}
```

---

### 7. 空闲块管理（位示图+空闲盘区链）

```javascript
// 空闲盘区链：记录连续的空闲区域
// [{start: 起始块号, count: 连续空闲块数}]
let freeAreas = [];

/**
 * 扫描整个磁盘块数组，构建空闲盘区链
 * 回收时自动合并相邻空闲区间
 */
function updateFreeAreas() {
  freeAreas = [];
  let start = -1, cnt = 0;
  
  for (let i = 0; i < DISK_SIZE; i++) {
    if (diskBlocks[i] === null) {
      // 当前块空闲
      if (start === -1) {
        start = i;   // 记录空闲区起始位置
        cnt = 1;
      } else {
        cnt++;       // 连续空闲，计数+1
      }
    } else {
      // 当前块已被占用，截断当前空闲区
      if (start !== -1) {
        freeAreas.push({ start, count: cnt });
        start = -1;
        cnt = 0;
      }
    }
  }
  // 处理末尾的空闲区
  if (start !== -1) {
    freeAreas.push({ start, count: cnt });
  }
}

/**
 * 位示图渲染 —— 64×16 网格展示磁盘状态
 */
function renderBitmap() {
  const el = $('bitmapGrid');
  let html = '';
  
  for (let i = 0; i < DISK_SIZE; i++) {
    const b = diskBlocks[i];
    let cls = 'bm-free';  // 默认空闲（绿色）
    
    if (b) {
      if (b.type === 'dir') cls = 'bm-dir';      // 目录区（紫色）
      else if (b.type === 'meta') cls = 'bm-meta'; // 元数据区（粉紫色）
      else cls = 'bm-used';                       // 文件数据区（红色）
    }
    
    html += `<div class="bitmap-cell ${cls}" 
      title="块${i}${b ? ' ' + b.file : ' 空闲'}" 
      onclick="diskCellClick(${i})"></div>`;
  }
  el.innerHTML = html;
}

/**
 * 空闲盘区链渲染 —— 链式可视化
 */
function renderFreeAreas() {
  const el = $('freeAreasContent');
  let html = '', totalFree = 0;
  
  // 链式结构可视化
  html += `<div style="display:flex;align-items:center;gap:4px;flex-wrap:wrap;
    margin-bottom:8px;padding:8px;background:var(--bg);border-radius:6px;
    font-size:11px;font-family:var(--font-mono)">`;
  
  freeAreas.forEach((a, i) => {
    totalFree += a.count;
    html += `<span style="background:rgba(22,198,12,.12);border:1px solid rgba(22,198,12,.25);
      border-radius:4px;padding:3px 8px;color:var(--green);font-weight:600">[${a.start}:${a.count}]</span>
      <span style="color:var(--text2)">→</span>`;
  });
  
  if (!freeAreas.length) {
    html += `<span style="color:var(--red)">(无空闲盘区)</span>`;
  } else {
    html += `<span style="color:var(--text2);font-weight:600">null</span>`;
  }
  html += `</div>`;
  
  // 统计信息
  html += `<div class="mb-8 text-sm">
    总空闲块: <b class="color-green">${totalFree}</b> / ${DISK_SIZE}
    (<b>${(totalFree/DISK_SIZE*100).toFixed(1)}%</b>)
    | 空闲链节点数: ${freeAreas.length}
    | 当前策略: <b class="color-accent">${allocStrategy === 'best-fit' ? 'Best-Fit' : 'First-Fit'}</b>
  </div>`;
  
  el.innerHTML = html;
}
```

---

### 8. 空间分配算法（First-Fit/Best-Fit）

```javascript
let allocStrategy = 'first-fit'; // first-fit | best-fit

/**
 * 首次适应（First-Fit）：从头扫描空闲链，找到第一个足够大的空闲区
 */
function allocateFirstFit(size) {
  for (let i = 0; i < freeAreas.length; i++) {
    const a = freeAreas[i];
    if (a.count >= size) {
      log(`  🔍 First-Fit扫描: [${a.start}:${a.count}] ≥ 需要${size}块 → 选中`, 'sys');
      
      const alloc = a.start;
      a.start += size;
      a.count -= size;
      
      if (a.count === 0) {
        freeAreas.splice(i, 1); // 删除空节点
      }
      return alloc;
    }
    log(`  🔍 First-Fit扫描: [${a.start}:${a.count}] < 需要${size}块 → 跳过`, 'sys');
  }
  return -1;
}

/**
 * 最佳适应（Best-Fit）：找到满足条件且最小的空闲区
 */
function allocateBestFit(size) {
  let bestIdx = -1, best = Infinity;
  
  for (let i = 0; i < freeAreas.length; i++) {
    if (freeAreas[i].count >= size && freeAreas[i].count < best) {
      best = freeAreas[i].count;
      bestIdx = i;
    }
  }
  
  if (bestIdx === -1) return -1;
  
  const a = freeAreas[bestIdx];
  const alloc = a.start;
  
  log(`  🔍 Best-Fit扫描: [${a.start}:${a.count}] 最佳匹配 (剩余${a.count-size}块)`, 'sys');
  
  a.start += size;
  a.count -= size;
  
  if (a.count === 0) {
    freeAreas.splice(bestIdx, 1);
  }
  return alloc;
}

/**
 * 分配策略入口：根据当前策略选择算法
 */
function bitmapFindFree(size) {
  log(`  📊 分配策略: ${allocStrategy === 'best-fit' ? 'Best-Fit (最佳适应)' : 'First-Fit (首次适应)'}`, 'info');
  return allocStrategy === 'best-fit' ? allocateBestFit(size) : allocateFirstFit(size);
}
```

---

### 9. 磁盘状态可视化

```javascript
/**
 * 32×32 磁盘网格视图
 */
function renderDiskGrid() {
  const grid = $('diskGrid');
  let html = '';
  
  for (let i = 0; i < DISK_SIZE; i++) {
    const b = diskBlocks[i];
    let cls = 'cell-free', tip = `块${i} 空闲`;
    
    if (b) {
      if (b.type === 'dir') {
        cls = 'cell-dir';
        tip = `块${i} ${b.file}`;
      } else if (b.type === 'meta') {
        cls = 'cell-meta';
        tip = `块${i} ${b.file}`;
      } else {
        cls = 'cell-data';
        tip = `块${i} ${b.file}`;
      }
    }
    
    html += `<div class="disk-cell ${cls}" 
      title="${tip}"
      style="${b && b.type === 'data' ? `background:${COLORS[(b.file || '').charCodeAt(0) % COLORS.length]}` : ''}"
      onclick="diskCellClick(${i})"></div>`;
  }
  grid.innerHTML = html;
}

/**
 * FAT 链表视图（连续分配的 FAT 表达形式）
 */
function renderFAT() {
  const el = $('fatContent');
  if (files.length === 0) {
    el.innerHTML = '<span class="text-sm color-text2">暂无文件</span>';
    return;
  }
  
  let html = '';
  files.forEach(f => {
    html += `<div class="mb-8"><b class="color-accent">📄 ${f.name}</b>
      <div class="fat-chain">`;
    for (let i = 0; i < f.size; i++) {
      html += `<span class="fat-block">${f.start + i}</span>`;
      if (i < f.size - 1) html += `<span class="fat-arrow">→</span>`;
    }
    html += `<span class="fat-arrow">→</span><span class="fat-eof">EOF</span></div></div>`;
  });
  
  // FAT 表格
  html += `<table class="data-table mt-8">
    <thead><tr><th>块号</th><th>Next</th><th>所属文件</th></tr></thead><tbody>`;
  files.forEach(f => {
    for (let i = 0; i < f.size; i++) {
      const next = i < f.size - 1 ? (f.start + i + 1) : 'EOF';
      html += `<tr>
        <td>${f.start + i}</td>
        <td style="color:${next === 'EOF' ? 'var(--red)' : 'var(--cyan)'}">${next}</td>
        <td>${f.name}</td></tr>`;
    }
  });
  html += `</tbody></table>`;
  el.innerHTML = html;
}
```

---

## 三、内存缓冲区管理功能

### 10. 缓冲池模型

```javascript
const BUF_SIZE = 8;  // 8 页缓冲页池，每页 64B

/**
 * 缓冲页数组，每个槽位存储：
 * {
 *   blockIdx:    对应磁盘块号,
 *   owner:       所属文件名,
 *   content:     块内容,
 *   accessTime:  最近访问时间戳（单调递增计数器）,
 *   dirty:       是否被修改过（脏页标记）
 * }
 */
let bufPages = new Array(BUF_SIZE).fill(null);
let bufAccessCounter = 0;  // 全局访问时间戳
let bufWritebacks = 0;     // 脏页写回次数
let bufHitCount = 0;       // 命中次数
let bufMissCount = 0;      // 缺失次数
```

---

### 11. LRU 页面置换算法

```javascript
/**
 * 将一个磁盘块加载到缓冲区 —— LRU 置换核心逻辑
 * 
 * 三种情况：
 *   1. 命中（已在缓冲区）→ 更新访问时间
 *   2. 有空闲槽位 → 直接放入
 *   3. 缓冲区满 → LRU 置换最久未访问的页面
 *      - 若被置换页是脏页，先模拟写回磁盘
 */
function bufLoadBlock(blockIdx, owner, content) {
  // === 情况1：检查是否已在缓冲区（命中）===
  let idx = bufPages.findIndex(p => 
    p && p.blockIdx === blockIdx && p.owner === owner
  );
  
  if (idx !== -1) {
    bufAccessCounter++;
    bufPages[idx].accessTime = bufAccessCounter;
    bufHitCount++;
    return idx;
  }
  
  bufMissCount++;
  
  // === 情况2：查找空闲槽位 ===
  idx = bufPages.findIndex(p => p === null);
  
  if (idx === -1) {
    // === 情况3：LRU 置换 ===
    let lruTime = Infinity, lruIdx = 0;
    
    // 遍历所有缓冲页，找 accessTime 最小的（最久未访问）
    bufPages.forEach((p, i) => {
      if (p.accessTime < lruTime) {
        lruTime = p.accessTime;
        lruIdx = i;
      }
    });
    
    const old = bufPages[lruIdx];
    
    // 脏页需要先写回磁盘
    if (old.dirty) {
      bufWritebacks++;
      log(`↩ 脏页写回: 块${old.blockIdx}(${old.owner})`, 'warn');
    } else {
      log(`✗ LRU置换: 丢弃块${old.blockIdx}(${old.owner})`, 'info');
    }
    
    idx = lruIdx;
  }
  
  // 将新块放入缓冲区
  bufAccessCounter++;
  bufPages[idx] = {
    blockIdx, owner, content,
    accessTime: bufAccessCounter,
    dirty: false  // 新加载的页默认为干净页
  };
  
  renderBufferView();
  return idx;
}
```

---

### 12. 脏页机制与性能统计

```javascript
/**
 * 缓冲页视图渲染 —— 显示脏页标记和访问时间
 */
function renderBufferView() {
  const el = $('bufPagesView');
  let html = '', used = 0;
  
  bufPages.forEach((p, i) => {
    if (p) {
      used++;
      html += `<div class="buf-page loaded" id="bpv${i}">
        <span class="bp-block">块${p.blockIdx}</span>
        <span class="bp-owner">${p.owner}</span>
        ${p.dirty ? '<span class="bp-dirty">D</span>' : ''}
        <span class="bp-time">t=${p.accessTime}</span>
      </div>`;
    } else {
      html += `<div class="buf-page empty">
        <span class="bp-block">空</span>
      </div>`;
    }
  });
  
  el.innerHTML = html;
  
  // 更新统计信息
  $('bufWbView').textContent = bufWritebacks;
  
  const total = bufHitCount + bufMissCount;
  $('bufHitView').textContent = bufHitCount;
  $('bufMissView').textContent = bufMissCount;
  $('bufRateView').textContent = total 
    ? ((bufHitCount / total * 100).toFixed(1) + '%') 
    : '0%';
  
  updateStatusBar();
}

/**
 * 编辑文件时标记脏页
 */
function _diskEditFile() {
  const name = selectedFile;
  // 将该文件在缓冲中的所有页面标记为脏页
  bufPages.forEach(p => {
    if (p && p.owner === name) p.dirty = true;
  });
  // ...
}
```

---

### 13. 加载动画模拟磁盘 I/O

```javascript
/**
 * 将文件内容逐块加载到缓冲区，带延时动画
 * 模拟真实磁盘 I/O 过程
 */
async function bufLoadFileAnimated(file) {
  if (!file) return;
  
  const totalBlocks = file.size;
  log(`⏳ 加载 ${file.name} 到缓冲区 (${totalBlocks}块)...`, 'info');
  
  for (let i = 0; i < totalBlocks; i++) {
    const blockIdx = file.start + i;
    
    // 高亮磁盘网格中正在读取的块（CSS reading 动画）
    const cells = document.querySelectorAll('#diskGrid .disk-cell');
    if (cells[blockIdx]) {
      cells[blockIdx].classList.add('reading');
      setTimeout(() => cells[blockIdx].classList.remove('reading'), 400);
    }
    
    // 检查是否已在缓冲区（命中/缺失）
    const alreadyIn = bufPages.find(p => 
      p && p.blockIdx === blockIdx && p.owner === file.name
    );
    
    bufLoadBlock(blockIdx, file.name, file.content && file.content[i] || '');
    
    if (alreadyIn) {
      log(`  ✅ HIT 块${blockIdx}(${file.name}) — 已在缓冲页中`, 'ok');
    } else {
      log(`  ❌ MISS 块${blockIdx}(${file.name}) — 加载到缓冲页`, 'info');
    }
    
    renderBufferView();
    
    // 延时 200ms，方便观察逐块加载过程
    await new Promise(r => setTimeout(r, 200));
  }
  
  log(`✓ ${file.name} 全部加载到缓冲区`, 'ok');
}
```

---

### 14. 五种页面置换算法对比

```javascript
/**
 * 五种页面置换算法缺页率对比
 * 算法：LRU / FIFO / OPT / LFU / CLOCK
 */
function pageCompareAlgorithms() {
  if (pageSeq.length === 0) pageGenerate();
  
  const algos = ['lru', 'fifo', 'opt', 'lfu', 'clock'];
  const algoNames = { lru: 'LRU', fifo: 'FIFO', opt: 'OPT', lfu: 'LFU', clock: 'CLOCK' };
  const colors = { lru: '#6366f1', fifo: '#3b78ff', opt: '#16c60c', lfu: '#fce100', clock: '#e83e8c' };
  
  const k = parseInt($('pageFrames').value) || 4;
  const results = {};
  
  algos.forEach(algo => {
    let mem = new Array(k).fill(null);
    let hits = 0, faults = 0, replaces = 0;
    let fifoQ = [], clockHand = 0;
    
    pageSeq.forEach((pid, idx) => {
      const fi = mem.findIndex(f => f && f.pageId === pid);
      
      if (fi !== -1) {
        hits++;
        mem[fi].lastAccess = idx;
        mem[fi].accessCount = (mem[fi].accessCount || 0) + 1;
        if (algo === 'clock') mem[fi].refBit = 1;
      } else {
        faults++;
        const ei = mem.findIndex(f => f === null);
        
        if (ei !== -1) {
          mem[ei] = { pageId: pid, lastAccess: idx, accessCount: 1, refBit: 1 };
          if (algo === 'fifo') fifoQ.push(ei);
        } else {
          let vi = -1;
          
          if (algo === 'lru') {
            // LRU：淘汰 lastAccess 最小的帧
            let t = Infinity;
            mem.forEach((f, i) => { if (f.lastAccess < t) { t = f.lastAccess; vi = i; } });
          } else if (algo === 'fifo') {
            // FIFO：淘汰队列头部帧
            vi = fifoQ.shift();
          } else if (algo === 'opt') {
            // OPT：淘汰未来最久不访问的帧
            let far = -1;
            mem.forEach((f, i) => {
              let nu = Infinity;
              for (let j = idx + 1; j < pageSeq.length; j++) {
                if (pageSeq[j] === f.pageId) { nu = j; break; }
              }
              if (nu > far) { far = nu; vi = i; }
            });
          } else if (algo === 'lfu') {
            // LFU：淘汰 accessCount 最小的帧
            let mc = Infinity;
            mem.forEach((f, i) => { if (f.accessCount < mc) { mc = f.accessCount; vi = i; } });
          } else if (algo === 'clock') {
            // CLOCK：扫描 refBit=0 的帧
            let sw = 0;
            while (sw < mem.length * 2) {
              const f = mem[clockHand];
              if (f && f.refBit === 0) { vi = clockHand; clockHand = (clockHand + 1) % mem.length; break; }
              if (f) f.refBit = 0;
              clockHand = (clockHand + 1) % mem.length;
              sw++;
            }
            if (vi === -1) vi = 0;
          }
          
          replaces++;
          mem[vi] = { pageId: pid, lastAccess: idx, accessCount: 1, refBit: 1 };
          if (algo === 'fifo') fifoQ.push(vi);
        }
      }
    });
    
    const total = hits + faults;
    results[algo] = {
      hits, faults, replaces,
      rate: total ? (hits / total * 100).toFixed(1) : 0,
      faultRate: total ? (faults / total * 100).toFixed(1) : 0
    };
  });
  
  // 渲染柱状图
  const box = $('compareBox');
  box.style.display = 'block';
  let html = '<div style="margin:8px 0">';
  
  algos.forEach(algo => {
    const r = results[algo];
    html += `<div style="display:flex;align-items:center;gap:8px;margin:4px 0">
      <span style="width:40px;text-align:right;font-size:11px;font-weight:600">${algoNames[algo]}</span>
      <div style="flex:1;height:22px;background:var(--bg);border-radius:4px;overflow:hidden">
        <div style="height:100%;width:${Math.max(r.faultRate, 5)}%;background:${colors[algo]};
          border-radius:4px;display:flex;align-items:center;justify-content:flex-end;
          padding-right:6px;font-size:10px;font-weight:700;color:#fff">${r.faultRate}%</div>
      </div>
      <span style="font-size:11px;color:${colors[algo]}">缺页${r.faults}</span>
    </div>`;
  });
  
  html += '</div>';
  
  // 命中率排行
  const sorted = algos.slice().sort((a, b) => 
    parseFloat(results[b].rate) - parseFloat(results[a].rate)
  );
  
  html += `<div class="text-sm mt-8">命中率排行: ${
    sorted.map((a, i) => 
      (i === 0 ? '🥇' : i === 1 ? '🥈' : i === 2 ? '🥉' : '') + 
      algoNames[a] + '(' + results[a].rate + '%)'
    ).join(' > ')
  }</div>`;
  
  $('compareResult').innerHTML = html;
}
```

---

## 四、进程同步与通信功能

### 15. 互斥同步（mutex + 等待队列）

```javascript
// 二元信号量：mutex + FIFO 等待队列
let mutex = { locked: false, owner: null };
let waitQueue = [];

/**
 * 请求锁 —— wait(mutex) 操作
 */
function lock(p) {
  if (mutex.locked) return false;
  mutex.locked = true;
  mutex.owner = p;
  updateMutexUI();
  return true;
}

/**
 * 释放锁 —— signal(mutex) 操作
 * 释放后自动调度等待队列中的下一个进程
 */
function unlock(p) {
  if (mutex.owner !== p) return;
  mutex.locked = false;
  mutex.owner = null;
  
  if (waitQueue.length > 0) {
    var n = waitQueue.shift();
    mutex.locked = true;
    mutex.owner = n.pid;
    log('unlock -> schedule ' + n.pid, 'sys');
    updateMutexUI();
    n.callback();
    unlock(n.pid);
  } else {
    updateMutexUI();
  }
}

/**
 * 请求锁封装 —— 带回调的完整流程
 */
function requestLock(p, cb) {
  if (lock(p)) {
    cb();
    unlock(p);
  } else {
    waitQueue.push({ pid: p, callback: cb });
    log('  wait ' + p + ' queued (len:' + waitQueue.length + ')', 'warn');
    updateMutexUI();
  }
}

/**
 * 更新互斥锁状态显示
 */
function updateMutexUI() {
  var el = $('statSel');
  if (mutex.locked) {
    el.innerHTML = 'lock:' + mutex.owner;
  } else {
    el.innerHTML = 'lock:free';
  }
  if (waitQueue.length > 0) {
    el.innerHTML += ' wait:' + waitQueue.length;
  }
}

/**
 * 文件操作封装 —— 操作前自动请求锁
 */
function diskCreateFile() {
  requestLock('createFile', function() { _diskCreateFile(); });
}

function diskDeleteFile() {
  requestLock('deleteFile', function() { _diskDeleteFile(); });
}

function diskEditFile() {
  requestLock('editFile', function() { _diskEditFile(); });
}
```

---

### 16. 文件访问冲突演示

```javascript
/**
 * 文件访问冲突演示 —— 直观展示编辑进程与删除进程的互斥逻辑
 */
async function showConflictDemo() {
  // 创建测试文件
  let df = files.find(f => f.name === 'conflict_test.txt');
  if (!df) {
    const wasCmd = cmdModeOn;
    cmdModeOn = false;
    $('nfName').value = 'conflict_test.txt';
    $('nfOwner').value = 'user1';
    $('nfSize').value = 3;
    $('nfContent').value = 'Conflict test file for mutual exclusion demo.';
    diskCreateFile();
    cmdModeOn = wasCmd;
    df = files.find(f => f.name === 'conflict_test.txt');
  }
  if (!df) return;
  
  const el = $('conflictContent');
  el.innerHTML = '';
  openModal('conflictModal');
  
  // 二进制信号量：1 = 未锁定，0 = 已锁定
  let fileLock = 1;
  
  const steps = [
    { c: 'info', m: '🔐 场景: P1(编辑进程) 和 P2(删除进程) 通过信号量互斥访问' },
    { c: 'info', m: '══════════════ 信号量初始化: mutex=1 ══════════════' },
    { c: 'info', m: '▶ P1: wait(mutex) → mutex=1, 获得锁' },
    { c: 'info', m: '✅ P1 获得文件锁，进入编辑模式' },
    { c: 'info', m: '▶ P2: wait(mutex) → mutex=0, 被阻塞！' },
    { c: 'blocked', m: '⛔ P2 阻塞在 wait(mutex) 操作上' },
    { c: 'info', m: '🔄 P1: signal(mutex) → 保存修改并释放锁' },
    { c: 'info', m: '▶ P2: wait(mutex) → mutex=1, 获得锁' },
    { c: 'info', m: '✅ P2 获得文件锁，执行删除操作' },
    { c: 'success', m: '✅ P2 成功删除文件，释放磁盘块' }
  ];
  
  for (let i = 0; i < steps.length; i++) {
    await new Promise(r => setTimeout(r, 600));
    
    // P1 获得锁，标记文件使用中
    if (i === 2) {
      selectedFile = 'conflict_test.txt';
      fileInUse = 'conflict_test.txt';
      fileLock = 0;
      renderAll();
      log('🔒 P1 wait(mutex): 获得信号量锁', 'sys');
    }
    
    // P2 被阻塞
    if (i === 4) {
      log('⛔ P2 wait(mutex): mutex=0, 进程阻塞', 'err');
      addLog('diskLog', '⛔ P2 删除被阻止: 文件正在被 P1 使用');
    }
    
    // P1 释放锁
    if (i === 6) {
      fileInUse = null;
      fileLock = 1;
      renderAll();
      log('🔓 P1 signal(mutex): 释放信号量锁', 'sys');
    }
    
    // P2 执行删除
    if (i === 8) {
      const idx = files.findIndex(f => f.name === 'conflict_test.txt');
      if (idx !== -1) {
        const f = files[idx];
        for (let j = f.start; j < f.start + f.size; j++) {
          diskBlocks[j] = null;
        }
        files.splice(idx, 1);
        selectedFile = null;
        updateFreeAreas();
        renderAll();
        log('✅ P2: 删除文件成功', 'ok');
      }
    }
    
    el.innerHTML += `<div class="conflict-step ${steps[i].c}">${steps[i].m}</div>`;
    el.scrollTop = el.scrollHeight;
  }
  
  await new Promise(r => setTimeout(r, 500));
  el.innerHTML += `<div class="conflict-step success" style="font-weight:600">🎬 演示完成！信号量机制有效保证互斥访问</div>`;
}
```

---

### 17. 生产者-消费者模型

```javascript
/**
 * 经典生产者-消费者问题的信号量模拟
 * 
 * 三个信号量：
 *   mutex = 1    —— 互斥信号量，保证对缓冲区的互斥访问
 *   empty = N    —— 同步信号量，表示空闲槽位数
 *   full  = 0    —— 同步信号量，表示已占用的槽位数
 */
let syncData = { buffer: [], mutex: 1, empty: 0, full: 0, bufSize: 5, step: 0 };
let syncProcs = [];

function syncInit() {
  const bs = parseInt($('syncBufSize').value) || 5;
  const np = parseInt($('syncProducers').value) || 2;
  const nc = parseInt($('syncConsumers').value) || 2;
  
  syncData = { buffer: [], mutex: 1, empty: bs, full: 0, bufSize: bs, step: 0 };
  syncProcs = [];
  
  for (let i = 0; i < np; i++) {
    syncProcs.push({ name: 'P' + (i + 1), type: 'producer', state: 'ready' });
  }
  for (let i = 0; i < nc; i++) {
    syncProcs.push({ name: 'C' + (i + 1), type: 'consumer', state: 'ready' });
  }
  syncRender();
}

function syncStep() {
  if (syncProcs.length === 0) syncInit();
  syncData.step++;
  
  const ready = syncProcs.filter(p => p.state === 'ready');
  if (ready.length === 0) {
    // 所有进程阻塞时，尝试唤醒
    syncProcs.forEach(p => { if (p.state === 'blocked') p.state = 'ready'; });
    syncRender();
    return;
  }
  
  const proc = ready[Math.floor(Math.random() * ready.length)];
  proc.state = 'running';
  
  if (proc.type === 'producer') {
    // === 生产者 ===
    if (syncData.empty <= 0 || syncData.mutex <= 0) {
      proc.state = 'blocked';
      $('syncAction').innerHTML = `<span class="color-red">${proc.name} 阻塞: 缓冲区已满</span>`;
    } else {
      syncData.empty--;
      syncData.mutex = 0;
      
      const item = 'D' + syncData.step;
      syncData.buffer.push(item);
      
      syncData.mutex = 1;
      syncData.full++;
      proc.state = 'ready';
      $('syncAction').innerHTML = `<span class="color-green">${proc.name} 生产 ${item}</span>`;
    }
  } else {
    // === 消费者 ===
    if (syncData.full <= 0 || syncData.mutex <= 0) {
      proc.state = 'blocked';
      $('syncAction').innerHTML = `<span class="color-red">${proc.name} 阻塞: 缓冲区为空</span>`;
    } else {
      syncData.full--;
      syncData.mutex = 0;
      
      const item = syncData.buffer.shift();
      
      syncData.mutex = 1;
      syncData.empty++;
      proc.state = 'ready';
      $('syncAction').innerHTML = `<span class="color-green">${proc.name} 消费 ${item}</span>`;
    }
  }
  
  syncRender();
}
```

---

### 18. 消息队列通信

```javascript
let msgQueue = [];     // 消息队列（FIFO）
let msgIdCounter = 0;  // 消息 ID 计数器

/**
 * 发送消息 —— 入队操作
 */
function msgSend() {
  const from = $('msgFrom').value.trim() || 'P1';
  const to = $('msgTo').value.trim() || 'P2';
  const content = $('msgContent').value.trim() || 'Hello';
  
  msgIdCounter++;
  msgQueue.push({ from, to, content, id: msgIdCounter });
  
  renderMsgQueue();
  log(`💬 ${from} → ${to}: "${content}"`, 'info');
}

/**
 * 接收消息 —— 出队操作（FIFO）
 */
function msgReceive() {
  if (msgQueue.length === 0) {
    log('消息队列为空', 'warn');
    return;
  }
  
  const msg = msgQueue.shift();
  renderMsgQueue();
  log(`✓ ${msg.to} 接收: "${msg.content}"`, 'ok');
}

/**
 * 渲染消息队列视图
 */
function renderMsgQueue() {
  let h = '';
  msgQueue.forEach(m => {
    h += `<div style="padding:3px 8px;background:rgba(99,102,241,.12);
      border:1px solid rgba(99,102,241,.25);border-radius:4px;font-size:10px">
      #${m.id} ${m.from}→${m.to}: <b>${m.content}</b></div>`;
  });
  $('msgQueueView').innerHTML = h || '<span class="text-xs color-text2">队列为空</span>';
}
```

---

### 19. 共享内存通信

```javascript
// 共享内存数据结构
let shmData = { value: 0, writer: '' };

/**
 * 写入共享内存
 */
function shmWrite() {
  const p = $('shmProc').value.trim() || 'P1';
  const v = parseInt($('shmData').value) || 0;
  
  shmData = { value: v, writer: p };
  $('shmValue').textContent = v;
  
  // 动画反馈
  $('shmValue').style.transform = 'scale(1.3)';
  setTimeout(() => $('shmValue').style.transform = 'scale(1)', 200);
  
  log(`🧠 ${p} 写入共享内存: ${v}`, 'info');
}

/**
 * 读取共享内存
 */
function shmRead() {
  const p = $('msgTo').value.trim() || 'P2';
  log(`🧠 ${p} 读取共享内存: ${shmData.value}` +
    (shmData.writer ? ` (由 ${shmData.writer} 写入)` : ''), 'info');
}
```

---

### 20. 管道通信（环形缓冲区）

```javascript
let pipeBuf = [];      // 管道缓冲区
let pipeCap = 6;       // 管道容量

/**
 * 渲染管道缓冲区
 */
function renderPipe() {
  const cap = parseInt($('pipeSize').value) || 6;
  pipeCap = cap;
  
  let h = '';
  for (let i = 0; i < cap; i++) {
    h += i < pipeBuf.length 
      ? `<div style="width:36px;height:28px;border-radius:4px;
          background:rgba(99,102,241,.15);border:1px solid rgba(99,102,241,.3);
          display:flex;align-items:center;justify-content:center;
          color:var(--accent);font-weight:700;font-size:12px">${pipeBuf[i]}</div>`
      : `<div style="width:36px;height:28px;border-radius:4px;
          background:var(--bg);border:1px dashed var(--border);
          display:flex;align-items:center;justify-content:center;
          color:var(--text2);font-size:9px">空</div>`;
  }
  
  $('pipeBuffer').innerHTML = h;
  $('pipeStatus').textContent = pipeBuf.length + '/' + cap;
  
  // 更新读写端状态
  $('pipeWriteEnd').textContent = pipeBuf.length >= cap ? '⛔ 阻塞' : '✓ 就绪';
  $('pipeWriteEnd').style.color = pipeBuf.length >= cap ? 'var(--red)' : 'var(--green)';
  
  $('pipeReadEnd').textContent = pipeBuf.length === 0 ? '⛔ 阻塞' : '✓ 就绪';
  $('pipeReadEnd').style.color = pipeBuf.length === 0 ? 'var(--red)' : 'var(--green)';
}

/**
 * 写入管道 —— 满时阻塞
 */
function pipeWrite() {
  const cap = parseInt($('pipeSize').value) || 6;
  pipeCap = cap;
  
  const w = $('pipeWriter').value.trim() || 'P1';
  const d = $('pipeData').value.trim() || 'A';
  
  if (pipeBuf.length >= cap) {
    log(`🔧 ${w} 写入阻塞: 管道已满`, 'warn');
    return;
  }
  
  pipeBuf.push(d);
  renderPipe();
  log(`🔧 ${w} 写入: "${d}"`, 'info');
}

/**
 * 读取管道 —— 空时阻塞（FIFO）
 */
function pipeRead() {
  if (pipeBuf.length === 0) {
    log('🔧 读取阻塞: 管道为空', 'warn');
    return;
  }
  
  const d = pipeBuf.shift();
  renderPipe();
  log(`🔧 取出: "${d}"`, 'info');
}
```

---

## 五、调度与死锁算法功能

### 21. 四种作业调度算法

```javascript
/**
 * 进程调度核心 —— 支持 FCFS / SJF / RR / 优先级
 */
function schedRun() {
  if (schedProcs.length === 0) return;
  
  schedProcs.forEach(p => {
    p.state = 'ready';
    p.remaining = p.burst;
    p.waitTime = 0;
    p.turnTime = 0;
  });
  
  const algo = $('schedAlgo').value;
  const q = parseInt($('rrQuantum').value) || 2;
  let gantt = [], t = 0;
  
  const procs = [...schedProcs].sort((a, b) => a.arrival - b.arrival);
  const colors = {};
  procs.forEach((p, i) => colors[p.name] = COLORS[i % COLORS.length]);
  
  if (algo === 'fcfs') {
    // === FCFS：按到达顺序依次执行 ===
    procs.forEach(p => {
      if (t < p.arrival) {
        gantt.push({ name: '空闲', color: '#333', time: p.arrival - t });
        t = p.arrival;
      }
      p.state = 'running';
      gantt.push({ name: p.name, color: colors[p.name], time: p.burst });
      p.waitTime = t - p.arrival;
      p.turnTime = t + p.burst - p.arrival;
      t += p.burst;
      p.state = 'done';
    });
    
  } else if (algo === 'sjf') {
    // === SJF：每次选执行时间最短的已到达进程 ===
    let rem = [...procs];
    while (rem.length) {
      const av = rem.filter(p => p.arrival <= t);
      if (av.length === 0) {
        gantt.push({ name: '空闲', color: '#333', time: 1 });
        t++;
        continue;
      }
      av.sort((a, b) => a.burst - b.burst);
      const p = av[0];
      rem.splice(rem.indexOf(p), 1);
      
      if (t < p.arrival) {
        gantt.push({ name: '空闲', color: '#333', time: p.arrival - t });
        t = p.arrival;
      }
      p.state = 'running';
      gantt.push({ name: p.name, color: colors[p.name], time: p.burst });
      p.waitTime = t - p.arrival;
      p.turnTime = t + p.burst - p.arrival;
      t += p.burst;
      p.state = 'done';
    }
    
  } else if (algo === 'rr') {
    // === RR：时间片轮转 ===
    let queue = [], idx = 0;
    const sorted = [...procs].sort((a, b) => a.arrival - b.arrival);
    
    while (idx < sorted.length || queue.length) {
      while (idx < sorted.length && sorted[idx].arrival <= t) {
        queue.push(sorted[idx]);
        idx++;
      }
      
      if (queue.length === 0) {
        gantt.push({ name: '空闲', color: '#333', time: 1 });
        t++;
        continue;
      }
      
      const p = queue.shift();
      const exec = Math.min(q, p.remaining);
      
      p.state = 'running';
      gantt.push({ name: p.name, color: colors[p.name], time: exec });
      t += exec;
      p.remaining -= exec;
      
      while (idx < sorted.length && sorted[idx].arrival <= t) {
        queue.push(sorted[idx]);
        idx++;
      }
      
      if (p.remaining > 0) {
        queue.push(p);
        p.state = 'ready';
      } else {
        p.state = 'done';
        p.turnTime = t - p.arrival;
        p.waitTime = p.turnTime - p.burst;
      }
    }
    
  } else if (algo === 'priority') {
    // === 优先级调度：数值越小优先级越高 ===
    let rem = [...procs];
    while (rem.length) {
      const av = rem.filter(p => p.arrival <= t);
      if (av.length === 0) {
        gantt.push({ name: '空闲', color: '#333', time: 1 });
        t++;
        continue;
      }
      av.sort((a, b) => a.priority - b.priority);
      const p = av[0];
      rem.splice(rem.indexOf(p), 1);
      
      if (t < p.arrival) {
        gantt.push({ name: '空闲', color: '#333', time: p.arrival - t });
        t = p.arrival;
      }
      p.state = 'running';
      gantt.push({ name: p.name, color: colors[p.name], time: p.burst });
      p.waitTime = t - p.arrival;
      p.turnTime = t + p.burst - p.arrival;
      t += p.burst;
      p.state = 'done';
    }
  }
  
  // ... 甘特图渲染和统计计算 ...
}
```

---

### 22. 甘特图可视化与统计

```javascript
/**
 * 渲染甘特图并计算统计指标
 */
function schedRun() {
  // ... 调度算法逻辑 ...
  
  // === 渲染甘特图 ===
  let gHtml = '<div class="gantt">';
  gantt.forEach(g => {
    const w = (g.time / t * 100).toFixed(1);
    gHtml += `<div class="gantt-block" style="width:${w}%;background:${g.color}"
      title="${g.name} ${g.time}单位">${g.time > 1 ? g.name : ''}</div>`;
  });
  gHtml += '</div>';
  $('ganttContainer').innerHTML = gHtml;
  
  // === 计算统计指标 ===
  const n = schedProcs.length;
  const avgWait = (schedProcs.reduce((s, p) => s + p.waitTime, 0) / n).toFixed(2);
  const avgTurn = (schedProcs.reduce((s, p) => s + p.turnTime, 0) / n).toFixed(2);
  const avgWTurn = (schedProcs.reduce((s, p) => s + p.turnTime / p.burst, 0) / n).toFixed(2);
  
  $('schedAvgWait').textContent = avgWait;
  $('schedAvgTurn').textContent = avgTurn;
  $('schedAvgWTurn').textContent = avgWTurn;
  
  renderSchedTable();
}
```

---

### 23. 银行家算法（安全序列检测）

```javascript
let bankData = null;

/**
 * 银行家算法数据初始化
 * 创建随机的进程数、资源种类、最大需求矩阵、分配矩阵、可用向量
 */
function bankInit() {
  const np = parseInt($('bankProc').value) || 5;
  const nr = parseInt($('bankRes').value) || 3;
  
  // 资源名称：A, B, C, D, E...
  const rnames = [];
  for (let i = 0; i < nr; i++) rnames.push(String.fromCharCode(65 + i));
  
  // 生成随机最大需求矩阵和分配矩阵
  const max = [], alloc = [];
  for (let i = 0; i < np; i++) {
    const mr = [], ar = [];
    for (let j = 0; j < nr; j++) {
      const m = 1 + Math.floor(Math.random() * 7);
      mr.push(m);
      ar.push(Math.floor(Math.random() * m)); // 分配量 < 最大需求
    }
    max.push(mr);
    alloc.push(ar);
  }
  
  // 可用资源向量
  let avail;
  const as = $('bankAvail').value.trim();
  if (as) {
    avail = as.split(/\s+/).map(Number);
  } else {
    avail = [];
    for (let j = 0; j < nr; j++) {
      let t = 0;
      alloc.forEach(r => t += r[j]);
      avail.push(t + Math.floor(Math.random() * 5) + 1);
    }
  }
  
  bankData = {
    np, nr, rnames, max, alloc,
    need: max.map((r, i) => r.map((v, j) => v - alloc[i][j])),
    avail
  };
  
  renderBankTable();
}

/**
 * 银行家算法核心 —— 安全序列检测
 * 
 * 算法步骤：
 *   1. 初始化 finish[] = false, safeSeq[] = []
 *   2. 查找 Need[i] ≤ Available 的未完成进程
 *   3. 执行该进程，释放资源：Available += Allocation[i]
 *   4. 标记 finish[i] = true，加入安全序列
 *   5. 重复步骤 2~4，直到找不到新进程
 *   6. 若所有进程都完成，则系统安全；否则存在死锁风险
 */
function bankFindSafe() {
  if (!bankData) {
    log('请先初始化数据', 'warn');
    return;
  }
  
  const { np, nr, need, alloc } = bankData;
  let avail = [...bankData.avail];
  const finish = new Array(np).fill(false);
  const safeSeq = [];
  const steps = [];
  
  let changed = true;
  while (changed) {
    changed = false;
    for (let i = 0; i < np; i++) {
      if (finish[i]) continue;
      
      // 检查 Need[i] ≤ Available
      let ok = true;
      for (let j = 0; j < nr; j++) {
        if (need[i][j] > avail[j]) { ok = false; break; }
      }
      
      if (ok) {
        // 执行完成，释放资源
        for (let j = 0; j < nr; j++) avail[j] += alloc[i][j];
        finish[i] = true;
        safeSeq.push(i);
        changed = true;
        steps.push(`P${i} 可执行，释放资源 → Available=[${avail.join(',')}]`);
      }
    }
  }
  
  const allDone = finish.every(f => f);
  if (allDone) {
    $('safeSeq').innerHTML = `<span class="color-green">✓ 安全！序列: ${safeSeq.map(i => 'P' + i).join(' → ')}</span>`;
    log(`安全序列: ${safeSeq.map(i => 'P' + i).join(' → ')}`, 'ok');
  } else {
    $('safeSeq').innerHTML = `<span class="color-red">✗ 不安全！存在死锁风险</span>`;
    log('系统不安全', 'err');
  }
  
  steps.forEach(s => log(s, 'info'));
  drawBankGraph(safeSeq);
}
```

---

### 24. 资源分配图可视化

```javascript
/**
 * 使用 Canvas 绘制资源分配图
 *   - 圆形节点表示进程（P0, P1, ...）
 *   - 方形节点表示资源（A, B, C, ...）
 *   - 实线箭头：资源 → 进程（已分配）
 *   - 虚线箭头：进程 → 资源（申请中）
 */
function drawBankGraph(safeSeq) {
  const c = $('graphCanvas');
  if (!c) return;
  
  const ctx = c.getContext('2d');
  const W = c.width, H = c.height;
  ctx.clearRect(0, 0, W, H);
  
  if (!bankData) return;
  
  const { np, nr, alloc, need, rnames } = bankData;
  
  // 计算节点位置
  const pNodes = [], rNodes = [];
  for (let i = 0; i < np; i++) {
    pNodes.push({
      x: 70,
      y: 30 + i * (H - 60) / (Math.max(np - 1, 1)),
      name: 'P' + i,
      color: safeSeq && safeSeq.includes(i) ? '#16c60c' : '#6366f1'
    });
  }
  for (let i = 0; i < nr; i++) {
    rNodes.push({
      x: W - 70,
      y: 50 + i * (H - 100) / (Math.max(nr - 1, 1)),
      name: rnames[i],
      color: '#fce100'
    });
  }
  
  ctx.lineWidth = 2;
  
  // 绘制分配边（资源 → 进程）
  for (let i = 0; i < np; i++) {
    for (let j = 0; j < nr; j++) {
      if (alloc[i][j] > 0) {
        ctx.strokeStyle = 'rgba(99,102,241,.5)';
        ctx.beginPath();
        ctx.moveTo(rNodes[j].x - 16, rNodes[j].y);
        ctx.lineTo(pNodes[i].x + 16, pNodes[i].y);
        ctx.stroke();
      }
      
      // 绘制需求边（进程 → 资源）
      if (need[i][j] > 0) {
        ctx.strokeStyle = 'rgba(240,58,23,.4)';
        ctx.setLineDash([4, 4]);
        ctx.beginPath();
        ctx.moveTo(pNodes[i].x + 16, pNodes[i].y);
        ctx.lineTo(rNodes[j].x - 16, rNodes[j].y);
        ctx.stroke();
      }
    }
  }
  
  ctx.setLineDash([]);
  
  // 绘制进程节点（圆形）
  pNodes.forEach(n => {
    ctx.fillStyle = n.color;
    ctx.beginPath();
    ctx.arc(n.x, n.y, 16, 0, Math.PI * 2);
    ctx.fill();
    ctx.fillStyle = '#fff';
    ctx.font = 'bold 11px sans-serif';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(n.name, n.x, n.y);
  });
  
  // 绘制资源节点（方形）
  rNodes.forEach(n => {
    ctx.fillStyle = n.color;
    ctx.fillRect(n.x - 14, n.y - 14, 28, 28);
    ctx.fillStyle = '#fff';
    ctx.font = 'bold 11px sans-serif';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(n.name, n.x, n.y);
  });
}
```

---

### 25. 打印作业调度

```javascript
let printJobs = [], printJobId = 0;

/**
 * 添加打印作业
 */
function addPrintJob(name, size, path, priority) {
  if (name === undefined) {
    name = $('ajName').value.trim() || 'report.pdf';
    path = $('ajPath').value.trim() || 'C:\\';
    size = parseInt($('ajSize').value) || 4;
    priority = parseInt($('ajPriority').value) || 3;
    closeModal('addJobModal');
  }
  
  printJobs.push({
    id: ++printJobId,
    name, size, path, priority,
    submitted: new Date().toLocaleTimeString(),
    status: 'waiting',
    progress: 0
  });
  
  renderPrintJobs();
  log(`📄 打印作业: ${name} 已添加`, 'info');
}

/**
 * 执行打印作业调度（支持 FCFS / SJF / RR / 优先级）
 */
function runPrintJobs() {
  if (printJobs.length === 0) {
    log('暂无打印作业', 'warn');
    return;
  }
  
  const algo = $('printAlgo').value;
  const q = parseInt($('printRRBox').value) || 2;
  
  let jobs = [...printJobs.filter(j => j.status === 'waiting')];
  let gantt = [], t = 0;
  
  // ... 调度逻辑与进程调度类似，略 ...
  
  renderPrintGantt(gantt);
}
```

---

## 六、交互与可视化功能

### 26. Windows 风格图形界面

```javascript
/**
 * 全局工具函数 —— 简化 DOM 操作
 */
const $ = id => document.getElementById(id);

function ts() {
  const d = new Date();
  return `[${d.getHours().toString().padStart(2, '0')}:${d.getMinutes().toString().padStart(2, '0')}:${d.getSeconds().toString().padStart(2, '0')}]`;
}

function log(msg, type = 'info') {
  const el = $('consoleBody');
  const clsMap = { ok: 'log-ok', err: 'log-err', warn: 'log-warn', info: 'log-info', sys: 'log-sys' };
  el.innerHTML += `<div class="log-line ${clsMap[type] || 'log-info'}">
    <span class="log-ts">${ts()}</span>${msg}</div>`;
  el.scrollTop = el.scrollHeight;
}

function toast(msg, icon = '✅', duration = 2800) {
  const c = $('toastContainer');
  const t = document.createElement('div');
  t.className = 'toast';
  t.innerHTML = `<span class="toast-icon">${icon}</span>${msg}`;
  c.appendChild(t);
  setTimeout(() => { if (t.parentNode) t.parentNode.removeChild(t); }, duration + 400);
}

function openModal(id) { $(id).classList.add('show'); }
function closeModal(id) { $(id).classList.remove('show'); }
```

---

### 27. 命令行终端（9条命令）

```javascript
/**
 * 命令行终端核心 —— 支持 dir/cat/create/delete/modify/info/stat/help/clear
 */
function execCommand() {
  const input = $('cmdInput');
  if (!input) return;
  
  const line = input.value.trim();
  if (!line) return;
  input.value = '';
  
  const parts = line.split(/\s+/);
  const cmd = parts[0].toLowerCase();
  const args = parts.slice(1);
  
  log(`<span class="cmd-input">C:\\OS&gt; ${line}</span>`, 'sys');
  
  try {
    switch (cmd) {
      // 列出目录
      case 'dir': case 'ls': {
        const user = args[0] || null;
        let flist = user ? files.filter(f => f.owner === user) : files;
        if (flist.length === 0) { log('  目录为空', 'cmd-out'); break; }
        log(`  文件名${' '.repeat(18)}创建日期${' '.repeat(6)}块数  权限    用户`, 'cmd-table');
        flist.forEach(f => {
          log(`  ${(f.name + '                    ').slice(0, 20)} ${f.time}  ${(f.size + '块').padStart(3)}   ${f.perm}  ${f.owner}`, 'cmd-table');
        });
        break;
      }
      
      // 查看文件内容
      case 'cat': case 'view': {
        if (!args[0]) { log('  用法: cat <文件名>', 'cmd-err'); break; }
        const f = files.find(x => x.name === args[0]);
        if (!f) { log(`  错误: 文件 ${args[0]} 不存在`, 'cmd-err'); break; }
        log(`  📄 ${f.name} (${f.owner}) 块${f.start}~${f.start + f.size - 1}`, 'cmd-table');
        for (let i = 0; i < f.size; i++) {
          const blockIdx = f.start + i;
          const content = f.content && f.content[i] ? f.content[i].replace(/\n/g, '\\n') : '(空)';
          log(`  块${blockIdx} [0x${blockIdx.toString(16).toUpperCase().padStart(4, '0')}]: ${content}`, 'cmd-table');
        }
        break;
      }
      
      // 创建文件
      case 'create': {
        if (!args[0]) { log('  用法: create <文件名> [块数] [用户]', 'cmd-err'); break; }
        const name = args[0];
        const size = parseInt(args[1]) || 5;
        const owner = args[2] || 'user1';
        $('nfName').value = name;
        $('nfOwner').value = owner;
        $('nfSize').value = size;
        const wasCmdMode = cmdModeOn;
        cmdModeOn = false;
        diskCreateFile();
        cmdModeOn = wasCmdMode;
        break;
      }
      
      // 删除文件
      case 'delete': {
        if (!args[0]) { log('  用法: delete <文件名>', 'cmd-err'); break; }
        selectedFile = args[0];
        diskDeleteFile();
        break;
      }
      
      // 修改文件
      case 'modify': {
        if (!args[0]) { log('  用法: modify <文件名> [块号] [内容]', 'cmd-err'); break; }
        const name = args[0];
        const f = files.find(x => x.name === name);
        if (!f) { log(`  错误: 文件 ${name} 不存在`, 'cmd-err'); break; }
        if (args.length >= 3) {
          const blockIdx = parseInt(args[1]);
          const content = args.slice(2).join(' ');
          if (blockIdx >= 0 && blockIdx < f.size) {
            if (!f.content) f.content = [];
            f.content[blockIdx] = content;
            log(`  ✓ 块${f.start + blockIdx} 内容已更新`, 'ok');
          } else {
            log(`  块号范围 0~${f.size - 1}`, 'cmd-err');
          }
        } else {
          selectedFile = name;
          diskEditFile();
        }
        break;
      }
      
      // 查看 FCB 信息
      case 'info': {
        if (!args[0]) { log('  用法: info <文件名>', 'cmd-err'); break; }
        const f = files.find(x => x.name === args[0]);
        if (!f) { log(`  错误: 文件 ${args[0]} 不存在`, 'cmd-err'); break; }
        log(`  📝 文件名: ${f.name}`, 'cmd-table');
        log(`  👤 用户: ${f.owner}`, 'cmd-table');
        log(`  📅 创建时间: ${f.time}`, 'cmd-table');
        log(`  🔒 权限: ${f.perm}`, 'cmd-table');
        log(`  📍 起始块: ${f.start}`, 'cmd-table');
        log(`  📦 占用块数: ${f.size}`, 'cmd-table');
        break;
      }
      
      // 系统状态
      case 'stat': {
        const used = diskBlocks.filter(b => b !== null).length;
        const free = DISK_SIZE - used;
        log(`  💾 磁盘: ${DISK_SIZE}块, 已用${used}块, 空闲${free}块 (${(free/DISK_SIZE*100).toFixed(1)}%)`, 'cmd-table');
        log(`  📄 文件: ${files.length}个`, 'cmd-table');
        log(`  📑 缓冲: ${bufPages.filter(p => p !== null).length}/${BUF_SIZE}页`, 'cmd-table');
        log(`  🔗 空闲盘区: ${freeAreas.length}个`, 'cmd-table');
        break;
      }
      
      // 帮助
      case 'help': {
        log('  ═══════════════ 可用命令 ═══════════════', 'cmd-table');
        log('  dir [用户]      — 列出目录信息', 'cmd-table');
        log('  cat <文件名>     — 查看文件内容', 'cmd-table');
        log('  create <名> [块] [用户] — 新建文件', 'cmd-table');
        log('  delete <文件名>  — 删除文件', 'cmd-table');
        log('  modify <名> [块号] [内容] — 修改文件', 'cmd-table');
        log('  info <文件名>    — 查看FCB详细信息', 'cmd-table');
        log('  stat            — 磁盘/系统状态', 'cmd-table');
        log('  help            — 显示此帮助', 'cmd-table');
        log('  clear           — 清空控制台', 'cmd-table');
        break;
      }
      
      // 清空控制台
      case 'clear': {
        $('consoleBody').innerHTML = '';
        $('consoleCount').textContent = '0 条';
        break;
      }
      
      default:
        log(`  未知命令: ${cmd}，输入 help 查看可用命令`, 'cmd-err');
    }
  } catch (e) {
    log(`  执行错误: ${e.message}`, 'cmd-err');
  }
}
```

---

### 28. 右键菜单与快捷键

```javascript
/**
 * 菜单栏点击处理
 */
document.querySelectorAll('.menu-item').forEach(mi => {
  mi.addEventListener('click', function(e) {
    e.stopPropagation();
    const dd = this.querySelector('.menu-dropdown');
    if (!dd) return;
    
    const wasOpen = dd.classList.contains('show');
    document.querySelectorAll('.menu-dropdown.show').forEach(d => d.classList.remove('show'));
    document.querySelectorAll('.menu-item.open').forEach(m => m.classList.remove('open'));
    
    if (!wasOpen) {
      dd.classList.add('show');
      this.classList.add('open');
    }
  });
});

/**
 * 点击外部关闭菜单
 */
document.addEventListener('click', () => {
  document.querySelectorAll('.menu-dropdown.show').forEach(d => d.classList.remove('show'));
  document.querySelectorAll('.menu-item.open').forEach(m => m.classList.remove('open'));
});
```

---

### 29. 一键演示模式

```javascript
/**
 * 文件访问冲突演示 —— 一键演示互斥同步原理
 */
async function showConflictDemo() {
  // 创建测试文件
  let df = files.find(f => f.name === 'conflict_test.txt');
  if (!df) {
    const wasCmd = cmdModeOn;
    cmdModeOn = false;
    $('nfName').value = 'conflict_test.txt';
    $('nfOwner').value = 'user1';
    $('nfSize').value = 3;
    $('nfContent').value = 'Conflict test file for mutual exclusion demo.';
    diskCreateFile();
    cmdModeOn = wasCmd;
    df = files.find(f => f.name === 'conflict_test.txt');
  }
  if (!df) return;
  
  const el = $('conflictContent');
  el.innerHTML = '';
  openModal('conflictModal');
  
  const steps = [
    { c: 'info', m: '🔐 场景: P1(编辑进程) 和 P2(删除进程) 通过信号量互斥访问' },
    { c: 'info', m: '══════════════ 信号量初始化: mutex=1 ══════════════' },
    { c: 'info', m: '▶ P1: wait(mutex) → mutex=1, 获得锁' },
    { c: 'info', m: '✅ P1 获得文件锁，进入编辑模式' },
    { c: 'info', m: '▶ P2: wait(mutex) → mutex=0, 被阻塞！' },
    { c: 'blocked', m: '⛔ P2 阻塞在 wait(mutex) 操作上' },
    { c: 'info', m: '🔄 P1: signal(mutex) → 保存修改并释放锁' },
    { c: 'info', m: '▶ P2: wait(mutex) → mutex=1, 获得锁' },
    { c: 'info', m: '✅ P2 获得文件锁，执行删除操作' },
    { c: 'success', m: '✅ P2 成功删除文件，释放磁盘块' }
  ];
  
  for (let i = 0; i < steps.length; i++) {
    await new Promise(r => setTimeout(r, 600));
    
    // P1 获得锁
    if (i === 2) {
      selectedFile = 'conflict_test.txt';
      fileInUse = 'conflict_test.txt';
      renderAll();
    }
    
    // P2 被阻塞
    if (i === 4) {
      log('⛔ P2 wait(mutex): mutex=0, 进程阻塞', 'err');
    }
    
    // P1 释放锁
    if (i === 6) {
      fileInUse = null;
      renderAll();
    }
    
    // P2 执行删除
    if (i === 8) {
      const idx = files.findIndex(f => f.name === 'conflict_test.txt');
      if (idx !== -1) {
        const f = files[idx];
        for (let j = f.start; j < f.start + f.size; j++) {
          diskBlocks[j] = null;
        }
        files.splice(idx, 1);
        updateFreeAreas();
        renderAll();
      }
    }
    
    el.innerHTML += `<div class="conflict-step ${steps[i].c}">${steps[i].m}</div>`;
    el.scrollTop = el.scrollHeight;
  }
  
  await new Promise(r => setTimeout(r, 500));
  el.innerHTML += `<div class="conflict-step success" style="font-weight:600">🎬 演示完成！信号量机制有效保证互斥访问</div>`;
}
```

---

## 七、全局状态与初始化

```javascript
// === 全局状态 ===
const DISK_SIZE = 1024, BLOCK_SIZE = 64, BUF_SIZE = 8;
let diskBlocks = new Array(DISK_SIZE).fill(null);
let files = [], folders = [];
let bufPages = new Array(BUF_SIZE).fill(null);
let freeAreas = [];
let allocStrategy = 'first-fit';

// === 缓冲统计 ===
let bufAccessCounter = 0;
let bufWritebacks = 0, bufHitCount = 0, bufMissCount = 0;

// === 互斥同步 ===
let mutex = { locked: false, owner: null }, waitQueue = [];
let fileInUse = null;

// === 目录导航 ===
let currentPath = 'user1';
let navHistory = [], navHistoryIdx = -1;

// === 命令模式 ===
let cmdQueue = [], cmdIdCounter = 0, cmdModeOn = false;

/**
 * 磁盘初始化 —— 系统启动时调用
 */
function diskInit() {
  // 前 8 块 —— MFD（主文件目录）
  for (let i = 0; i < 8; i++) {
    diskBlocks[i] = { type: 'dir', file: 'MFD' };
  }
  
  // 第 8~15 块 —— UFD（用户文件目录）
  for (let i = 8; i < 16; i++) {
    diskBlocks[i] = { type: 'dir', file: 'UFD' };
  }
  
  // 第 16~23 块 —— FreeTable（空闲盘区表）
  for (let i = 16; i < 24; i++) {
    diskBlocks[i] = { type: 'meta', file: 'FreeTable' };
  }
  
  files = [];
  folders = ['user1', 'user2', 'system'];
  bufPages = new Array(BUF_SIZE).fill(null);
  
  // 创建系统应用程序
  createExeFile('printer.exe', 'system', 'PRINTER:打印作业调度程序');
  createExeFile('paint.exe', 'system', 'PAINT:画图程序');
  createExeFile('notepad.exe', 'system', 'NOTEPAD:文本编辑器');
  createExeFile('calc.exe', 'system', 'CALC:计算器');
  
  updateFreeAreas();
  renderAll();
}

/**
 * 页面加载完成后初始化
 */
window.addEventListener('load', () => {
  diskInit();
  log('操作系统模拟器已启动', 'ok');
});
```