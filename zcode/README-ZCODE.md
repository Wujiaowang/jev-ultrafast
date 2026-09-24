# ZCode / IAB 移植层

上游 jev-ultrafast 通过 browser-harness(CDP) 驱动 Chrome。本目录把它移植到
ZCode 原生生态：**brain 原样保留（snapshot.js / 提示词 / choice 校验），浏览器层
换成 ZCode IAB**，循环在一个 node_repl 单元内闭环，页面快照不经过对话上下文。

## 组成

- `jev-loop.mjs` — 全部逻辑：上游 `snapshot.js` verbatim + `questions.py` 提示词
  verbatim + `model.py` 的 choice 校验/动作空间移植 + IAB 适配（evaluate 观察、
  页内事件执行）+ 双重门控的 `runTask`。
- `.env`（仓库根，gitignored）— TYPESAFE_API_KEY / TEXT_MODEL_*（指向本机
  NewAPI 网关）。

## 架构

```
调度终审（ZCode 会话）──任务卡片──▶ runTask(tab, {goal, ...})
                                       │ 观察: evaluate(SNAPSHOT_JS)
                                       │ 决策: POST api.typesafe.ai (Jev)
                                       │ 填字: TEXT_MODEL → NewAPI /v1
                                       │ 执行: 页内事件 / dom_cua.scroll
                                       ▼
                              {status, evidence, action_log}
门1: 任一置信度 < 0.65 → status=escalated 打回调度者
门2: DONE/BLOCKED 一律由调度者审 evidence（final_url/title/text）后交付
```

## 与上游的差异

| 上游 | 本移植 |
|---|---|
| browser-harness + CDP 真实输入 | IAB `playwright.evaluate` 页内事件（`el.click()` / 原生 setter 填值） |
| 60 步预算 | 默认 40 步（会话内更保守） |
| 无置信度门 | 门 1：`confidenceThreshold`（默认 0.65） |
| 后台标签 focus 模拟 | 无（IAB 前台标签） |
| combobox after-input 稳定等待 | 简化为固定 400ms |

已知限制：页内合成事件 `isTrusted=false`，极少数反自动化站点不认；无 shadow
DOM/iframe 支持（同上游）；百度首页元素的 label 抽取会混入热搜文本（不影响
执行，node id 驱动而非 label）。

## 致谢

决策层全部来自 [browser-use/jev-ultrafast](https://github.com/browser-use/jev-ultrafast)
（MIT）。与上游合并时：`SNAPSHOT_JS` 与三段提示词保持 verbatim，只动适配层。
