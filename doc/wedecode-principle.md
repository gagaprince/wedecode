# Wedecode 项目原理讲解

本文档基于当前项目源码整理，用来帮助理解 Wedecode 如何把微信小程序包 `wxapkg` 解包并尽量还原成可读、可导入开发者工具的项目。

> 说明：本项目适合用于学习反编译原理、代码安全审计和合法授权场景。不要把它用于未授权的小程序代码获取、复制或传播。

## 1. 项目定位

当前仓库不是某个小程序的反编译产物，而是一个微信小程序/小游戏 `wxapkg` 反编译工具。

它做的事情可以拆成四层：

1. 找到或接收 `wxapkg` 包。
2. 解析微信包格式，把包里的文件按原路径写到输出目录。
3. 识别微信打包后的运行时代码特征，把 `JS`、`JSON`、`WXSS`、`WXML`、`WXS` 等文件重新拆出来。
4. 补齐小程序项目运行需要的默认文件、开发者工具配置，并清理中间产物。

最终目标不是 100% 还原开发者原始工程，而是还原出接近源码形态、便于阅读和在微信开发者工具中打开的项目结构。

## 2. 整体结构

主要目录和文件如下：

```text
src/
  bin/wedecode/                 命令行入口、交互选择、扫描本机小程序缓存
  workspace/                    Web UI 工作区服务
  interface/
    unpack-wxapkg.ts            wxapkg 解包
    base-decompilation.ts       小程序和小游戏共用能力
    app-decompilation.ts        小程序反编译主逻辑
    game-decompilation.ts       小游戏反编译主逻辑
    ployfill-cover.ts           polyfill 覆盖逻辑
  utils/
    create-vm.ts                构造沙箱和假微信/DOM 环境
    decompile-wxml.ts           WXML AST 逆向还原
    get-z.ts                    WXML 表达式池恢复
    get-pack-codeInfo.ts        读取包内关键运行时代码文件
    decrypt-wxapkg.ts           加密包解密
    default-files-generator.ts  补默认页面/组件文件
    project-config.ts           生成开发者工具配置
    file-cleaner.ts             清理中间文件
public/                         Web UI 前端静态文件
decryption-tool/                额外解密工具
```

核心调用链如下：

```text
wedecode CLI / Web UI
  -> DecompilationController.startDecompilerProcess()
    -> UnpackWxapkg.unpackWxapkg()
    -> AppDecompilation.decompileAll()
       或 GameDecompilation.decompileAll()
    -> DefaultFilesGeneratorUtils.generateDefaultAppFiles()
    -> ProjectConfigUtils.generateProjectConfigFiles()
    -> FileCleanerUtils.removeCache()
```

## 3. CLI 和 UI 如何进入反编译流程

命令行入口在 `src/bin/wedecode/wedecode.ts`。

它用 `commander` 定义了两种模式：

- `wedecode`：命令行模式，支持交互式选择、自动扫描、手动指定包路径。
- `wedecode ui`：启动 Web UI 服务。

命令行模式会进入 `src/bin/wedecode/main-commander-process.ts`。这个文件主要负责：

- 判断用户是否已经传入输入路径和输出路径。
- 如果没有输入路径，则通过交互菜单选择扫描方式。
- 调用 `startSacnPackagesProcess()` 扫描微信缓存目录中的 `wxapkg`。
- 根据 `--out`、`--clear`、`--px`、`--unpack-only`、`--wxid` 等参数组装配置。
- 创建 `DecompilationController` 并启动反编译。

Web UI 服务在 `src/workspace/workspace-server.ts`。它提供工作区、文件树、上传、批量反编译、下载和日志推送等接口。UI 反编译时不是在服务进程里直接调用控制器，而是启动子进程执行构建后的 `dist/wedecode.js`，再通过 WebSocket 把输出推给前端。

需要注意：`WorkspaceServer` 还提供了通用命令执行接口 `/api/workspaces/:id/execute`，这适合本机学习和工具使用，不适合作为公网服务暴露。

## 4. DecompilationController 的职责

`src/decompilation-controller.ts` 是整个项目的编排中心。

它做三件事：

1. 判断输入是文件还是目录。
2. 对每个包调用 `UnpackWxapkg.unpackWxapkg()`。
3. 根据包类型选择小程序或小游戏反编译器。

输入是目录时，它会查找目录下的 `.wxapkg` 文件；如果没有 `.wxapkg`，则把目录内普通文件都当成待处理包。多个包会按主包优先排序，主包名来自 `AppMainPackageNames`：

