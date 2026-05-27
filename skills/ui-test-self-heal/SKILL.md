---
name: ui-test-self-heal
description: 写 Playwright UI 用例 + 自动诊断失败根因。两层覆盖：(L1 页面级)"探索→编写→执行→归因→修复"五步闭环，控件可用性；(L2 业务级)从 PRD/接口契约/真实埋点派生 E2E 业务流，含数据流三方对账、多角色权限、时序边界。区分"功能 Bug vs 脚本问题"，根因级修复而非单条打补丁。Use when 用户要求：(1) 写/补充 UI 自动化用例 (2) UI 测试失败需要诊断 (3) 探索页面后再写用例 (4) 给已有用例脚本"自愈/修复" (5) 跑自动化后出测试报告 (6) 设计端到端业务流测试 (7) 数据一致性/计算正确性验证 (8) 多角色权限测试 (9) 并发/重复提交边界测试。关键词：UI 自动化、Playwright 用例、自愈、用例修复、根因分析、E2E、业务流、数据一致性、权限测试、并发边界、PRD 派生用例。
---

# UI 测试自愈闭环

## 何时用

用户让你 写/改/跑 Playwright UI 用例，且：
- 用例首次写，需要先**探索页面**；或
- 用例**有失败**，要判断是 Bug 还是脚本问题、并修复脚本问题；或
- 要在已有用例基础上**补充覆盖**。

不适合：纯接口测试、单元测试、性能测试。

## 核心原则（不要违反）

1. **先 probe 后 run** — 不要凭假设写用例。先写一个只 dump DOM 的探索脚本，跑一次，再写断言。
2. **每条失败必须归因** — FAIL 分两类：**功能 Bug**（开发要修） vs **脚本问题**（你要修）。不归因不重跑。
3. **根因级修复** — 一类 selector 错就 sed 全替；不要逐条 selector 改。
4. **截图 + 网络日志缺一不可** — 没有这两样，失败时无法归因，只能瞎猜。

## 判定"Bug vs 脚本问题"（3 条全满足才能盖章脚本问题）

1. 截图里能看到该功能正常工作
2. 手工复现操作能成功
3. 网络请求实际发出且返回 200 / code=0

3 条都满足 → 改脚本。否则按 Bug 上报，**不要改脚本掩盖**。

## 五步闭环

### ① 探索（probe/probeN.js）

写**只 dump DOM、不做断言**的脚本。每个 probe 单独一个文件，编号递增。要 dump：

```js
// 必查 6 项
{
  tabs:        document.querySelectorAll('.ant-tabs-tab').length,
  topButtons:  Array.from(document.querySelectorAll('button')).slice(0,20).map(b=>b.innerText.trim()),
  filterSelects: Array.from(document.querySelectorAll('.ant-select:not(.ant-select-disabled)')).map(s=>({
    placeholder: s.querySelector('.ant-select-selection-placeholder')?.innerText,
    mode: s.className.includes('ant-select-multiple') ? 'multi' : 'single',
  })),
  thead:       Array.from(document.querySelectorAll('.ant-table-thead th')).map(t=>t.innerText.trim()),
  // 关键：dump 第一行最后一个 cell 的 outerHTML，看操作列真实结构
  lastCellHtml: document.querySelector('.ant-table-row')?.querySelectorAll('.ant-table-cell')?.[..]?.outerHTML?.slice(0,2000),
  // 还要确认是不是 div 模拟表格（虚拟滚动）：
  rowTag: document.querySelector('.ant-table-row')?.tagName,  // 'TR' 还是 'DIV'
}
```

**特别注意**：很多新 Antd 是 `<div class="ant-table-row" data-row-key="...">` + `<div class="ant-table-cell">`（虚拟滚动），不是 `<tr>` `<td>`。selector 用 `.ant-table-row[data-row-key]` + `.ant-table-cell`。

### ② 编写（run/runN.js）

脚本顶部写**用例矩阵注释**（U1~Un），每条一行说明意图。然后：

