# 2026-09-18 Electron 厚客户端 v6→v7 许可体系迁移与汉化补丁修复（StarUML 7.1.1）

## 场景分类

- 类别: Electron / 厚客户端 / 许可校验迁移 / 本地化补丁工程
- 目标: 自持项目 StarUML 汉化激活工具（v6 时代产物）→ 适配 StarUML 7.1.1
- 平台: macOS 15 (Darwin 24.0.0, arm64)
- 授权: own_system + offline-sample

## 目标摘要

把一份只在 StarUML 6.x 上有效的「激活 + 汉化」脚本，修复为能正确支持 7.1.1 的版本，
要求汉化与激活在真实应用上可验证生效，且不破坏源码语法。

## 关键发现

### 1. asar 不是"加密黑盒"

`npx @electron/asar extract` 直接得到完整源码树（7987 条目），`package.json` 里的
`version` / `productId` 就是判定版本策略的最可靠依据。

### 2. 同一产品跨大版本，许可体系可以整体重写

| | v6 | v7 |
|---|---|---|
| 源码 | `src/engine/license-manager.js` | `src/utils/license-client.js` |
| 凭证 | `license.key`（明文 JSON） | `activation.key`（AES-256-GCM） |
| 自签 | `SK` + SHA-1 | 无（改为 webcrypto 解密常量密钥） |
| 交互 | `$.post` 劫持可拦截 | 全部走 IPC，无前端网络调用 |
| 评估 | 无 | `lib.so` + 30 天 |

**教训**：迁移类任务不要假设"文件名相同 = 逻辑相同"。必须确认新版本的**调用点**。
`grep -rn '\$\.post' src/` 返回空，就足以判定基于 `$.post` 劫持的方案已死。

### 3. "本地校验通过"不等于"激活能存活"

v7 的 `remoteValidate()` 有一个反直觉分支：

```js
if (deviceId === "*") {                       // 离线激活
  const response = await fetch(`${LICENSE_SERVER_URL}/ping`, { method: "POST" });
  if (response.ok) {
    await localDeactivate();                  // 删除 activation.key
    return { success: false, message: "License deactivated (illegal offline use)" };
  }
}
```

即：**本地加密校验通过后，只要服务器可达就被主动撤销**。
这解释了用户反馈的"激活成功但下次打开又变试用版"。

**定位方法（可复用）**：
1. 写入合法凭证 → 启动应用 → 检查凭证文件是否还存在
2. 若被删除，在源码里搜 `unlink` / `deactivate`，定位到删除点
3. 用 `curl` 确认触发条件（本例 ping 返回 200）

### 4. 运行时 monkey-patch 在解构导入下无效

```js
// src/main-process/application.js
const { getDeviceId, remoteActivate, remoteDeactivate, remoteValidate } =
  require("../utils/license-client");
```

解构在模块加载时完成，之后再覆写 `module.exports.remoteValidate` 不会影响已绑定的引用。
**必须做源码级替换**。最初写的运行时补丁是无效的，白做一轮。

### 5. 短字面量全局替换是本地化补丁的经典陷阱

原脚本在 en 未命中时退化为裸 `content.replace(en, cn)`，导致同文件内
`"Model",` / `"Close",` / `"Save",` 被多处误伤；
更严重的是把 `title += ' (UNREGISTERED)'\n    }` 这种**跨行含代码的分片**当作文案替换，
直接把 `else { ... }` 注入到 JS 里。

**对策**：
- 优先用带 key 的精确匹配（`"label": "About"`）
- 确实需要裸替换时，先统计该串在文件内的出现次数，>1 时要求人工确认
- 替换后必须做全量语法自检（本例 652 个 JS 文件跑 `node --check`）

### 6. 语言包必须按大版本拆分

同一个 `StarUML_Language.json` 覆盖两个大版本会出现：
- 指向已删除文件的死键（4 条）
- 与新源码不符的过期字面量（33 条）
- 覆盖率虚高但实际漏翻

**做法**：`StarUML_Language.v<major>.json` + 启动时读 `package.json` 选包。

## 完整执行链

```text
1. 平台检测 → macOS + 离线样本 case-init
2. asar list / extract → 结构清单 + package.json 版本
3. grep 关键调用点 → 判定 v6 手段是否还有效
4. 覆盖率审计脚本 → 逐条 en 在目标文件中的命中统计
5. 写入合法凭证 → 启动 → 观察凭证存亡 → 定位撤销点
6. 源码级补丁（精确匹配，不匹配就跳过并告警）
7. 替换逻辑重写：精确字面量 + str.replace（不用 re.sub）
8. 打包前 node --check 全量语法门禁
9. 隔离副本（cp -R /tmp + codesign -）实测启动与 UI
10. 真实安装不动，测试后还原用户数据
11. case-review --verify-hashes --strict 清零
```

## 坑记录

| 坑 | 现象 | 解法 |
|---|---|---|
| asar 头部布局 | 按 `u32 + u32 + json` 解析失败 | 实际是 `u32(4) + u32(pickleLen) + u32(jsonLen) + json`；直接 `find(b'{"files"')` 定位更稳 |
| Electron 29 完整性校验 | 担心改 asar 后启动失败 | `Info.plist` 有 `ElectronAsarIntegrity` 但无 `EnableEmbeddedAsarIntegrityValidation`；实测重打包可启动 |
| 签名 | 复制到 /tmp 后无法启动 | `codesign --force --deep --sign - <app>` 走 ad-hoc 签名 |
| `subprocess.call` | `TypeError: unexpected keyword argument 'check'` | `call` 不接受 `check=`，只有 `run` 接受 |
| 中文数字位数 | 309 个 9 → `Infinity` | 13 位即可，避免 UI 渲染成 "Infinity 天" |
| PowerShell 无关 | 本项目纯 Python/Node | macOS 下无需 pwsh |

