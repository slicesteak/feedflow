# 我的 FeedFlow 项目笔记

> 个人学习/维护笔记，非原作者文档。这个文件是我自己加的，原作者不会改它，
> 所以同步上游更新时不会冲突。官方文档看 `README.md` 和 `AGENTS.md`。

## 1. 这个项目是什么

FeedFlow = **多源信息流聚合桌面应用**。把微博 / X / V2EX / GitHub Trending /
Hacker News / Product Hunt 等多个信息源，聚合到一条统一的时间线里，不用挨个
App/网站去刷。

- 原作者：joyme123（https://github.com/joyme123/feedflow）
- 我的 fork：https://github.com/slicesteak/feedflow
- 技术栈：Electron（桌面端） + React 18 + TypeScript + SQLite（本地存储）
  + zustand（状态管理），构建工具是 electron-vite。

## 2. 一个仓库里其实有「两个程序」

容易搞混的点：这不是一个程序，是一个仓库放了两个独立程序（monorepo）。

| 位置 | 是什么 | 跑在哪 |
|---|---|---|
| `src/` | FeedFlow 桌面应用（主角） | 我的电脑桌面（Electron 窗口） |
| `extensions/cookie-sync/` | Chrome 扩展（帮手） | Chrome 浏览器里 |

**为什么桌面 App 还要个 Chrome 扩展？** 为了拿登录 cookie。
刷微博/X 需要登录状态（cookie），而 cookie 存在浏览器里，桌面 App 没权限直接
读浏览器 cookie。所以需要一个跑在 Chrome 里的扩展当「内应」：在浏览器里读到
cookie → 通过 localhost 递给桌面 App → 桌面 App 拿着 cookie 去抓内容。
- 只有 cookie 登录的源（微博、X）才需要这个扩展。
- 公开源（GitHub Trending、Hacker News）桌面 App 直接抓，用不到扩展。

## 3. 为什么我运行要用 `npm run dev`，而 Release 里是 .dmg / .deb

两种东西，服务两类人：

| | `npm run dev` | Release 的 .dmg / .deb |
|---|---|---|
| 给谁 | 开发者（改代码的我） | 普通用户（装来就用） |
| 是什么 | 从源码实时编译运行、带热更新 | 打包好的成品安装包 |
| 前提 | 源码 + Node + `npm install` | 双击即装，啥都不用 |

比喻：.dmg 是装修好的成品房（拎包入住）；`npm run dev` 是拿图纸在工地边改边看。
我要改代码，所以必须用 dev 模式。想把自己改的版本也做成 .dmg，跑打包命令即可。

**为什么源码需要编译？** 因为用了 TypeScript(.ts/.tsx) 和 React JSX，浏览器/Node
不能直接跑，必须编译成普通 JS。electron-vite 把 `src/` 编译到 `out/`
（`package.json` 里 `"main": "./out/main/index.js"` 就是加载编译产物）。
注意：改插件（`plugins/` 里的纯 .js）不用编译；改主程序源码要重新 build。

## 4. GitHub 上的 "Deployments" 是什么

跟运行 App 无关。是原作者的自动发布流水线（GitHub Actions，见
`.github/workflows/release.yml` 里 `publish-extension` 任务，
`environment: chrome-web-store`）：每次发版自动把 Chrome 扩展发布到 Chrome 应用
商店，GitHub 把这种发布动作记录为一次 Deployment。
- 是 GitHub 帮作者跑的，不在我或用户电脑上跑。
- 我的 fork 里没有发布密钥，触发也会自动跳过，不影响我。

## 5. 常用命令小抄

| 想干嘛 | 命令 |
|---|---|
| 装依赖（第一次/依赖变了） | `npm install` |
| 启动开发（日常） | `npm run dev` |
| 停止运行 | 终端里按 `Ctrl + C` |
| 打包成自己的 .dmg | `npm run package:mac` |

**踩过的坑：** 若报 `Error: Electron uninstall`（找不到 Electron 二进制），
是 npm install 时那个上百 MB 的 Electron 运行时没下全。补救不用重装，跑：
`node node_modules/electron/install.js` 即可。

## 6. 我的 fork 与上游（原作者）的 git 关系

本地目录已初始化成 git 仓库，接了两个远端：

```
origin    → https://github.com/slicesteak/feedflow.git   （我的 fork，往这里推）
upstream  → https://github.com/joyme123/feedflow.git      （原作者，从这里拉修复）
```

**A. 提交我的个性化修改并推到我的 fork：**
```bash
git add .
git commit -m "描述改了啥"
git push
```

**B. 同步原作者的 bug 修复：**
```bash
git fetch upstream
git merge upstream/main
git push
```

只有我和作者改了同一处代码时才会冲突，到时手动解决。

**第一次 push 前注意：** 推送要 GitHub 身份验证（不能用账号密码），需配
Personal Access Token 或 SSH key。commit 署名邮箱想算到 GitHub 账号名下，
可 `git config user.email "我的GitHub邮箱"`。

## 7. 我的目标 & 扩展思路

目标：用它当个人信息中枢，加一些自己想要的信息源。
- **加博客/RSS 源**：非常契合。信息源都是插件（`plugins/` 下一个文件夹 + 一个
  纯 .js，实现 `fetchItems()` 返回统一的 item 结构即可），不用改主程序、不用编译。
- **加视频节目（需先抓取再转文本）**：输出（转好的文字）能装进 item 模型，但
  "下载+语音转文字" 很慢、要外部工具，不该塞进 `fetchItems`（有超时、无后台队列）。
  正确做法：转写放到插件外部预处理，插件只读已转好的结果，保持快进快出。
- 插件跑在主进程、有完整 Node 权限（自用方便，装别人的插件要留心）。
- 项目还内置 MCP server，聚合进来的内容能被本地 AI agent 搜索/问答。