```text
__APP__.wxapkg
app.wxapkg
```

主包优先很关键，因为 `app-config.json`、分包列表、页面入口、全局配置等信息通常都在主包里。先处理主包，后续处理分包时才能更准确地合并路径和配置。

每个包处理完后，控制器会执行收尾工作：

- 尝试获取小程序信息并更新 UI 工作区。
- 如果是 `--unpack-only`，到此结束。
- 生成缺失的默认页面/组件文件。
- 生成 `project.private.config.json`。
- 非开发模式下清理中间运行时代码文件。

## 5. wxapkg 解包原理

解包逻辑在 `src/interface/unpack-wxapkg.ts`。

### 5.1 文件头识别

正常 `wxapkg` 有固定的头部标记：

- 第 0 字节：`0xbe`
- 第 13 字节：`0xed`

项目通过这两个标记判断包是否已经是可解析格式。如果标记不对，会认为包可能是加密包。

### 5.2 加密包解密

解密逻辑在 `src/utils/decrypt-wxapkg.ts`。

项目的判断逻辑很直接：

```text
如果第 0 字节不是 0xbe，或第 13 字节不是 0xed，则认为需要解密。
```

解密需要 `wxid`，来源有两个：

- 用户通过 `--wxid` 指定。
- 从包路径中提取形如 `wx1234567890abcdef` 的 appid。

解密算法分两段：

1. 用 `wxid`、固定 salt `saltiest` 和 PBKDF2 派生 AES-256-CBC 密钥，解密前 1024 字节。
2. 对剩余数据做 XOR 解密，key 取 `wxid` 倒数第二个字符的 charCode。

解密后的数据再交给正常解包流程解析。

### 5.3 文件索引表

`genFileList()` 从包头读取文件数量和索引区长度。每个文件条目包含：

- 文件路径长度
- 文件路径
- 文件内容偏移量
- 文件大小

拿到列表后，项目用 `Buffer.subarray(info.off, info.off + info.size)` 切出每个文件内容，并写入输出目录。

这一步完成后，输出目录里已经有了微信打包后的中间文件，例如：

```text
app-config.json
app-service.js
app-wxss.js
page-frame.js
page-frame.html
workers.js
game.js
subContext.js
```

### 5.4 主包、分包和小游戏判断

解包后会得到一个 `UnPackInfo`：

```ts
{
  appType,
  packType,
  subPackRootPath,
  pathInfo,
  outputPathInfo,
  inputPath,
  outputPath
}
```

判断逻辑：

- `findCommonRoot(fileList.map(item => item.path))` 找公共根路径，作为分包根路径。
- 如果 `app-config.json` 中没有把这个根路径标记为分包，则认为是主包。
- 如果分包配置里 `independent` 为真，则是独立分包。
- 如果输出目录存在 `game.js`，则认为是小游戏，否则是普通小程序。

`pathInfo` 和 `outputPathInfo` 的区别也很重要：

- `pathInfo` 会随着当前包根路径变化，适合写入当前分包文件。
- `outputPathInfo` 永远指向主包输出目录，适合判断整个项目级别的文件。

## 6. 小程序反编译核心

小程序反编译逻辑在 `src/interface/app-decompilation.ts`。

它的整体顺序是：

```text
initApp()
decompileAllJSON()
decompileAppJSON()
decompileAppJS()
decompileAppWXSSWithRpx() 或 decompileAppWXSS()
decompileAppWXML()
decompileAppWXS()
decompileAppWorkers()
```

### 6.1 读取关键运行时代码

`initApp()` 通过 `getAppPackCodeInfo()` 读取几个关键文件：

- `app-config.json`
- `app-wxss.js`
- `app-service.js`
- `appservice.app.js`
- `page-frame.js`
- `page-frame.html`
- `workers.js`

`page-frame.html` 会用 `cheerio` 提取里面的 `<script>` 内容。因为不同微信版本的 WXML/WXSS 运行时代码可能藏在不同文件里，项目会从 `app-wxss`、`page-frame`、`page-frame.html` 中挑一个最像目标运行时代码的候选。

### 6.2 沙箱执行是核心思想

这个项目不是完全靠字符串硬拆源码，而是大量使用“沙箱执行 + 拦截运行时接口”。

沙箱在 `src/utils/create-vm.ts` 中创建，使用：