```js
const results = [];
const rec = (id, name, expect, actual, pass, note='') => {
  results.push({id, name, expect, actual: String(actual).slice(0,400), pass, note});
  console.log(pass?'✅':'❌', id, name, '| expect:', expect, '| actual:', actual);
};
// 每条用例
async function screen(n) { await page.screenshot({path:`${OUT}/sN-${n}.png`, fullPage:true}); }
// 网络拦截：所有相关请求落盘
const apiLog = [];
page.on('response', async r => { if (r.url().includes('/api/')) {
  apiLog.push({s:r.status(), u:r.url(), resp:(await r.text().catch(()=>''))?.slice(0,2000)});
}});
// 收尾
fs.writeFileSync(`${OUT}/runN-results.json`, JSON.stringify({summary:{...}, results}, null, 2));
fs.writeFileSync(`${OUT}/runN-apilog.json`, JSON.stringify(apiLog, null, 2));
```

工具函数抽进 lib/，全部用例复用。常见：`screen()` / `dismissOverlays()` / `pickMulti()` / `findRowDirect()` / `openCreateModal()` / `waitToast()`。

### ③ 执行

跑完必有 4 类产物在 `out/`：
- `sN-*.png` 截图（命名带用例 id）
- `runN-results.json` 结构化结果
- `runN-apilog.json` 网络流水
- `runN.log` 文本日志

### ④ 归因（**最关键的一步**）

跑完拿 FAIL 列表，按**根因**归类，不按"用例"归类。常见根因桶：

| 根因 | 典型现象 | 修复成本 |
|---|---|---|
| selector 整套错 | 多个用例 `count=0` / `not found` 但截图正常 | 1 次 sed 全改 |
| 时机/race | 切 Tab 后立刻查 → 0；多等一会就有 | 改 waitFor / polling |
| 按钮文本嵌套 span | `has-text("X")` 找不到但截图有 X 按钮 | 改 `text-is` 或 `>>text=/^X$/` |
| 弹窗未关闭 | 后续用例都从一个错误起点开始 | 强制 dismissOverlays |
| 接口名变了 | save=undefined | 改 waitForResponse pattern |
| 真 Bug | 截图就是错的 / 手工也复现 | **不改脚本**，报 Bug |

3 类根因覆盖 8 个失败，比 8 个独立修补丁省 80% 时间。

### ⑤ 修复 + 回到①

根因级修复举例（实际用过的）：
```bash
# div 表格 selector 错（一次替换 11 处）
sed -i '' "s|\.ant-table-tbody tr\.ant-table-row|.ant-table-row[data-row-key]|g" run/*.js
```

修完**重跑同一个脚本**，验证根因判断。若新 FAIL 出现 → 回到 ① 再 probe 一次。

## 标准目录布局（强烈建议）

```
tmp/  (或任何工作目录)
├── README.md              # 自文档化：使用流程
├── package.json           # scripts: login / test / report / probe:*
├── probe/                 # 探索脚本（不断言）
│   └── probeN.js
├── run/                   # 用例 + 报告生成
│   ├── runN.js
│   └── gen-report.js
├── lib/                   # 复用工具 + 登录
│   └── relogin.js
├── out/                   # 所有产物（gitignored）
└── (profile 放 /tmp/<project>/profile，不放项目里)
```

## 报告生成（run/gen-report.js）

读 `out/runN-results.json` → 出 HTML 报告，必须包含：
1. 顶部 4 个卡片：总数/通过/失败/通过率 + 进度条
2. ❌ 失败用例区，每条带 **根因** + **修复方案** 两个块（黄/蓝条）
3. 分组（groupMap）展示，便于对照功能模块
4. 每条用例旁边挂相关截图链接
5. 末尾 TODO 区：列出尚未覆盖的功能点

报告 = 给人看 + 给下一轮 Claude 看，根因栏必填，否则下次还得重判。

## 已知反模式（踩过的坑，别再踩）

