# 操作系统课设 —— 核心代码详解

> 本文档摘录 `simulator.html` 中各功能模块的核心代码，并附有详细注释。

---

## 目录

1. [磁盘初始化与空闲盘区链](#1-磁盘初始化与空闲盘区链)
2. [文件创建（连续存储分配）](#2-文件创建连续存储分配)
3. [文件删除与文件保护](#3-文件删除与文件保护)
4. [内存缓冲页系统（LRU 置换）](#4-内存缓冲页系统lru-置换)
5. [缓冲页动画加载](#5-缓冲页动画加载)
6. [页面置换算法（5 种）](#6-页面置换算法5-种)
7. [进程调度算法（4 种）](#7-进程调度算法4-种)
8. [银行家算法（安全序列检测）](#8-银行家算法安全序列检测)
9. [资源分配图绘制](#9-资源分配图绘制)
10. [生产者-消费者同步模拟](#10-生产者-消费者同步模拟)
11. [消息传递机制](#11-消息传递机制)
12. [位示图(Bitmap)渲染](#12-位示图bitmap渲染)
13. [FAT 链表视图](#13-fat-链表视图)
14. [FCB/iNode 详情弹窗](#14-fcbinode-详情弹窗)
15. [五种算法缺页率对比](#15-五种算法缺页率对比)
16. [共享内存通信](#16-共享内存通信)
17. [管道通信](#17-管道通信)
18. [命令进程调度](#18-命令进程调度)
19. [导览演示模式](#19-导览演示模式)

---

## 1. 磁盘初始化与空闲盘区链

### 1.1 数据结构定义

```javascript
// 磁盘常量：1024 个盘块，每块 64 字节
const DISK_SIZE = 1024, BLOCK_SIZE = 64;

// 磁盘块数组：null 表示空闲，否则存储 {type, file, content}
// type: 'dir'（目录区）| 'meta'（元数据区）| 'data'（文件数据区）
let diskBlocks = new Array(DISK_SIZE).fill(null);

// 文件目录表：存储所有文件的 FCB（文件控制块）信息
// {name: 文件名, owner: 所属用户, time: 创建时间,
//  start: 起始物理块号, size: 占用块数, perm: 权限,
//  content: [每个块的内容字符串]}
let files = [];

// 空闲盘区链：记录连续的空闲区域
// [{start: 起始块号, count: 连续空闲块数}]
let freeAreas = [];
```

### 1.2 磁盘初始化

```javascript
function diskInit() {
  // 前 8 块保留给 MFD（主文件目录）
  for (let i = 0; i < 8; i++)
    diskBlocks[i] = { type: 'dir', file: 'MFD' };

  // 第 8~15 块保留给 UFD（用户文件目录）
  for (let i = 8; i < 16; i++)
    diskBlocks[i] = { type: 'dir', file: 'UFD' };

  // 第 16~23 块保留给 FreeTable（空闲盘区表元数据）
  for (let i = 16; i < 24; i++)
    diskBlocks[i] = { type: 'meta', file: 'FreeTable' };

  // 清空文件列表和缓冲页
  files = [];
  bufPages = new Array(BUF_SIZE).fill(null);
  bufAccessCounter = 0;
  bufWritebacks = 0;

  // 扫描并建立空闲盘区链
  updateFreeAreas();
  diskRender();
  bufRender();
}
```

### 1.3 空闲盘区链维护

```javascript
/**
 * 扫描整个磁盘块数组，将连续的空闲区域合并为链表节点。
 * 这是"空闲盘区链"管理方式的核心：
 *   遍历 0~1023 号块，遇到 null（空闲块）就开始计数，
 *   遇到非 null 块就截断，形成一个 {start, count} 节点。
 */
function updateFreeAreas() {
  freeAreas = [];
  let start = -1, cnt = 0;

  for (let i = 0; i < DISK_SIZE; i++) {
    if (diskBlocks[i] === null) {
      // 当前块空闲
      if (start === -1) {
        start = i;  // 记录空闲区起始位置
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
  if (start !== -1)
    freeAreas.push({ start, count: cnt });
}
```

---

## 2. 文件创建（连续存储分配）

```javascript
/**
 * 新建文件 —— 连续存储方式
 *
 * 流程：
 *   1. 将文件内容按 64B 拆分成多个内容块
 *   2. 在空闲盘区链中查找满足 size 的连续空闲区域（首次适应）
 *   3. 将文件数据写入连续的物理块
 *   4. 创建文件目录项（FCB），加入文件目录表
 */
function diskCreateFile() {
  const name = $('newFileName').value.trim() || ('file' + (files.length + 1));
  const owner = $('newFileOwner').value.trim() || 'user1';
  const rawContent = $('newFileContent').value;

  // 检查文件名是否重复
  if (files.find(f => f.name === name)) {
    addLog('diskLog', `错误: 文件 ${name} 已存在`);
    return;
  }

  // ---- 第一步：按 BLOCK_SIZE(64B) 拆分内容 ----
  let contentBlocks = [];
  if (rawContent) {
    for (let i = 0; i < rawContent.length; i += BLOCK_SIZE) {
      contentBlocks.push(rawContent.substring(i, i + BLOCK_SIZE));
    }
  }

  // 文件占用块数 = max(用户指定块数, 内容实际需要的块数)
  const size = Math.max(
    parseInt($('newFileSize').value) || 5,
    contentBlocks.length || 1
  );

  // ---- 第二步：在空闲盘区链中首次适应分配 ----
  let start = -1;
  for (const a of freeAreas) {
    if (a.count >= size) {  // 找到第一个足够大的空闲区
      start = a.start;
      break;
    }
  }
  if (start === -1) {
    addLog('diskLog', `错误: 无足够连续空闲空间 (需要${size}块)`);
    return;
  }

  // ---- 第三步：将文件数据写入物理块 ----
  for (let i = start; i < start + size; i++) {
    const blockIdx = i - start;  // 文件内的相对块号
    diskBlocks[i] = {
      type: 'data',
      file: name,
      content: blockIdx < contentBlocks.length ? contentBlocks[blockIdx] : ''
    };
  }

  // ---- 第四步：创建 FCB 并加入目录 ----
  const now = new Date();
  const time = `${now.getFullYear()}-${(now.getMonth() + 1)
    .toString().padStart(2, '0')}-${now.getDate().toString().padStart(2, '0')}`;

  files.push({
    name, owner, time,
    start,        // 起始物理块号
    size,         // 占用物理块数
    perm: 'rw-r--r--',  // 基本权限：所有者读写，其他只读
    content: contentBlocks
  });

  updateFreeAreas();  // 更新空闲盘区链
  diskRender();       // 重新渲染磁盘视图和目录树
}
```

---

## 3. 文件删除与文件保护

```javascript
/**
 * 删除文件 —— 含文件保护机制
 *
 * 文件保护：
 *   变量 fileInUse 记录当前正在查看/编辑的文件名。
 *   若待删文件正在使用中，阻止删除操作。
 *
 * 删除流程：
 *   1. 检查文件保护 → 2. 释放物理块 → 3. 清理缓冲区 → 4. 更新目录
 */
let fileInUse = null;  // 文件保护：当前正在操作的文件名

function diskDeleteFile() {
  const name = selectedFile || $('newFileName').value.trim();
  if (!name) {
    addLog('diskLog', '请先选择或输入文件名');
    return;
  }

  // ---- 文件保护检查 ----
  if (fileInUse === name) {
    addLog('diskLog',
      `⛔ 删除被阻止: 文件 ${name} 正在使用中，请先关闭查看/编辑后再删除`);
    return;
  }

  const idx = files.findIndex(f => f.name === name);
  if (idx === -1) {
    addLog('diskLog', `文件 ${name} 不存在`);
    return;
  }

  const f = files[idx];

  // ---- 释放物理块 ----
  for (let i = f.start; i < f.start + f.size; i++)
    diskBlocks[i] = null;

  // ---- 清理内存缓冲区中该文件的页面 ----
  bufPages.forEach((p, bi) => {
    if (p && p.owner === name) bufPages[bi] = null;
  });
  bufRender();

  // ---- 从目录表中移除 ----
  files.splice(idx, 1);
  selectedFile = null;

  updateFreeAreas();  // 重建空闲盘区链
  diskRender();
}
```

---

## 4. 内存缓冲页系统（LRU 置换）

### 4.1 数据结构

```javascript
const BUF_SIZE = 8;  // K=8 个缓冲页，每页 M=64B

/**
 * 缓冲页数组，每个槽位存储：
 *   {blockIdx: 对应磁盘块号,
 *    owner:    所属文件名,
 *    content:  块内容,
 *    accessTime: 最近访问时间戳（单调递增计数器）,
 *    dirty:    是否被修改过（脏页标记）}
 */
let bufPages = new Array(BUF_SIZE).fill(null);
let bufAccessCounter = 0;  // 全局访问时间戳（每访问一次 +1）
let bufWritebacks = 0;     // 脏页写回磁盘的累计次数
```

### 4.2 LRU 缓冲页加载核心逻辑

```javascript
/**
 * 将一个磁盘块加载到缓冲区
 *
 * 三种情况：
 *   1. 命中（已在缓冲区）→ 更新访问时间
 *   2. 有空闲槽位 → 直接放入
 *   3. 缓冲区满 → LRU 置换最久未访问的页面
 *      - 若被置换页是脏页，先模拟写回磁盘
 */
function bufLoadBlock(blockIdx, owner, content) {
  // ---- 情况1：检查是否已在缓冲区（命中） ----
  let idx = bufPages.findIndex(p => p && p.blockIdx === blockIdx && p.owner === owner);
  if (idx !== -1) {
    bufAccessCounter++;
    bufPages[idx].accessTime = bufAccessCounter;  // 刷新访问时间
    return idx;
  }

  // ---- 情况2：查找空闲槽位 ----
  idx = bufPages.findIndex(p => p === null);

  if (idx === -1) {
    // ---- 情况3：LRU 置换 ----
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
      addLog('diskLog',
        `↩ 脏页写回磁盘: 块${old.blockIdx}(${old.owner})`);
    } else {
      addLog('diskLog',
        `✗ LRU置换: 丢弃块${old.blockIdx}(${old.owner})`);
    }

    idx = lruIdx;  // 使用被置换页的槽位
  }

  // 将新块放入缓冲区
  bufAccessCounter++;
  bufPages[idx] = {
    blockIdx, owner, content,
    accessTime: bufAccessCounter,
    dirty: false  // 新加载的页默认为干净页
  };
  return idx;
}
```

---

## 5. 缓冲页动画加载

```javascript
/**
 * 将文件内容逐块加载到缓冲区，带延时动画
 *
 * 流程（每块间隔 300ms）：
 *   1. 高亮磁盘网格中对应的物理块（reading 动画）
 *   2. 调用 bufLoadBlock 加载到缓冲页
 *   3. 重新渲染缓冲页面板
 *
 * 这样可以在块与块之间观察数据传输过程。
 */
async function bufLoadFileAnimated(file, callback) {
  const totalBlocks = file.size;
  addLog('diskLog', `⏳ 加载 ${file.name} 到缓冲区 (${totalBlocks}块)...`);

  for (let i = 0; i < totalBlocks; i++) {
    const blockIdx = file.start + i;
    const content = i < file.content.length ? file.content[i] : '';

    // 高亮磁盘网格中正在读取的块（CSS reading 动画）
    const cells = document.querySelectorAll('.disk-cell');
    if (cells[blockIdx]) {
      cells[blockIdx].classList.add('reading');
      setTimeout(() => cells[blockIdx].classList.remove('reading'), 500);
    }

    // 加载到缓冲页（可能触发 LRU 置换）
    bufLoadBlock(blockIdx, file.name, content);
    bufRender();  // 实时刷新缓冲页面板

    // 延时 300ms，方便观察逐块加载过程
    await new Promise(r => setTimeout(r, 300));
  }

  addLog('diskLog', `✓ ${file.name} 全部加载到缓冲区`);
  if (callback) callback();
}
```

---

## 6. 页面置换算法（5 种）

```javascript
/**
 * 页面置换 —— 逐步执行一次页面访问
 *
 * 支持算法：LRU / FIFO / OPT / LFU / CLOCK
 *
 * 核心流程：
 *   1. 检查当前页面是否在物理帧中（命中 / 缺页）
 *   2. 命中 → 更新访问信息
 *   3. 缺页 → 有空帧则直接装入，否则按算法选择牺牲页
 */
function pageStep() {
  if (pageIndex >= pageSeq.length) return;

  const pid = pageSeq[pageIndex];  // 当前要访问的页面号

  // ---- 检查是否命中 ----
  const fi = pageMemory.findIndex(f => f && f.pageId === pid);

  if (fi !== -1) {
    // ===== 命中 =====
    pageHits++;
    pageMemory[fi].lastAccess = pageIndex;           // 更新最近访问时间（LRU 用）
    pageMemory[fi].accessCount = (pageMemory[fi].accessCount || 0) + 1; // 访问计数（LFU 用）
    if ($('pageAlgo').value === 'clock')
      pageMemory[fi].refBit = 1;                     // CLOCK 访问位置 1

    pageRenderFrames(fi, true);  // 渲染帧，标记命中动画

  } else {
    // ===== 缺页 =====
    pageFaults++;
    const algo = $('pageAlgo').value;
    const emptyIdx = pageMemory.findIndex(f => f === null);

    if (emptyIdx !== -1) {
      // ---- 有空帧，直接装入 ----
      pageMemory[emptyIdx] = {
        pageId: pid,
        lastAccess: pageIndex,
        accessCount: 1,
        dirty: Math.random() > 0.7,  // 随机标记脏页
        refBit: 1
      };
      if (algo === 'fifo') pageFifoQueue.push(emptyIdx);  // FIFO 队列记录装入顺序
      pageRenderFrames(emptyIdx, false);

    } else {
      // ---- 无空帧，按算法选择牺牲页 ----
      let victimIdx = -1;

      if (algo === 'lru') {
        // LRU：淘汰 lastAccess 最小的帧（最久未访问）
        let lruTime = Infinity;
        pageMemory.forEach((f, i) => {
          if (f.lastAccess < lruTime) {
            lruTime = f.lastAccess;
            victimIdx = i;
          }
        });

      } else if (algo === 'fifo') {
        // FIFO：淘汰最早装入的帧（队列头部）
        victimIdx = pageFifoQueue.shift();

      } else if (algo === 'opt') {
        // OPT：淘汰未来最久不会被访问的帧（理论最优）
        let farthest = -1;
        pageMemory.forEach((f, i) => {
          let nextUse = Infinity;
          // 向后扫描访问序列，找到该页面下一次出现的位置
          for (let k = pageIndex + 1; k < pageSeq.length; k++) {
            if (pageSeq[k] === f.pageId) {
              nextUse = k;
              break;
            }
          }
          if (nextUse > farthest) {
            farthest = nextUse;
            victimIdx = i;
          }
        });

      } else if (algo === 'lfu') {
        // LFU：淘汰 accessCount 最小的帧（最不经常使用）
        let minCount = Infinity;
        pageMemory.forEach((f, i) => {
          if (f.accessCount < minCount) {
            minCount = f.accessCount;
            victimIdx = i;
          }
        });

      } else if (algo === 'clock') {
        // CLOCK（时钟算法）：
        // 从 clockHand 位置开始扫描，
        //   refBit=1 → 清零并跳过（给第二次机会）
        //   refBit=0 → 选中为牺牲页
        let sweeps = 0;
        while (sweeps < pageMemory.length * 2) {
          const f = pageMemory[pageClockHand];
          if (f && f.refBit === 0) {
            victimIdx = pageClockHand;
            pageClockHand = (pageClockHand + 1) % pageMemory.length;
            break;
          }
          if (f) f.refBit = 0;  // 清引用位
          pageClockHand = (pageClockHand + 1) % pageMemory.length;
          sweeps++;
        }
        if (victimIdx === -1) victimIdx = 0;
      }

      // ---- 执行置换 ----
      const old = pageMemory[victimIdx];
      pageReplaces++;

      pageMemory[victimIdx] = {
        pageId: pid,
        lastAccess: pageIndex,
        accessCount: 1,
        dirty: Math.random() > 0.7,
        refBit: 1
      };
      if (algo === 'fifo') pageFifoQueue.push(victimIdx);
      pageRenderFrames(victimIdx, false);
    }
  }

  pageIndex++;
  // 更新统计信息
  const total = pageHits + pageFaults;
  $('statHit').textContent = pageHits;
  $('statFault').textContent = pageFaults;
  $('statRate').textContent = total
    ? (pageHits / total * 100).toFixed(1) + '%' : '0%';
  $('statReplace').textContent = pageReplaces;
}
```

---

## 7. 进程调度算法（4 种）

```javascript
/**
 * 进程调度核心 —— 支持 FCFS / SJF / RR / 优先级
 *
 * 输出：甘特图数据（gantt 数组）+ 各进程的 waitTime / turnTime
 *
 * 甘特图数据结构：[{name: 'P1', color: '#...', time: 3}, ...]
 */
function schedRun() {
  const algo = $('schedAlgo').value;
  const quantum = parseInt($('rrQuantum').value) || 2;  // RR 时间片
  let gantt = [];
  let time = 0;
  const procs = [...schedProcs].sort((a, b) => a.arrival - b.arrival);
  const colors = {};
  procs.forEach((p, i) => { colors[p.name] = COLORS[i % COLORS.length] });

  if (algo === 'fcfs') {
    // ===== FCFS：按到达顺序依次执行，不可抢占 =====
    let t = 0;
    procs.forEach(p => {
      if (t < p.arrival) {
        gantt.push({ name: '空闲', color: '#333', time: p.arrival - t });
        t = p.arrival;
      }
      gantt.push({ name: p.name, color: colors[p.name], time: p.burst });
      p.waitTime = t - p.arrival;          // 等待时间 = 开始时间 - 到达时间
      p.turnTime = t + p.burst - p.arrival; // 周转时间 = 完成时间 - 到达时间
      t += p.burst;
      p.state = 'done';
    });
    time = t;

  } else if (algo === 'sjf') {
    // ===== SJF：每次从已到达进程中选执行时间最短的 =====
    let t = 0, remaining = [...procs];
    while (remaining.length) {
      const avail = remaining.filter(p => p.arrival <= t);
      if (avail.length === 0) {
        gantt.push({ name: '空闲', color: '#333', time: 1 });
        t++;
        continue;
      }
      avail.sort((a, b) => a.burst - b.burst); // 按执行时间升序
      const p = avail[0];
      remaining.splice(remaining.indexOf(p), 1);
      if (t < p.arrival) {
        gantt.push({ name: '空闲', color: '#333', time: p.arrival - t });
        t = p.arrival;
      }
      gantt.push({ name: p.name, color: colors[p.name], time: p.burst });
      p.waitTime = t - p.arrival;
      p.turnTime = t + p.burst - p.arrival;
      t += p.burst;
      p.state = 'done';
    }
    time = t;

  } else if (algo === 'rr') {
    // ===== RR（时间片轮转）：每个进程最多执行 quantum 个时间单位 =====
    let t = 0, queue = [], idx = 0;
    const sorted = [...procs].sort((a, b) => a.arrival - b.arrival);

    while (idx < sorted.length || queue.length) {
      // 将已到达的进程加入就绪队列
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
      const exec = Math.min(quantum, p.remaining); // 本次执行时间

      gantt.push({ name: p.name, color: colors[p.name], time: exec });
      t += exec;
      p.remaining -= exec;

      // 时间片执行期间到达的进程入队
      while (idx < sorted.length && sorted[idx].arrival <= t) {
        queue.push(sorted[idx]);
        idx++;
      }

      if (p.remaining > 0) {
        queue.push(p);   // 未完成，放回队尾
        p.state = 'ready';
      } else {
        p.state = 'done';
        p.turnTime = t - p.arrival;
        p.waitTime = p.turnTime - p.burst;
      }
    }
    time = t;

  } else if (algo === 'priority') {
    // ===== 优先级调度：每次从已到达进程中选优先级最高的（数值最小） =====
    let t = 0, remaining = [...procs];
    while (remaining.length) {
      const avail = remaining.filter(p => p.arrival <= t);
      if (avail.length === 0) {
        gantt.push({ name: '空闲', color: '#333', time: 1 });
        t++;
        continue;
      }
      avail.sort((a, b) => a.priority - b.priority); // 优先级升序（越小越优先）
      const p = avail[0];
      remaining.splice(remaining.indexOf(p), 1);
      if (t < p.arrival) {
        gantt.push({ name: '空闲', color: '#333', time: p.arrival - t });
        t = p.arrival;
      }
      gantt.push({ name: p.name, color: colors[p.name], time: p.burst });
      p.waitTime = t - p.arrival;
      p.turnTime = t + p.burst - p.arrival;
      t += p.burst;
      p.state = 'done';
    }
    time = t;
  }

  // ---- 计算并展示统计指标 ----
  const n = schedProcs.length;
  const avgW = (schedProcs.reduce((s, p) => s + p.waitTime, 0) / n).toFixed(2);
  const avgT = (schedProcs.reduce((s, p) => s + p.turnTime, 0) / n).toFixed(2);
  const avgWT = (schedProcs.reduce((s, p) => s + p.turnTime / p.burst, 0) / n).toFixed(2);
  // avgW: 平均等待时间, avgT: 平均周转时间, avgWT: 平均带权周转时间
}
```

---

## 8. 银行家算法（安全序列检测）

```javascript
/**
 * 银行家算法 —— 查找安全序列
 *
 * 核心思想：
 *   反复尝试找到一个尚未完成的进程 Pi，
 *   其 Need[i] ≤ Available（当前可用资源能满足其需求），
 *   假设 Pi 执行完毕后释放其 Allocation[i] 到 Available 中，
 *   直到所有进程都完成（安全）或无法继续（不安全）。
 *
 * 数据结构：
 *   max[i][j]    —— 进程 Pi 对资源 Rj 的最大需求
 *   alloc[i][j]  —— 进程 Pi 已分配到的资源 Rj 数量
 *   need[i][j]   —— 进程 Pi 还需要的资源 Rj 数量 = max - alloc
 *   avail[j]     —— 资源 Rj 当前可用数量
 */
function bankFindSafe() {
  const { np, nr, need, alloc } = bankData;
  let avail = [...bankData.avail];     // 拷贝可用资源（避免修改原数据）
  const finish = new Array(np).fill(false); // 各进程是否已完成
  const safeSeq = [];                   // 安全序列
  let changed = true;

  // 循环扫描，直到一轮中没有任何进程可以完成
  while (changed) {
    changed = false;
    for (let i = 0; i < np; i++) {
      if (finish[i]) continue;

      // 检查 Pi 的需求是否都能被满足：Need[i][j] ≤ Available[j]
      let ok = true;
      for (let j = 0; j < nr; j++) {
        if (need[i][j] > avail[j]) {
          ok = false;
          break;
        }
      }

      if (ok) {
        // 假设 Pi 执行完毕，释放其占有的所有资源
        for (let j = 0; j < nr; j++)
          avail[j] += alloc[i][j];

        finish[i] = true;
        safeSeq.push(i);
        changed = true;
      }
    }
  }

  // 判断是否所有进程都完成
  const allDone = finish.every(f => f);
  if (allDone) {
    // 系统安全，输出安全序列（如 P1 → P3 → P0 → P2 → P4）
    // safeSeq 即为一个合法的执行顺序
  } else {
    // 系统不安全，存在死锁风险
  }
}
```

---

## 9. 资源分配图绘制

```javascript
/**
 * Canvas 绘制资源分配图（RAG）
 *
 * 图结构：
 *   - 左侧圆形节点：进程（P0 ~ Pn）
 *   - 右侧方形节点：资源（A, B, C...）
 *   - 实线箭头 R→P：分配边（资源分配给进程）
 *   - 虚线箭头 P→R：请求边（进程请求资源）
 */
function bankDrawGraph(safeSeq) {
  const canvas = $('graphCanvas');
  const ctx = canvas.getContext('2d');
  const W = canvas.width, H = canvas.height;
  ctx.clearRect(0, 0, W, H);

  const { np, nr, alloc, need, rnames } = bankData;

  // ---- 计算节点位置 ----
  const pNodes = [], rNodes = [];
  const px = 80, rx = W - 80;
  for (let i = 0; i < np; i++) {
    const y = 40 + i * (H - 80) / (Math.max(np - 1, 1));
    pNodes.push({
      x: px, y, name: 'P' + i,
      // 安全序列中的进程标绿色，否则标紫色
      color: safeSeq && safeSeq.includes(i) ? '#10b981' : '#6366f1'
    });
  }
  for (let i = 0; i < nr; i++) {
    const y = 60 + i * (H - 120) / (Math.max(nr - 1, 1));
    rNodes.push({ x: rx, y, name: rnames[i], color: '#f59e0b' });
  }

  // ---- 绘制边 ----
  ctx.lineWidth = 2;
  for (let i = 0; i < np; i++) {
    for (let j = 0; j < nr; j++) {
      if (alloc[i][j] > 0) {
        // 分配边：资源 → 进程（实线，紫色）
        ctx.strokeStyle = 'rgba(99,102,241,.6)';
        ctx.setLineDash([]);
        ctx.beginPath();
        ctx.moveTo(rNodes[j].x - 18, rNodes[j].y);
        ctx.lineTo(pNodes[i].x + 18, pNodes[i].y);
        ctx.stroke();
      }
      if (need[i][j] > 0) {
        // 请求边：进程 → 资源（虚线，红色）
        ctx.strokeStyle = 'rgba(239,68,68,.5)';
        ctx.setLineDash([4, 4]);
        ctx.beginPath();
        ctx.moveTo(pNodes[i].x + 18, pNodes[i].y);
        ctx.lineTo(rNodes[j].x - 18, rNodes[j].y);
        ctx.stroke();
      }
    }
  }
  ctx.setLineDash([]);

  // ---- 绘制节点 ----
  // 进程节点（圆形）
  pNodes.forEach(n => {
    ctx.fillStyle = n.color;
    ctx.beginPath();
    ctx.arc(n.x, n.y, 18, 0, Math.PI * 2);
    ctx.fill();
    ctx.fillStyle = '#fff';
    ctx.font = 'bold 12px sans-serif';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(n.name, n.x, n.y);
  });

  // 资源节点（方形）
  rNodes.forEach(n => {
    ctx.fillStyle = n.color;
    ctx.fillRect(n.x - 16, n.y - 16, 32, 32);
    ctx.fillStyle = '#fff';
    ctx.font = 'bold 13px sans-serif';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(n.name, n.x, n.y);
  });
}
```

---

## 10. 生产者-消费者同步模拟

```javascript
/**
 * 经典生产者-消费者问题的信号量模拟
 *
 * 三个信号量：
 *   mutex = 1    —— 互斥信号量，保证对缓冲区的互斥访问
 *   empty = N    —— 同步信号量，表示空闲槽位数（初始为缓冲区大小）
 *   full  = 0    —— 同步信号量，表示已占用的槽位数
 *
 * 生产者操作顺序：wait(empty) → wait(mutex) → 生产 → signal(mutex) → signal(full)
 * 消费者操作顺序：wait(full)  → wait(mutex) → 消费 → signal(mutex) → signal(empty)
 */
function syncStep() {
  syncState.step++;

  // 从就绪进程中随机选一个执行
  const ready = syncProcs.filter(p => p.state === 'ready');
  if (ready.length === 0) {
    // 所有进程阻塞时，尝试唤醒
    syncProcs.forEach(p => { if (p.state === 'blocked') p.state = 'ready' });
    syncRender();
    return;
  }
  const proc = ready[Math.floor(Math.random() * ready.length)];
  proc.state = 'running';

  if (proc.type === 'producer') {
    // ---- 生产者 ----

    if (syncState.empty <= 0) {
      // wait(empty) 失败：缓冲区已满，生产者阻塞
      proc.state = 'blocked';
    } else if (syncState.mutex <= 0) {
      // wait(mutex) 失败：其他进程正在访问缓冲区
      proc.state = 'blocked';
    } else {
      // 成功进入临界区
      syncState.empty--;    // wait(empty)：消耗一个空闲槽
      syncState.mutex = 0;  // wait(mutex)：加锁

      const item = 'D' + syncState.step;  // 生产一个数据项
      syncState.buffer.push(item);         // 放入缓冲区

      syncState.mutex = 1;  // signal(mutex)：解锁
      syncState.full++;     // signal(full)：增加一个已用槽
      proc.state = 'ready';
    }

  } else {
    // ---- 消费者 ----

    if (syncState.full <= 0) {
      // wait(full) 失败：缓冲区为空，消费者阻塞
      proc.state = 'blocked';
    } else if (syncState.mutex <= 0) {
      // wait(mutex) 失败
      proc.state = 'blocked';
    } else {
      // 成功进入临界区
      syncState.full--;     // wait(full)：消耗一个已用槽
      syncState.mutex = 0;  // wait(mutex)：加锁

      const item = syncState.buffer.shift();  // 从缓冲区取出数据

      syncState.mutex = 1;  // signal(mutex)：解锁
      syncState.empty++;    // signal(empty)：增加一个空闲槽
      proc.state = 'ready';
    }
  }
  syncRender();
}
```

---

## 11. 消息传递机制

```javascript
/**
 * 进程间消息传递模拟
 *
 * 使用消息队列（FIFO）实现进程间通信：
 *   - send(from, to, content)：将消息加入队列
 *   - receive()：从队列头部取出一条消息
 *
 * 消息数据结构：{from, to, content, id}
 */
let msgQueue = [];     // 消息队列
let msgIdCounter = 0;  // 消息 ID 计数器

/** 发送消息 —— 入队操作 */
function msgSend() {
  const from = $('msgFrom').value.trim() || 'P1';
  const to = $('msgTo').value.trim() || 'P2';
  const content = $('msgContent').value.trim() || 'Hello';

  msgIdCounter++;
  msgQueue.push({ from, to, content, id: msgIdCounter });
  msgRender();
  addLog('syncLog', `💬 ${from} → ${to}: "${content}" (消息#${msgIdCounter} 已入队)`);
}

/** 接收消息 —— 出队操作（FIFO） */
function msgReceive() {
  if (msgQueue.length === 0) {
    addLog('syncLog', '消息队列为空，无消息可接收');
    return;
  }
  const msg = msgQueue.shift();  // 取出队首消息
  msgRender();
  addLog('syncLog', `✓ ${msg.to} 接收来自 ${msg.from} 的消息: "${msg.content}"`);
}
```

---

## 12. 位示图(Bitmap)渲染

```javascript
/**
 * 位示图渲染 —— 将磁盘块状态映射为 64×16 网格
 *
 * 每个盘块对应一个小格子：
 *   - bm-free（绿色）：空闲块
 *   - bm-used（红色）：文件数据块
 *   - bm-dir（紫色）：目录区
 *   - bm-meta（粉紫色）：元数据区
 *
 * 与空闲盘区链互为补充，直观展示磁盘空间分配全貌。
 */
function bitmapRender() {
  const el = $('bitmapGrid');
  if (!el) return;
  let html = '';
  for (let i = 0; i < DISK_SIZE; i++) {
    const b = diskBlocks[i];
    let cls = 'bm-free';  // 默认空闲
    if (b) {
      if (b.type === 'dir') cls = 'bm-dir';
      else if (b.type === 'meta') cls = 'bm-meta';
      else cls = 'bm-used';
    }
    html += `<div class="bitmap-cell ${cls}" title="块${i}${b ? ' ' + b.file : ' 空闲'}"></div>`;
  }
  el.innerHTML = html;
}
```

---

## 13. FAT 链表视图

```javascript
/**
 * FAT 链表视图 —— 将连续分配的块号映射为 FAT 表结构
 *
 * 虽然底层采用连续存储分配，但展示层将其转换为 FAT 链表形式：
 *   块号 → 块号+1 → ... → EOF
 *
 * 同时生成 FAT 表格：
 *   块号 | Next | 所属文件
 *   24   | 25   | readme.txt
 *   25   | 26   | readme.txt
 *   26   | EOF  | readme.txt
 */
function fatRender() {
  const el = $('fatView');
  if (!el) return;
  if (files.length === 0) {
    el.innerHTML = '<span>创建文件后查看</span>';
    return;
  }
  let html = '';
  files.forEach(f => {
    html += `<div><span>📄 ${f.name}</span>`;
    html += `<div class="fat-chain">`;
    for (let i = 0; i < f.size; i++) {
      html += `<span class="fat-chain-block">${f.start + i}</span>`;
      if (i < f.size - 1) html += `<span class="fat-chain-arrow">→</span>`;
    }
    html += `<span class="fat-chain-arrow">→</span><span class="fat-chain-eof">EOF</span>`;
    html += `</div></div>`;
  });
  // FAT 表格
  html += `<table class="fat-table"><thead><tr><th>块号</th><th>Next</th><th>所属文件</th></tr></thead><tbody>`;
  files.forEach(f => {
    for (let i = 0; i < f.size; i++) {
      const next = i < f.size - 1 ? (f.start + i + 1) : 'EOF';
      html += `<tr><td>${f.start + i}</td><td>${next}</td><td>${f.name}</td></tr>`;
    }
  });
  html += `</tbody></table>`;
  el.innerHTML = html;
}
```

---

## 14. FCB/iNode 详情弹窗

```javascript
/**
 * FCB / iNode 详情弹窗 —— 展示文件的完整控制块信息
 *
 * 点击文件目录表任意行或双击目录树文件名触发。
 * 展示字段包括：
 *   - 文件名、所属用户、创建时间、权限
 *   - 起始物理块、占用块数、实际数据量
 *   - 存储分配方式、当前状态
 *   - 数据块指针列表（可视化每个物理块）
 */
function showFCB(name) {
  const f = files.find(x => x.name === name);
  if (!f) return;
  diskSelectFile(name);

  const usedBytes = f.content
    ? f.content.reduce((s, c) => s + c.length, 0) : 0;

  let html = `<div class="fcb-row"><span class="fcb-key">📝 文件名</span>`
    + `<span class="fcb-val">${f.name}</span></div>`;
  html += `<div class="fcb-row"><span class="fcb-key">👤 所属用户</span>`
    + `<span class="fcb-val">${f.owner}</span></div>`;
  html += `<div class="fcb-row"><span class="fcb-key">📅 创建时间</span>`
    + `<span class="fcb-val">${f.time}</span></div>`;
  html += `<div class="fcb-row"><span class="fcb-key">🔒 权限</span>`
    + `<span class="fcb-val">${f.perm}</span></div>`;
  html += `<div class="fcb-row"><span class="fcb-key">📍 起始物理块</span>`
    + `<span class="fcb-val">${f.start}</span></div>`;
  html += `<div class="fcb-row"><span class="fcb-key">📦 占用块数</span>`
    + `<span class="fcb-val">${f.size}</span></div>`;
  html += `<div class="fcb-row"><span class="fcb-key">💾 实际数据量</span>`
    + `<span class="fcb-val">${usedBytes}B / ${f.size * BLOCK_SIZE}B</span></div>`;
  html += `<div class="fcb-row"><span class="fcb-key">📐 存储分配</span>`
    + `<span class="fcb-val">连续分配</span></div>`;
  html += `<div class="fcb-row"><span class="fcb-key">🔄 状态</span>`
    + `<span class="fcb-val">${fileInUse === f.name ? '🟡 使用中' : '🟢 空闲'}</span></div>`;

  // 数据块指针
  html += `<div class="fcb-blocks">`;
  for (let i = 0; i < f.size; i++) {
    const hasData = f.content && f.content[i] && f.content[i].length > 0;
    html += `<span class="fcb-block">块${f.start + i}${hasData ? '' : ' (空)'}</span>`;
  }
  html += `</div>`;

  $('fcbContent').innerHTML = html;
  $('fcbOverlay').style.display = 'flex';
}
```

---

## 15. 五种算法缺页率对比

```javascript
/**
 * 五种页面置换算法缺页率对比
 *
 * 对同一组访问序列，分别用 LRU / FIFO / OPT / LFU / CLOCK
 * 五种算法独立运行，统计缺页次数和缺页率，
 * 用彩色柱状图可视化对比结果。
 *
 * 答辩时可一键展示，直观说明不同算法的性能差异。
 */
function pageCompareAlgorithms() {
  if (pageSeq.length === 0) { /* 先生成序列 */ }

  const algos = ['lru', 'fifo', 'opt', 'lfu', 'clock'];
  const k = parseInt($('pageFrames').value) || 4;
  const results = {};

  algos.forEach(algo => {
    let mem = new Array(k).fill(null);
    let hits = 0, faults = 0, replaces = 0;
    let fifoQ = [], clockHand = 0;

    pageSeq.forEach((pid, idx) => {
      const fi = mem.findIndex(f => f && f.pageId === pid);
      if (fi !== -1) {
        hits++;  // 命中
        // 更新访问信息...
      } else {
        faults++;  // 缺页
        const emptyIdx = mem.findIndex(f => f === null);
        if (emptyIdx !== -1) {
          // 有空帧，直接装入
        } else {
          // 按算法选择牺牲页并置换
          // LRU: 淘汰 lastAccess 最小的
          // FIFO: 淘汰最早装入的
          // OPT:  淘汰未来最久不访问的
          // LFU:  淘汰 accessCount 最小的
          // CLOCK: 扫描 refBit=0 的帧
          replaces++;
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

  // 渲染柱状图 + 命中率排行
  renderCompareChart(results);
}
```

---

## 16. 共享内存通信

```javascript
/**
 * 共享内存通信模拟
 *
 * 与消息传递（FIFO 队列）互补，共享内存是最常见的 IPC 方式之一。
 * 进程通过读写共享变量进行通信：
 *   - shmWrite()：将值写入共享内存
 *   - shmRead()：从共享内存读取值
 *
 * 数据结构：
 *   shmData = { value: 共享变量值, writer: 写入进程名 }
 */
let shmData = { value: 0, writer: '', reader: '' };

/** 写入共享内存 */
function shmWrite() {
  const proc = $('shmProc').value.trim() || 'P1';
  const val = parseInt($('shmData').value) || 0;
  shmData.value = val;
  shmData.writer = proc;
  $('shmValue').textContent = val;
  // 动画反馈 + 日志记录
  addLog('syncLog', `🧠 ${proc} 写入共享内存: Value = ${val}`);
}

/** 读取共享内存 */
function shmRead() {
  const proc = $('msgTo').value.trim() || 'P2';
  // 动画反馈 + 日志记录
  addLog('syncLog',
    `🧠 ${proc} 读取共享内存: Value = ${shmData.value}` +
    (shmData.writer ? ` (由 ${shmData.writer} 写入)` : ''));
}
```

---

## 17. 管道通信

```javascript
/**
 * 管道通信模拟 —— 有界缓冲区 + 读写端阻塞
 *
 * 管道是经典的 IPC 方式：
 *   - 固定容量的字节数组（pipeBuf）
 *   - FIFO 顺序：写入追加到尾部，读取从头部取出
 *   - 管道满时写端阻塞，管道空时读端阻塞
 *
 * 与消息队列、共享内存共同构成三种 IPC 方式。
 */
let pipeBuf = [];       // 管道缓冲区
let pipeCapacity = 6;   // 管道容量

/** 写入管道 —— 满时阻塞 */
function pipeWrite() {
  const writer = $('pipeWriter').value.trim() || 'P1';
  const data = $('pipeData').value.trim() || 'A';

  if (pipeBuf.length >= pipeCapacity) {
    // 管道满，写端阻塞
    addLog('syncLog', `🔧 ${writer} 写入阻塞: 管道已满`);
    return;
  }
  pipeBuf.push(data);    // 追加到尾部
  pipeRender();
  addLog('syncLog', `🔧 ${writer} 写入管道: "${data}" (${pipeBuf.length}/${pipeCapacity})`);
}

/** 读取管道 —— 空时阻塞（FIFO） */
function pipeRead() {
  if (pipeBuf.length === 0) {
    // 管道空，读端阻塞
    addLog('syncLog', `🔧 读取阻塞: 管道为空`);
    return;
  }
  const data = pipeBuf.shift();  // 从头部取出
  pipeRender();
  addLog('syncLog', `🔧 读端取出: "${data}" (剩余 ${pipeBuf.length}/${pipeCapacity})`);
}
```

---

## 18. 命令进程调度

```javascript
/**
 * 命令进程模式 —— 文件操作生成命令进程，经调度后执行
 *
 * 开启进程模式后，文件操作（新建/删除/查看/修改）不直接执行，
 * 而是生成命令进程节点进入队列：
 *   Ready (就绪) → Running (运行) → Done (完成)
 *
 * 实现文件系统与进程管理的耦合展示。
 */
let cmdQueue = [];    // 命令进程队列
let cmdIdCounter = 0; // 进程 ID 计数器

/** 入队 —— 将文件操作包装为命令进程 */
function cmdEnqueue(action, label) {
  const id = ++cmdIdCounter;
  cmdQueue.push({ id, action, label, status: 'ready' });
  cmdRender();
}

/** 调度执行单个命令进程 */
async function cmdDispatchOne(id) {
  const cmd = cmdQueue.find(c => c.id === id);
  if (!cmd || cmd.status !== 'ready') return;

  cmd.status = 'running'; cmdRender();          // Ready → Running
  await new Promise(r => setTimeout(r, 1200));  // 模拟调度延时

  switch (cmd.action) {  // 执行实际文件操作
    case 'create': diskCreateFile(); break;
    case 'delete': diskDeleteFile(); break;
    case 'view':   diskViewFile();   break;
    case 'edit':   diskEditFile();   break;
  }

  cmd.status = 'done'; cmdRender();             // Running → Done
}

/** 调度全部就绪进程 */
async function cmdDispatchAll() {
  const ready = cmdQueue.filter(c => c.status === 'ready');
  for (const cmd of ready) {
    await cmdDispatchOne(cmd.id);
    await new Promise(r => setTimeout(r, 400));
  }
}
```

---

## 19. 导览演示模式

```javascript
/**
 * 一键演示 —— 自动导览 6 个步骤，展示所有模块核心功能
 *
 * 演示流程:
 *   Step 1: 磁盘示例数据加载
 *   Step 2: 查看文件 (缓冲页动画)
 *   Step 3: 页面置换算法对比
 *   Step 4: 进程调度 RR
 *   Step 5: 银行家算法
 *   Step 6: 生产者-消费者同步
 */
let demoRunning = false;
let demoAbort = false;

async function startDemoTour() {
  if (demoRunning) return;
  demoRunning = true; demoAbort = false;
  $('demoBar').classList.add('visible');
  const total = 6;

  try {
    demoUpdate(1, total, '加载磁盘示例数据');
    switchTab('disk'); await demoWait(2000);
    loadDiskSample(); await demoWait(4000);

    demoUpdate(2, total, '查看文件内容');
    selectedFile = 'readme.txt'; diskViewFile();
    await demoWait(5000); closeFileView();

    demoUpdate(3, total, '页面置换算法对比');
    switchTab('page'); await demoWait(1500);
    pageGenerate(); await demoWait(3000);
    pageCompareAlgorithms(); await demoWait(4000);

    demoUpdate(4, total, '进程调度 (RR)');
    switchTab('sched'); await demoWait(1500);
    loadSchedSample(); await demoWait(2000);
    schedRun(); await demoWait(4000);

    demoUpdate(5, total, '银行家算法');
    switchTab('dead'); await demoWait(1500);
    loadBankSample(); await demoWait(2000);
    bankFindSafe(); await demoWait(4000);

    demoUpdate(6, total, '生产者-消费者同步');
    switchTab('sync'); await demoWait(1500);
    syncStart(); await demoWait(4000);

    demoUpdate(total, total, '✓ 演示完成！');
    await demoWait(3000);
  } catch (e) { /* aborted */ }

  demoRunning = false;
  $('demoBar').classList.remove('visible');
}

/** 可中断延时 */
async function demoWait(ms) {
  for (let i = 0; i < ms / 100; i++) {
    if (demoAbort) throw new Error('aborted');
    await new Promise(r => setTimeout(r, 100));
  }
}

function stopDemo() { demoAbort = true; }
```

---

> 以上代码均摘自 `simulator.html`，为各功能模块的核心逻辑部分。
> 完整实现（含 UI 渲染、CSS 动画等）请查看源文件。