## 工具链发现

- `npx --yes @electron/asar` 可免全局安装完成 extract/pack/list（无全局 `asar` 时自动回退）
- `node --check` 是最便宜的"替换是否破坏语法"门禁，652 文件 ~25s
- `osascript -e 'tell application "System Events" to ...'` 可读取 macOS 原生菜单栏文本，
  是**验证汉化是否真的生效**的高性价比手段（比截图 OCR 准确）
- 隔离验证模板：`cp -R <App>.app /tmp/x.app` → 换 asar → `codesign --force --deep --sign -` → 启动

## 关键命令

```bash
# 解包 / 打包
npx --yes @electron/asar extract app.asar app
npx --yes @electron/asar pack app app.asar.new

# 判定版本
python3 -c "import json;print(json.load(open('app/package.json'))['version'])"

# 判定 v6 手段是否还有调用点
grep -rn '\$\.post' app/src/ --include=*.js

# 语法门禁
find app -name '*.js' -not -path '*/node_modules/*' | xargs -n1 node --check

# macOS 菜单汉化验证
osascript -e 'tell application "System Events" to tell process "StarUML" to get name of every menu bar item of menu bar 1'

# 隔离启动
cp -R /Applications/StarUML.app /tmp/t.app
cp new.asar /tmp/t.app/Contents/Resources/app.asar
codesign --force --deep --sign - /tmp/t.app && /tmp/t.app/Contents/MacOS/StarUML
```

## 可复用模式

1. **Electron 汉化补丁四原则**
   - 版本自适应（读 `package.json`）
   - 精确字面量替换（带 key 优先）
   - 打包前语法门禁
   - 备份 + 原子写盘 + 可还原

2. **凭证类"激活被还原"问题定位法**
   写凭证 → 启动 → 查存亡 → 搜 `unlink/deactivate` → `curl` 确认触发条件 → 源码级打补丁

3. **隔离验证法**
   永远在 `/tmp` 副本上验证，绝不在用户真实安装上试错；
   测试若触碰用户数据，先备份哈希、测完还原并比对哈希

4. **"覆盖率 100%"要有定义**
   本例定义为：语言包每条 en 字面量在目标文件（按 showKey 规则构造的匹配串）中至少出现一次。
   先写审计脚本，再改数据，最后用同一脚本验收。

5. **补丁的"安全失败"优先级高于"自动成功"**
   精确多行匹配 + 三重守卫：
   - 已含补丁标记 → 跳过（幂等）
   - 片段出现 0 次 → 告警 + 返回 None（不误改）
   - 片段出现 >1 次 → 告警 + 中止（`str.replace` 会全替换，风险不可控）
   这样上游逻辑一变，脚本会明确报"需要人工重新定位"，而不是静默改错。

6. **一次性分析脚本要固化成工具**
   本次审计覆盖率最初是用内联 heredoc 跑的，脚本本身丢了，下次还得重写。
   已固化为 `audit-language-pack.py`（带退出码，可接 CI）。
   **教训：凡是"用来判定任务是否完成"的检查逻辑，都必须落盘成可重跑的工具。**

7. **给未来版本的适配路径要写在项目里，不是只写在经验库里**
   经验库（本文件）是给"做同类任务的人"看的；
   项目内的 `MAINTENANCE.md` 是给"下次升级这个项目的人"看的。
   两者受众不同，都要写。

## 演进动作

- [x] `main.py` 重写为版本自适应 + 安全替换 + 语法门禁 + 原子打包 + 可还原
- [x] 新增 `StarUML_Language.v6.json` / `StarUML_Language.v7.json`，按版本自动选包
- [x] 新增 `patch-license.js`（可单独对已解包目录打补丁）
- [x] `main-en.py` 改为薄封装，消除双实现漂移
- [x] 文档同步：v7 变化说明、激活方式修正、英文文档对齐
- [x] 中英文输出（`SK_LANG=en` / `--lang en`）
- [x] case-review PASS（0 error / 0 warning）
- [x] 新增 `audit-language-pack.py`：把一次性审计逻辑固化为可重跑工具
- [x] 新增 `MAINTENANCE.md`：版本矩阵 + 新大版本适配 7 步流程 + 坑位速查 + 验证清单
- [x] `main.py` 补丁函数加"片段唯一性"守卫（出现 >1 次时中止）

## 未来版本适配路径（已落盘）

本次已把"下次怎么适配"写成可执行文档，而非仅存于记忆：

| 场景 | 入口 |
|---|---|
| 小版本更新（7.1.1→7.1.2） | 直接跑 `main.py`，再用 `audit-language-pack.py` 确认覆盖率仍 100% |
| 新大版本（8.0.0） | `MAINTENANCE.md` 第 2 节，7 步流程 |
| 补丁匹配失败 | `MAINTENANCE.md` 第 4 节，重新定位撤销逻辑 |
| 覆盖率跌破 100% | `MAINTENANCE.md` 第 3 节 + `audit-language-pack.py` |

安全边界（已实测）：
- v8 样例 → `pick_strategy()` 返回 None → 明确报"仅支持 v6.x 与 v7.x"
- v8 结构不同的 license-client → 补丁拒绝并告警，**不误改**

## 遗留

- `hook.js` / `license-manager.js` / `staruml.js` 保留为 v6 兼容资产，文档已标注"仅 v6.x"
- 若 StarUML 后续再次改动 `remoteValidate` 结构，补丁因精确匹配失败会**自动跳过并告警**，
  不会误改；届时需重新定位