- `vm2` 提供受控 JS 执行环境。
- `jsdom` 提供假 DOM。
- `createWxFakeDom()` 提供假的 `wx`、`App`、`Page`、`Component`、`Behavior` 等微信 API。

这样做的原因是：微信打包后的代码仍然保留了许多运行时注册点，例如：

- `define(name, factory)` 注册 JS 模块。
- `__wxAppCode__` 保存页面配置或 WXSS 执行函数。
- `setCssToHead()` 把 WXSS 写入页面。
- `$gwx` / `$gwx_xxx` 保存 WXML 编译后的渲染函数。

工具让这些运行时代码在假环境里跑一遍，然后在关键函数上“截获”结果，再把结果保存成源码文件。

### 6.3 JSON 还原

JSON 分两类：

#### app.json

`decompileAppJSON()` 只处理主包。

它从 `app-config.json` 中还原 `app.json`，主要处理：

- 把 `global` 合并到顶层。
- 删除微信运行时内部字段。
- 去掉页面路径里的 `.html`。
- 处理 `renderer`、`skyline`、`componentFramework` 等字段。
- 从 `pages` 中拆分出主包页面和分包页面。
- 还原 `tabBar` 图标路径。
- 如果选择本地化插件，则把远程插件配置清空，改为本地目录。

#### page.json / component.json

`decompileAllJSON()` 会执行 `app-service.js`，拦截 `__wxAppCode__` 中的 `.json` 项。

处理重点：

- 修正 `usingComponents` 的相对路径。
- 把异步分包占位组件 `componentPlaceholder` 替换成 `view`。
- 把插件路径从 `plugin-private://` 转成输出目录中的本地插件路径。
- 把每个页面/组件的 JSON 写回对应位置。

### 6.4 JS 还原

JS 还原在 `decompileAppJS()` 和基类的 `_parseJsDefine()`。

微信打包后的业务 JS 通常以模块形式注册：

```js
define("pages/index/index.js", function(require, module, exports) {
  ...
})
```

项目在沙箱里替换 `define()`：

1. 执行 `app-service.js`。
2. 每遇到一个 `.js` 模块，就拿到模块名和 factory 函数。
3. 从函数体中截取真正代码。
4. 去掉 `"use strict"`、修正部分 `require("@babel")` 路径。
5. 用 `js-beautify` 格式化后保存成 `.js` 文件。

如果包目录旁边有 `polyfill` 目录，或者项目内置 polyfill 能匹配当前模块路径，则会优先使用 polyfill 文件覆盖原始模块。这个能力在 `PloyFillCover` 中实现，常用于替换一些无法可靠还原或需要兼容的模块。

插件 JS 通过 `definePlugin()` 捕获，再把插件内的模块写到本地 `plugin_/<appid>/...` 目录。项目把 `__plugin__` 重命名成 `plugin_`，原因是微信开发者工具不允许某些 `__` 开头目录作为反编译产物直接使用。

### 6.5 WXSS 还原

WXSS 有两种还原方式：

- 默认：`decompileAppWXSSWithRpx()`，优先还原为 `rpx`。
- 使用 `--px`：`decompileAppWXSS()`，从运行后 DOM 中读取样式文本，结果更接近实际渲染的 `px`。

默认方式会修改运行时代码，让 `setCssToHead()` 返回中间结构，再由 `_setCssToHead()` 把微信样式数组转换成文本：

- 类型 `0` 的数组项转成 `rpx`。
- 类型 `2` 的数组项转成 `@import`。
- `body{` 替换为小程序里的 `page{`。
- 最后用 `cssbeautify` 格式化并写入 `.wxss`。

插件样式也会重定向到本地插件目录。

### 6.6 WXML 还原

WXML 是最复杂的部分，涉及三个文件：

- `app-decompilation.ts`
- `utils/get-z.ts`
- `utils/decompile-wxml.ts`

微信不会直接把原始 WXML 放进包里，而是把它编译成 `$gwx` 渲染函数。还原过程大致是：

1. 执行包含 `$gwx` 的运行时代码。
2. 收集所有 WXML 入口函数、模块定义和模板定义。
3. 从 `gz$gwx...` 函数中提取表达式池 `z`。
4. 用 `get-z.ts` 把表达式池里的操作码还原成 `{{...}}`、对象、数组、条件表达式等 WXML 表达式。
5. 用 `decompile-wxml.ts` 解析渲染函数 AST。
6. 识别 `_n`、`_m`、`_v`、`_o`、`_2`、`_ic` 等微信编译辅助函数。
7. 重建标签树、属性、文本节点、`wx:if`、`wx:for`、`template`、`include`、`import` 等结构。
8. 输出 `.wxml` 文件。