1. **第一次直接写大用例脚本** — 假设大概率错。先 probe 一次成本 5 分钟，省后面 1 小时。
2. **headless 模式跑 SSO 站点** — 二次认证不工作。用 headed + persistent profile。
3. **probe 和 run 共用一个 profile** — probe 死了浏览器，run 时 SingletonLock 残留卡死。删 `<profile>/SingletonLock` 即可。
4. **没归因就动手改** — 8 个失败逐个改 = 8 倍工作量，归因后可能 2 类根因覆盖全部。
5. **截图证明功能正常但你改脚本绕过** — 等于掩盖 Bug。3 条判定必须全过才改脚本。
6. **删除按钮真的去点** — 测试约束就是"完整提交但不删除"。删除类操作只验 tooltip 存在，不点。
7. **ctx.close() 后 session 丢** — UAT 的 SSO 常是 session-scope cookie。设计上接受"每次跑前可能要重登"，或用 storageState dump/restore。

## 调用示例

用户："用 Playwright 给 XX 页面写 UI 自动化"
你应该：
1. 先创建 probe/ run/ lib/ out/ 目录 + package.json
2. 写 lib/relogin.js（弹浏览器手动登录，保存 persistent profile）
3. 写 probe/probe1.js dump 顶层 DOM，让用户登录后跑一次
4. 根据 probe 结果写 run/run1.js 用例矩阵（注释里列 U1~Un）
5. 跑 run1 → 看 FAIL → 按根因归类 → 根因级修复 → 重跑
6. 写 run/gen-report.js 出 HTML 报告（含根因栏 + TODO 区）

用户："这个用例脚本有失败，帮我修"
你应该：
1. 读 `out/*-results.json` 和 `*.log`
2. 看每个 FAIL 对应的截图 + 网络日志，对照"3 条判定"
3. 按根因桶归类
4. 根因级修复（首选 sed）
5. 重跑验证

## 参考实现

项目 `tmp/` 目录是这套方法的完整实现样例：
- `tmp/probe/probe6.js probe8.js probe9.js`
- `tmp/run/run6.js run/gen-final-report.js`
- `tmp/lib/relogin.js`
- `tmp/out/report-final.html`（含根因栏）
- `tmp/README.md`（使用说明）

直接复制改路径即可起一个新项目的 UI 测试。

---

# L2：业务级用例（深入业务流 / 数据流）

> 上面 L1 验证"控件能用"，**L2 验证"用户走完业务后系统状态对"**。L1 是必要不充分，L2 才是测试真正要回答的问题。
>
> **判断要不要做 L2**：被测页面是不是有"提交后跨页面/跨角色/跨时间产生影响"？如果是（绝大多数后台系统都是），仅做 L1 等于没测。

## L2 的 4 个维度

| 维度 | 回答的问题 | 关键产物 |
|---|---|---|
| **业务流** | 多步跨页面流程能走通吗？前一步生成的数据后一步能用吗？ | 流程图 + flow 编排脚本 |
| **数据流** | UI 看到的 = 接口返回 = DB 落库？金额/数量算对了吗？ | 三方对账断言 |
| **权限** | 不同角色看到的功能/数据是不是符合矩阵？越权能否拦截？ | 角色 × 操作矩阵 |
| **时序边界** | 重复提交/并发改/过期/断网时表现对吗？ | 故障注入用例 |

## L2 的探索阶段（在 L1 的 probe 之上加 3 步）

L1 探索页面 DOM；L2 还要探索**业务模型**：

### probe-prd.js（PRD/需求文档驱动）

读 PRD，自动抽出：
- **业务实体**（策略、订单、用户、库存…）+ 它们的字段
- **状态机**（草稿 → 待审 → 审核中 → 已生效 → 已停用 → 已归档）
- **跨实体规则**（"策略生效后 → 海外预估售价页能查到对应 SKU"）
- **计算规则**（利润率 / 优惠金额 / 库存扣减公式）—— 这些是断言的金矿
- **角色矩阵**（管理员 / 普通用户 / 财务 / 审核员 各能做什么）

