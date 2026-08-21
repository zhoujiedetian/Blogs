# DeepseekHarness源码阅读（二）
#### Profile：一个可启动的配置栈
dsh 用 profile 表示"一种可启动的形态"。官方内置两个，其余用插件创建：
| Profile | 用途 | 命令 |
|---|---|---|
| `web` | Web UI（对话、侧边栏、工具） | `dsh web` |
| `headless` | 一次性 CLI 任务 | `dsh --profile headless "任务"` |
| `tui`（需插件） | 终端 UI | `dsh --profile tui`（未内置，需先安装插件） |
一个 profile 目录长这样（~/.dsh/profiles/<*name*>/）：
```bash
profiles/web/
├── package.json        # 插件依赖 + dsh.profile 清单（bundles 顺序）
├── cordis.patch.yml    # 你的补丁层：挂载/覆盖插件的声明
├── cordis.yml          # （生成的）合成配置
├── pnpm-workspace.yaml
└── node_modules/
```
**加载流程：** 启动 DSH 时，首先定位目标 profile 并读取 `profile/package.json`。其中 `dependencies` 让 Node 能解析到所需的插件包；`dsh.profile.bundles` 则声明内置 bundle 的加载顺序。随后 DSH 按顺序合并各 bundle 自带的 patch，再叠加 profile、用户和命令行的补丁层：

```text
1. profile/package.json
   ├── dependencies：声明/链接当前 profile 可用的插件包
   └── dsh.profile.bundles：声明内置 bundle 顺序
       （例如：dsh-base → dsh-web-app）

2. 内置 bundle 的 cordis.patch.yml
   └── 挂载 DSH 官方默认插件

3. profile 的 cordis.patch.yml
   └── 为当前 profile 添加、覆盖或禁用插件
       （例如手动挂载 dsh-better-sidebar）

4. 用户级 ~/.dsh/cordis.patch.yml
   └── 对该用户启动的 profile 追加全局定制

5. --patch 覆盖层
   └── 仅对本次命令生效的临时覆盖

6. 合成最终插件树并启动 DSH
```
#### 内置 Agent 预设：标准 / PTC / 极简 / 创造
| 预设 | 定位 | 加载内容 | 典型场景 |
|---|---|---|---|
| **标准** | 默认预设 | 完整工具组合 | 日常 Agent 工作 |
| **PTC**（Code mode） | 代码驱动工具链 | 程序化工具调用：模型生成一段代码，组合多轮工具调用 | 复杂、多步骤工具工作流 |
| **极简** | 基准测试 | 仅 `bash` 加一个文件编辑工具 | 最小环境下的模型基准测试 |
| **创造** | 插件开发用 | 检查当前运行时、在内存中试验 Cordis 插件、组合新预设 | 构建与测试新的插件组合 |


#### 挂载一个插件：两处改动
- package.json 加依赖（link: 指向本地源码，或用 npm 包名）：
```bash
"dependencies": {
   "dsh-better-sidebar": "link:/Users/zhoujie/Desktop/FortuneBuilding/DeepseekHarness/DSH-better-sidebar"
  }
```
- cordis.patch.yml 加挂载行：
```bash
- insert:
    - id: better-sidebar
      name: dsh-better-sidebar
```
- 安装并重启：
```bash
cd ~/.dsh/profiles/web && pnpm install
dsh web   # 重启后生效
```

#### host 半与 client 半：一个包，两副面孔
| 半边 | 运行位置 | 职责 | 示例 |
|---|---|---|---|
| **host 半** | Node 进程 | 工具、服务、事件、文件系统、进程 | `apply(ctx)` 注册工具或服务 |
| **client 半** | 浏览器（`web` profile） | UI、交互、DOM | `package.json` 的 `dsh.client` 声明，加上 `src/client/` 中的客户端代码 |

package.json 里如何声明 client 半：
```
{
  "dsh": {
    "client": {
      "platform": "web",
      "inject": [],        // 需要注入的 client 服务（通常为空；官方示例即 []）
      "immediately": true  // 是否随 web 启动立即加载
    }
  },
  "exports": {
    ".": { "types": "./lib/types/index.d.ts", "default": "./lib/index.js" },
    "./client": { "types": "./lib/types/client/index.d.ts", "default": "./lib/client.js" }
  }
}
```