可以把它理解成：

```text
WXML 原文
  -> 微信编译器
    -> $gwx 渲染函数 + z 表达式池
      -> Wedecode 逆向分析
        -> WXML 近似源码
```

这种方式强依赖微信编译产物的函数名和结构，所以当微信升级编译器后，最容易坏的通常也是 WXML 还原。

### 6.7 WXS 还原

WXS 还原在 `decompileAppWXS()`。

它同样从 `$gwx` 模块信息中找到 WXML 引用的 WXS 模块，然后：

- 对外部 WXS 文件，调用 `functionToWXS()` 从函数体中提取 WXS 代码并保存 `.wxs`。
- 对内联 WXS，把代码追加回对应 `.wxml`。
- 修正 `require()` 引用路径。

WXS 的命名里会出现 `p_./`、`m_./` 之类前缀，项目通过 `resetWxsRequirePath()` 去掉这些运行时前缀。

### 6.8 Worker 还原

Worker 处理在 `BaseDecompilation.decompileAppWorkers()`。

它执行 `workers.js`，拦截其中的 `define(name, func)`：

- 每个 worker JS 模块通过 `_parseJsDefine()` 写出。
- 同时计算 worker 文件所在的公共目录。
- 把这个目录写回 `app.json` 或 `game.json` 的 `workers` 字段。

## 7. 小游戏反编译

小游戏逻辑在 `src/interface/game-decompilation.ts`，比小程序简单很多。

主要流程：

1. `getGamePackCodeInfo()` 读取 `game.js`、`subContext.js`、`app-config.json`、`workers.js`。
2. `decompileGameJSON()` 从 `app-config.json` 还原 `game.json`，并移除不适合输出的字段。
3. `decompileGameJS()` 执行 `game.js` 和 `subContext.js`，拦截 `define()`，用 `_parseJsDefine()` 还原模块 JS。
4. 复用基类逻辑还原 worker。

小游戏没有 WXML/WXSS 这类页面模板还原流程，所以整体更像“模块拆包 + 配置整理”。

## 8. 默认文件和项目配置

反编译完成后，项目会补齐一些文件，避免导入开发者工具时报缺失文件。

`DefaultFilesGeneratorUtils.generateDefaultAppFiles()` 会：

- 从 `app-config.json` 读取页面列表。
- 递归分析页面 JSON 里的 `usingComponents`。
- 对缺失的页面或组件生成默认 `.json`、`.js`、`.wxml`。

默认内容比较简单：

```js
Page({ data: {} })
```

```xml
<text>页面路径.wxml</text>
```

这些默认文件不是源码还原结果，而是容错占位。README 里提到“好多文件只有默认模板”通常说明缺少分包或组件所在包没有一起反编译。

`ProjectConfigUtils.generateProjectConfigFiles()` 会生成 `project.private.config.json`，默认关闭 ES6 转换和 URL 校验：

```json
{
  "setting": {
    "es6": false,
    "urlCheck": false
  }
}
```

这有助于把反编译结果直接导入开发者工具。

## 9. 缓存清理

`FileCleanerUtils.removeCache()` 会删除反编译过程中用到的中间文件，例如：

- `page-frame.html`
- `app-wxss.js`
- `app-service.js`
- `index.appservice.js`
- `index.webview.js`
- `appservice.app.js`
- `page-frame.js`
- `webview.app.js`
- `common.app.js`
- `subContext.js`
- `worker.js`

这些文件是微信运行时代码，不是最终希望展示给用户的源码结构。开发模式下不会清理，方便调试反编译算法。

## 10. 本机扫描逻辑

扫描逻辑在 `src/bin/wedecode/scan.ts`，平台路径在 `src/bin/wedecode/enum.ts`。

项目内置了不同平台的微信小程序缓存目录：

- macOS：`~/Library/Containers/.../packages` 等。
- Windows：`WeChat Files`、`xwechat/radium/Applet/packages` 等。
- Linux：`~/.config/WeChat/Applet`。

扫描时会找 `__APP__.wxapkg` 或 `app.wxapkg`，再尝试从路径中识别 appid，并通过远程接口查询小程序名称、描述等信息，供交互表格展示。

这部分不是反编译算法核心，只是为了更方便地找到微信本地缓存里的包。