如果用户没给 PRD，**主动让用户提供**或读项目里 `docs/` `requirements/` 之类目录。已有 skill `test_case_generate` 可联动派生初版用例。

### probe-api.js（接口契约驱动）

```js
// 在 page.on('response') 基础上聚类：
// 1) 每个业务操作对应哪些接口（提交时打了 /save + /list + /detail 三个）
// 2) 接口的 query/body schema（哪些字段，类型）
// 3) 接口返回的 data schema（哪些字段会展示在哪）
```

输出 `out/api-contract.json`：每个业务操作 → 接口集合 → 请求/响应 schema。**这是后面"三方对账"断言的依据**。

### probe-flows.js（真实用户路径）

如果有 access log / 埋点 / Sentry，dump top-N 用户路径。优先覆盖 80% 流量的路径，而不是把所有页面都点一遍。没有埋点就让用户口述"最常用的 3 条流程是什么"。

## L2 用例编写（按 flow 组织，不按页面）

### 编排模式

L1 是 `runN.js` 一个文件 N 个独立用例；L2 是 `flows/<flow-name>.js` 一个文件 = 一条端到端流程，**前一步的输出是后一步的输入**：

```js
// flows/strategy-lifecycle.js
async function flow() {
  const ctx = { ts: Date.now(), strategyId: null, strategyName: `E2E-${Date.now()}` };

  // Step 1: 创建（管理员角色）
  await loginAs('admin');
  ctx.strategyId = await createStrategy(ctx.strategyName, { profit: 5, quantity: 1 });
  rec('F1.1', '创建返回 id', '>0', ctx.strategyId, ctx.strategyId > 0);

  // Step 2: 提交审核
  await submitForReview(ctx.strategyId);
  rec('F1.2', '提交后状态变 待审', '待审', await getStatus(ctx.strategyId), ...);

  // Step 3: 切角色审核（审核员）
  await loginAs('auditor');
  await approve(ctx.strategyId);

  // Step 4: 切回管理员验证生效
  await loginAs('admin');
  // 三方对账：UI / 接口 / 数据库
  const uiStatus = await readRowStatus(ctx.strategyName);
  const apiStatus = (await api.detail(ctx.strategyId)).status;
  const dbStatus = await db.query(`select status from strategy where id=?`, ctx.strategyId);
  rec('F1.4', '三方状态一致', `ui=api=db=生效`, `ui=${uiStatus} api=${apiStatus} db=${dbStatus}`,
      uiStatus === apiStatus && apiStatus === String(dbStatus));

  // Step 5: 跨页面影响验证
  await goto('/sales-price');
  const appears = await searchAndCheck(ctx.strategyName);
  rec('F1.5', '生效策略出现在销售价页', '可查到', appears, appears);
}
```

**3 条编排纪律**：
1. **flow 内共享 ctx**（id / 时间戳 / 名字），别用全局变量
2. **每一步是一个 rec()**，失败时能精确定位到哪一步断
3. **失败就停**（不像 L1 可继续）—— 第 2 步失败时第 3 步无意义

### 数据流：三方对账（最关键的断言）

UI 测试最常见的隐藏 Bug 是 **"UI 显示对了，但 DB 错了"** 或反过来。必须三方对账：

```js
// 提交利润策略后断言金额
const formProfit = await modal.input.profit.inputValue();    // ① UI 输入
await submit();
const apiResp = await waitFor('/strategy/save');             // ② 接口请求 body
const dbRow = await db.query('select profit from ...');      // ③ DB 落库（如不通 db 至少调 GET 详情接口）

const all = [Number(formProfit), apiResp.body.profit, Number(dbRow.profit)];
rec('D1', '利润率三方一致', '相等', all.join('/'), new Set(all).size === 1);
```