## 11. Web UI 工作区服务

`WorkspaceServer` 是 UI 包装层，主要能力包括：

- 创建、删除、读取工作区。
- 上传多个 `.wxapkg` 到工作区。
- 对工作区上传目录批量反编译。
- 读取、保存、新建、删除工作区文件。
- WebSocket 推送执行日志。
- 下载反编译结果压缩包。
- 更新工作区中的小程序信息。

UI 的批量反编译命令大致是：

```text
node dist/wedecode.js <workspace/uploads> --out <workspace/OUTPUT> --clear ...
```

也就是说，UI 和 CLI 最终复用同一套 `DecompilationController`。

## 12. 关键技术点总结

### 12.1 利用包格式恢复文件

`wxapkg` 包里有文件索引表，包含路径、偏移和大小。只要包没有加密，或能正确解密，就能把包内文件原样释放出来。

### 12.2 利用运行时注册点恢复源码

微信打包产物不是完全不可读，它保留了很多注册函数。Wedecode 的关键思路是执行这些注册代码，并替换注册函数：

```text
原始运行时 define()
  -> 注册模块给微信运行时

Wedecode 替换后的 define()
  -> 拿到模块名和函数体
  -> 格式化
  -> 保存成源码文件
```

WXSS、WXML、WXS 也是类似思路，只是拦截点分别变成 `setCssToHead()`、`$gwx` 和 WXS 模块映射。

### 12.3 用假微信环境降低执行失败率

反编译过程中会执行包内 JS。如果没有 `wx`、`App`、`Page`、`document` 等对象，代码很容易报错。项目用 `createWxFakeDom()` 和 `jsdom` 提供假环境，让代码尽量“跑到注册阶段”。

这里的目标不是模拟完整微信，而是尽量不让包内代码因为缺 API 而中断。

### 12.4 WXML 还原是结构逆向

JS 和 WXSS 更多是拦截注册结果；WXML 则需要分析编译后的渲染函数和表达式池。项目通过 `esprima` 解析 AST，再按微信编译辅助函数的语义重新构建标签树。

这是整个项目最依赖微信内部实现的地方。

## 13. 局限性

1. 反编译结果不等于原始源码。

   变量名、注释、工程组织、构建前源码结构通常无法完全恢复。

2. WXML/WXSS 对微信版本敏感。

   一旦微信改变 `$gwx`、`setCssToHead`、表达式池或运行时代码格式，相关还原逻辑就需要适配。

3. 缺少分包会产生默认模板。

   如果只反编译主包，而页面或组件实际在分包中，工具会生成占位文件保证项目结构完整。

4. 沙箱执行不是业务运行。

   工具只希望执行注册逻辑，不保证小程序业务代码真的能完整运行。

5. UI 服务不适合公网暴露。

   工作区服务具备文件读写、压缩下载和命令执行能力，应只在可信本地环境使用。

## 14. 推荐阅读顺序

如果想继续深入源码，建议按这个顺序读：

1. `src/decompilation-controller.ts`

   先理解总流程：输入、解包、选择反编译器、收尾。

2. `src/interface/unpack-wxapkg.ts`

   理解 `wxapkg` 文件头、索引表、分包路径和包类型判断。

3. `src/utils/create-vm.ts` 和 `src/utils/wx-dom.ts`

   理解沙箱和假微信环境。

4. `src/interface/base-decompilation.ts`

   理解 JS 模块函数如何被拆出、格式化和写盘。

5. `src/interface/app-decompilation.ts`

   重点看 JSON、JS、WXSS、WXML、WXS 的还原顺序。

6. `src/utils/get-z.ts` 和 `src/utils/decompile-wxml.ts`

   深入 WXML 表达式和标签树还原。

7. `src/interface/game-decompilation.ts`

   对比小游戏和小程序还原流程差异。

8. `src/workspace/workspace-server.ts`

   理解 Web UI 如何包装同一套能力。

## 15. 一句话理解这个项目

Wedecode 的本质是：

```text
先按 wxapkg 文件索引把包拆开，
再在受控沙箱里执行微信打包后的运行时代码，
通过拦截 define、__wxAppCode__、setCssToHead、$gwx 等注册点，
把微信运行时需要的中间结构重新落盘为 JS、JSON、WXSS、WXML、WXS 和项目配置。
```

它不是“魔法还原源码”，而是充分利用了微信小程序打包产物中仍然保留的元信息和运行时注册结构。