**没 DB 连接也要做**：`POST /save` 后立刻 `GET /detail`，比较二者。这能抓住"前端骗你"的 Bug（接口收了但没存）。

**计算公式必断**：PRD 里所有"X = a × b + c"的公式都要写成断言。例：
- 利润率 × 售价 = 利润额
- 库存 - 已售 = 可售
- 折后价 = 原价 × 折扣
错一个分都是钱。

### 权限：角色 × 操作矩阵

不是给每个角色都跑一遍所有用例，而是写一个**矩阵表**，每格一个用例：

```js
const matrix = {
  admin:      { create: 'allow', edit: 'allow', delete: 'allow', view: 'allow' },
  auditor:    { create: 'deny',  edit: 'deny',  delete: 'deny',  view: 'allow', approve: 'allow' },
  viewer:     { create: 'deny',  edit: 'deny',  delete: 'deny',  view: 'allow' },
};
for (const [role, ops] of Object.entries(matrix)) {
  await loginAs(role);
  for (const [op, expected] of Object.entries(ops)) {
    const visible = await isButtonVisible(op);  // 前端隐藏
    const callable = await tryCallApi(op);       // 直调接口
    const ok = expected === 'allow' ? (visible && callable === 200) : (!visible && callable === 403);
    rec(`P-${role}-${op}`, `${role} ${op}`, expected, `vis=${visible} api=${callable}`, ok);
  }
}
```

**关键**：要同时测**前端隐藏 + 接口拦截**。只测前端隐藏 = 把"伪安全"当安全（用户用 Postman 直调就穿了）。

### 时序/并发/边界

固定的几类必须覆盖：

```js
// 1) 重复提交（按钮防抖 + 后端幂等）
await page.locator('button:has-text("确定提交")').click();
await page.locator('button:has-text("确定提交")').click({ force: true });   // 立刻再点
const count = await api.list({name: ctx.name});
rec('T1', '重复提交不产生 2 条', '=1', count, count === 1);

// 2) 并发改同一条（乐观锁）
await Promise.all([
  ctxA.editQuantity(id, 5),
  ctxB.editQuantity(id, 8),
]);
// 期望：一个成功一个报"版本冲突"

// 3) 长时间挂起后再提交（token 过期续期）
await page.waitForTimeout(30 * 60 * 1000);  // 30 分钟
await submit();
// 期望：自动续期或友好提示重登，不是 500

// 4) 断网恢复（用 ctx.route 模拟）
await ctx.route('**/save', r => r.abort());
await submit();
await ctx.unroute('**/save');
await submit();   // 期望：第二次能成
```

## L2 失败的根因桶（在 L1 之上扩展）

| 根因桶 | 典型现象 | 判定方法 |
|---|---|---|
| **业务规则错** | 计算结果 UI ≠ PRD 公式 | 拿 PRD 公式手算一遍对比 |
| **数据未落库** | UI 显示成功但 GET 详情查不到 | 看接口 response 是不是真的成功 |
| **跨页面缓存陈旧** | A 页面改了 B 页面不刷新 | 看是否走了缓存接口 |
| **状态机非法跳转** | 待审能直接点编辑（应只允许撤回） | 比对 PRD 状态机 |
| **权限前端隐藏但后端没拦** | 按钮不可见但直调接口 200 | 必查项 |
| **金额浮点误差** | 0.1 + 0.2 != 0.3 | 用 Decimal / 整数分计算 |
| **时区/日期错位** | 跨时区显示差 1 天 | 看接口返回时间格式 |
| **并发冲突未处理** | 后写覆盖先写不报错 | 必查项 |

## L2 报告增项

`gen-report.js` 在 L1 报告基础上加：

1. **流程图**（mermaid）：每条 flow 一张图，节点标 PASS/FAIL
2. **数据流追踪表**：UI 值 / 接口 body / DB 值 / 是否一致 / 差异点
3. **权限矩阵热力图**：role × operation 表格，绿/红/黄
4. **覆盖度统计**：PRD 业务规则 N 条 → 被断言 M 条 → 覆盖率 M/N

## L1 → L2 升级路径（用户已有 L1 用例时）

不要重写，**渐进升级**：

1. 先按业务流分组现有用例（同一 flow 的 U1 U5 U11 合并成 F1）
2. 给每条加 ctx 共享：把硬编码的名字/id 改成 `ctx.name`
3. 在每个"提交"动作后追加 GET 详情对账（双方对账即可，DB 视情况）
4. 加 1 条权限矩阵用例 + 1 条重复提交用例 = 最小 L2 增量
5. 全部跑通后再加复杂的并发/计算公式断言

## 调用示例（升级后）

用户："给 XX 后台系统写完整 UI 自动化"
你应该：
1. **先问 / 找 PRD** — 没 PRD 就让用户口述核心业务流（"用户最常做的 3 件事"）
2. **probe 三件套** — DOM 探索（L1）+ PRD 业务模型（L2）+ 接口契约（L2）
3. **画流程图**给用户确认覆盖范围
4. **先 L1 后 L2** — L1 保证控件可用作为基线（也是 L2 的前置条件），再写 flow
5. **每个 flow 必含**：三方对账 + 至少 1 条权限断言 + 至少 1 条时序边界
6. **报告**含流程图 + 数据流表 + 权限矩阵 + 覆盖度

用户："这个用例只验了能点能填，加上业务断言"
你应该：
1. 读用例脚本，识别每个"提交"动作
2. 在提交后插入 GET 详情 + 字段比对
3. 把 PRD 里的计算公式抽 1-2 条做断言示范
4. 跑一遍，把"显式正确但隐式错"的 Bug 抓出来 —— 这是 L2 的最大价值

## L2 反模式（特有的坑）

1. **flow 步骤之间用 sleep 而不是 waitForResponse** — 时序不稳，10 次跑 1 次假阳
2. **三方对账只比对一个字段** — 应该比对**整个对象**（漏字段是常见 Bug）
3. **权限测试只切 UI 不切 token** — 你以为切了角色，其实是同一个 token，等于没测
4. **并发用例用 page.locator 并发** — 同一 page 实例不能真并发，要起两个 BrowserContext
5. **断言计算结果时用 ==** — 浮点比较必须 `Math.abs(a-b) < 0.01` 或转整数分
6. **flow 用 try/catch 吞错继续** — 第 2 步失败时第 3 步必然错，应立即 fail-fast
7. **不区分"前端规则"和"后端规则"** — 前端校验 0 < 数量 ≤ 1000 应在前端断言；后端幂等应在后端断言。混在一起出问题时不知道改哪边。

## L2 参考骨架

```
tmp/
├── probe/
│   ├── probe-dom.js        # L1：DOM 结构
│   ├── probe-prd.js        # L2：从 PRD 抽业务模型 → out/business-model.json
│   ├── probe-api.js        # L2：聚类接口 → out/api-contract.json
│   └── probe-flows.js      # L2：识别 top-N 用户路径
├── run/
│   ├── unit/               # L1：单页面用例
│   │   └── runN.js
│   └── flows/              # L2：业务流用例
│       ├── strategy-lifecycle.js     # 端到端生命周期
│       ├── permission-matrix.js      # 权限矩阵
│       ├── concurrency.js            # 并发/重复提交
│       └── data-consistency.js       # 三方对账
├── lib/
│   ├── relogin.js
│   ├── auth.js             # loginAs(role) 多角色切换
│   ├── api-client.js       # 直调接口（用于对账 + 越权测试）
│   ├── db-client.js        # DB 直读（可选）
│   └── flow-runner.js      # ctx 注入 + fail-fast 编排
└── out/
    ├── business-model.json
    ├── api-contract.json
    ├── flows-*.png         # 每条 flow 截图集
    ├── consistency-*.json  # 三方对账详情
    └── report-final.html   # 含流程图/数据流表/权限矩阵
```
