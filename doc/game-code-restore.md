# 小游戏代码还原逻辑详解

本文档专门讲解当前项目中“微信小游戏”方向的代码还原流程。

相关源码：

- `src/decompilation-controller.ts`
- `src/interface/unpack-wxapkg.ts`
- `src/interface/game-decompilation.ts`
- `src/interface/base-decompilation.ts`
- `src/utils/get-pack-codeInfo.ts`
- `src/utils/create-vm.ts`
- `src/utils/wx-dom.ts`

## 1. 小游戏还原和小程序还原有什么不同

普通小程序通常包含：

```text
JS
JSON
WXML
WXSS
WXS
workers
```

所以普通小程序反编译要处理页面模板、样式、组件配置等复杂结构。

小游戏则简单很多，主要关注：

```text
game.js
subContext.js
workers.js
app-config.json
```

小游戏没有普通页面的 WXML/WXSS 体系，因此不需要走 `$gwx` 逆向 WXML、`setCssToHead` 还原 WXSS 这套复杂流程。

小游戏代码还原的核心是：

```text
执行打包后的 game.js / subContext.js / workers.js
  -> 拦截 define(name, factory)
  -> 从 factory 函数体中抽出真实模块代码
  -> 格式化
  -> 按模块名写回独立 .js 文件
```

一句话说，小游戏还原主要是“模块拆包”。

## 2. 整体调用链

小游戏的完整还原链路如下：

```text
DecompilationController.startDecompilerProcess()
  -> singlePackMode()
    -> UnpackWxapkg.unpackWxapkg()
      -> 解包并判断 appType
    -> 如果 appType === "game"
      -> new GameDecompilation(packInfo)
      -> decompileAll()
        -> initGame()
        -> decompileGameJSON()
        -> decompileGameJS()
        -> decompileAppWorkers()
```

控制器入口在 `src/decompilation-controller.ts`：

```ts
private async singlePackMode(wxapkgPath: string, outputPath: string): Promise<void> {
  const packInfo = await UnpackWxapkg.unpackWxapkg(wxapkgPath, outputPath, this.config.wxid)

  if (this.config.unpackOnly) return
  if (packInfo.appType === 'game') {
    // 小游戏
    const decompilationGame = new GameDecompilation(packInfo)
    await decompilationGame.decompileAll()
  } else {
    // 小程序
    const decompilationApp = new AppDecompilation(packInfo)
    ...
  }
}
```

这里的关键判断是：

```ts
if (packInfo.appType === 'game')
```

`appType` 来自解包阶段。

## 3. 第一步：识别这是小游戏

小游戏识别逻辑在 `src/interface/unpack-wxapkg.ts`：

```ts
let appType: MiniAppType = 'app'
...
if (fs.existsSync(outputPathInfo.gameJsPath)) {
  appType = 'game'
}
```

`outputPathInfo.gameJsPath` 最终指向输出目录里的：

```text
game.js
```

含义：

- 解包后，如果输出目录中存在 `game.js`，项目就认为这是小游戏包。
- 否则默认按普通小程序处理。

为什么看 `game.js`？

因为小游戏的入口文件就是 `game.js`。普通小程序的核心运行时代码一般是 `app-service.js`、`page-frame.js`、`app-wxss.js` 等，不以 `game.js` 作为主入口。

## 4. 第二步：创建 GameDecompilation

当识别为小游戏后，控制器会创建 `GameDecompilation`：

```ts
const decompilationGame = new GameDecompilation(packInfo)
await decompilationGame.decompileAll()
```

`GameDecompilation` 继承自 `BaseDecompilation`：

```ts
export class GameDecompilation extends BaseDecompilation {
  private codeInfo: GameCodeInfo
  ...
}
```

基类提供了这些共用能力：

- 保存当前包路径和输出路径。
- 保存包类型和应用类型。
- 支持 polyfill 覆盖。
- 拆出 `define()` 注册的 JS 模块。
- 还原 worker 模块。

小游戏自己的类只保留和小游戏相关的流程：读取关键文件、生成 `game.json`、还原 JS。

## 5. 第三步：读取小游戏关键文件

初始化逻辑在 `src/interface/game-decompilation.ts`：

```ts
private async initGame() {
  this.codeInfo = getGamePackCodeInfo(this.pathInfo)
  const loadInfo = {}
  for (const name in this.codeInfo) {
    loadInfo[name] = this.codeInfo[name].length
  }
  console.log(loadInfo)
}
```

真正读取文件的是 `src/utils/get-pack-codeInfo.ts`：

```ts
export function getGamePackCodeInfo(
  pathInfo: PathResolveInfo,
  opt: { adaptLen?: number } = {}
): GameCodeInfo {
  const {adaptLen = 100} = opt || {}
  
  function __readFile(path: string) {
    if (!path) return ''
    const content = readLocalFile(path)
    return content.length > adaptLen ? content : ''
  }

  return {
    workers: __readFile(pathInfo.workersPath),
    gameJs: __readFile(pathInfo.gameJsPath),
    appConfigJson: __readFile(pathInfo.appConfigJsonPath),
    subContextJs: __readFile(pathInfo.subContextJsPath),
  }
}
```

这里读取四类文件：

| 字段 | 对应文件 | 作用 |
| --- | --- | --- |
| `gameJs` | `game.js` | 小游戏主包打包代码，最核心 |
| `subContextJs` | `subContext.js` | 开放数据域代码，比如排行榜、好友数据 |
| `workers` | `workers.js` | worker 线程打包代码 |
| `appConfigJson` | `app-config.json` | 微信运行时配置，后续整理成 `game.json` |

`adaptLen = 100` 的意思是：太短的文件会被当作无效内容忽略。

## 6. 第四步：还原 game.json

`game.json` 还原逻辑在 `src/interface/game-decompilation.ts`：

```ts
private async decompileGameJSON() {
  if (this.packType !== 'main') return
  await sleep(200)
  const gameConfigString = this.codeInfo.appConfigJson
  const gameConfig: Record<any, any> = JSON.parse(gameConfigString)
  Object.assign(gameConfig, gameConfig.global)
  GameJsonExcludeKeys.forEach(key => delete gameConfig[key])

  const outputFileName = 'game.json'
  const gameConfigSaveString = JSON.stringify(gameConfig, null, 2)
  saveLocalFile(this.pathInfo.outputResolve(outputFileName), gameConfigSaveString, {force: true})
  ...
}
```

这段逻辑分几步：

### 6.1 只处理主包

```ts
if (this.packType !== 'main') return
```

`game.json` 是项目级配置，只应该由主包生成。分包不需要重复生成。

### 6.2 读取 app-config.json

```ts
const gameConfigString = this.codeInfo.appConfigJson
const gameConfig: Record<any, any> = JSON.parse(gameConfigString)
```

解包后拿到的通常不是最终可用的 `game.json`，而是微信运行时使用的 `app-config.json`。这里先把它解析成对象。

### 6.3 合并 global 配置

```ts
Object.assign(gameConfig, gameConfig.global)
```

微信运行时配置里，很多全局配置会放在 `global` 字段下。最终输出 `game.json` 时，项目把 `global` 里的字段合并到顶层。

注意：这里没有删除 `global` 字段本身，只是把里面的内容拷贝到顶层。普通小程序的 `decompileAppJSON()` 会显式 `delete appConfig.global`，小游戏这里没有做这一步。

### 6.4 删除不适合输出的字段

```ts
GameJsonExcludeKeys.forEach(key => delete gameConfig[key])
```

排除字段定义在 `src/constant/index.ts`：

```ts
export const GameJsonExcludeKeys = [
  'openDataContext',
]
```

也就是说，小游戏还原 `game.json` 时会删除 `openDataContext`。

### 6.5 写出 game.json

```ts
const outputFileName = 'game.json'
const gameConfigSaveString = JSON.stringify(gameConfig, null, 2)
saveLocalFile(this.pathInfo.outputResolve(outputFileName), gameConfigSaveString, {force: true})
```

最终把整理后的配置写入输出目录根部：

```text
game.json
```

## 7. 第五步：还原 game.js 和 subContext.js 中的模块

小游戏 JS 还原逻辑在 `decompileGameJS()`：

```ts
private async decompileGameJS() {
  const _this = this
  let cont = 0
  const evalCodeList = [
    this.codeInfo.subContextJs,
    this.codeInfo.gameJs
  ].filter(Boolean)
  const allJsList = []
  const sandbox = {
    define(name: string, func: Function) {
      allJsList.push(name)
      _this._parseJsDefine(name, func)
      cont++
    },
    require() {
    },
  }
  evalCodeList.forEach(code => {
    const vm = createVM({sandbox})
    if (!code.includes('define(') || !code.includes('function(require, module, exports)')) return
    try {
      runVmCode(vm, code)
    } catch (e) {
      console.error(e.message)
    }
  })
  if (cont) {
    printLog(` ▶ 反编译所有 js 文件成功. \n`)
  }
}
```

这是小游戏还原中最核心的一段。

## 8. 打包后的 game.js 长什么样

微信小游戏打包后的 `game.js` 往往是一个大文件，里面用模块注册函数组织原始源码。

形态大致类似：

```js
define("js/main.js", function(require, module, exports) {
  "use strict";
  const player = require("./player");
  GameGlobal.main = ...
});

define("js/player.js", function(require, module, exports) {
  "use strict";
  class Player {}
  module.exports = Player;
});
```

真实文件可能被压缩、混淆或包裹得更复杂，但核心点是：

```text
模块名在 define 的第一个参数里
模块代码在 define 的第二个参数函数体里
```

所以只要执行这段打包产物，并把 `define()` 换成自己的函数，就能拿到每个模块。

## 9. 关键技术：拦截 define()

项目构造了一个沙箱对象：

```ts
const sandbox = {
  define(name: string, func: Function) {
    allJsList.push(name)
    _this._parseJsDefine(name, func)
    cont++
  },
  require() {
  },
}
```

这里的 `define()` 不是微信原来的 `define()`，而是项目自己提供的“钩子函数”。

当打包代码执行：

```js
define("js/main.js", function(require, module, exports) {
  ...
})
```

实际进入的是项目的函数：

```ts
define(name, func) {
  _this._parseJsDefine(name, func)
}
```

于是项目得到了：

- `name`：模块路径，比如 `js/main.js`
- `func`：模块函数，里面包着原始模块代码

这就是小游戏 JS 还原的关键。

## 10. 为什么要在沙箱里执行

执行代码用的是：

```ts
const vm = createVM({sandbox})
runVmCode(vm, code)
```

`createVM()` 在 `src/utils/create-vm.ts`，内部使用：

- `vm2`
- `jsdom`
- 假 `wx` API
- 假 `App`、`Page`、`Component`
- 假 `window`、`document`
- 假 `require`

目的不是运行小游戏业务，而是让打包代码能顺利跑到 `define()` 注册阶段。

如果没有沙箱和假环境，打包代码里一旦访问 `wx`、`window`、`document`、`GameGlobal` 等对象，就可能直接报错，导致模块注册无法完成。

当前项目的假环境主要在 `src/utils/create-vm.ts` 和 `src/utils/wx-dom.ts`。

## 11. 执行前的格式保护

代码执行前有一个判断：

```ts
if (!code.includes('define(') || !code.includes('function(require, module, exports)')) return
```

含义：

- 如果文件里没有 `define(`，说明它不像模块打包产物。
- 如果没有 `function(require, module, exports)`，说明它不像标准 CommonJS 模块包装。
- 这种情况下直接跳过，避免误执行无关代码。

这是一个简单但实用的保护。

## 12. 第六步：从 define 函数体里抽出真实 JS

真正拆模块的逻辑在基类 `src/interface/base-decompilation.ts`：

```ts
protected _parseJsDefine(name: string, func: Function) {
  if (path.extname(name) !== '.js') return
  ...
}
```

### 12.1 只处理 .js 模块

```ts
if (path.extname(name) !== '.js') return
```

如果 `define()` 注册的不是 `.js` 文件，就不处理。

### 12.2 polyfill 覆盖

```ts
const foundPloyfill = this.ployFill.findPloyfill(name)
let resultCode: string = ''
if (foundPloyfill) {
  resultCode = readLocalFile(foundPloyfill.fullPath)
} else {
  ...
}
```

如果用户在包同级目录提供了 `polyfill`，或者项目内置 polyfill 能匹配当前模块路径，就使用 polyfill 的代码。

这个机制可以用来替换一些难以还原、还原后不可运行，或者需要兼容的模块。

### 12.3 取出函数体

```ts
let code = func.toString();
code = code.slice(code.indexOf("{") + 1, code.lastIndexOf("}") - 1).trim();
```

`func` 原本类似：

```js
function(require, module, exports) {
  "use strict";
  const x = 1;
  module.exports = x;
}
```

`func.toString()` 后还是完整函数文本。项目通过字符串截取，去掉最外层函数壳，只留下：

```js
"use strict";
const x = 1;
module.exports = x;
```

### 12.4 去掉 use strict

```ts
if (code.startsWith('"use strict";')) {
  code = code.replaceAll('"use strict";', '')
} else if (code.startsWith("'use strict';")) {
  code = code.replaceAll(`'use strict';`, '')
} else if ((code.startsWith('(function(){"use strict";') || code.startsWith("(function(){'use strict';")) && code.endsWith("})();")) {
  code = code.slice(25, -5);
}
```

微信打包后的模块通常带 `"use strict"`。项目会把它移除，让输出结果更简洁。

第三个分支处理的是这种自执行函数包裹：

```js
(function(){"use strict";
  ...
})();
```

### 12.5 修正 @babel require 路径

```ts
code = code.replaceAll('require("@babel', 'require("./@babel')
```

有些打包代码会引用 `@babel` helper。输出到本地目录后，直接 `require("@babel...")` 可能会被当成 npm 包路径。

项目把它改成：

```js
require("./@babel...")
```

这样可以指向输出目录里的本地 helper 文件。

### 12.6 格式化代码

```ts
resultCode = jsBeautify(code.trim());
```

`jsBeautify()` 使用 `js-beautify`：

```ts
export function jsBeautify(code: string) {
  return JS.js_beautify(code, {indent_size: 2})
}
```

这一步只负责排版，不会恢复变量名、注释或 TypeScript 源码。

### 12.7 移除 vm2 异常包装痕迹

```ts
removeVM2ExceptionLine(resultCode.trim())
```

对应工具函数：

```ts
export function removeVM2ExceptionLine(code: string) {
  const reg = /\s*[a-z]\x20?=\x20?VM2_INTERNAL_STATE_DO_NOT_USE_OR_PROGRAM_WILL_FAIL\.handleException\([a-z]\);?/g
  return code.replace(reg, '')
}
```

`vm2` 执行某些代码时可能插入内部异常处理片段。这个函数会把这些噪声去掉。

### 12.8 写回文件

```ts
saveLocalFile(
  this.pathInfo.outputResolve(name),
  removeVM2ExceptionLine(resultCode.trim()),
  {force: true}
)
```

如果模块名是：

```text
js/main.js
```

那最终会写到输出目录：

```text
OUTPUT/js/main.js
```

如果是分包，`pathInfo` 会结合分包根路径处理输出位置。

## 13. 第七步：还原 worker 代码

小游戏还原最后会执行：

```ts
await this.decompileAppWorkers()
```

这个方法来自 `BaseDecompilation`：

```ts
protected async decompileAppWorkers(): Promise<any> {
  await sleep(200)
  if (!fs.existsSync(this.pathInfo.workersPath)) {
    return
  }
  const _this = this
  let commPath: string = '';
  let code = readLocalFile(this.pathInfo.workersPath)
  let vm = createVM({
    sandbox: {
      define(name: string, func: Function) {
        _this._parseJsDefine(name, func)
        const workerPath = path.dirname(name) + '/';
        if (!commPath) commPath = workerPath;
        commPath = commonDir(commPath, workerPath);
      }
    }
  })
  runVmCode(vm, code)
  ...
}
```

逻辑和 `game.js` 类似：

1. 读取 `workers.js`。
2. 在沙箱中执行。
3. 拦截 `define(name, func)`。
4. 用 `_parseJsDefine()` 拆出 worker 模块。
5. 计算 worker 模块共同目录。

然后把 worker 目录写回配置文件：

```ts
if (commPath) {
  const configFileName = this.appType === 'game'
    ? this.pathInfo.gameJsonPath
    : this.pathInfo.appJsonPath
  const appConfig: Record<any, any> = JSON.parse(readLocalFile(configFileName))
  appConfig.workers = commPath
  saveLocalFile(configFileName, JSON.stringify(appConfig, null, 2), {force: true})
}
```

因为当前是小游戏：

```ts
this.appType === 'game'
```

所以会更新：

```text
game.json
```

写入类似：

```json
{
  "workers": "workers/"
}
```

具体值取决于 worker 模块路径的公共目录。

## 14. decompileAll 的完整顺序

小游戏最终流程在 `GameDecompilation.decompileAll()`：

```ts
public async decompileAll() {
  super.decompileAll()
  /* 开始编译 */
  await this.initGame()
  await this.decompileGameJSON()
  await this.decompileGameJS()
  await this.decompileAppWorkers()
}
```

逐步解释：

### 14.1 super.decompileAll()

打印当前目标信息：

```text
当前反编译目标[ 小游戏 ] (主包/分包/独立分包)
当前输出目录
```

不做实际还原。

### 14.2 initGame()

读取关键文件：

```text
game.js
subContext.js
app-config.json
workers.js
```

### 14.3 decompileGameJSON()

只在主包处理：

```text
app-config.json -> game.json
```

### 14.4 decompileGameJS()

处理：

```text
subContext.js
game.js
```

通过拦截 `define()` 拆出 JS 模块。

### 14.5 decompileAppWorkers()

处理：

```text
workers.js -> worker 模块文件
```

并把 worker 公共目录写回 `game.json`。

## 15. 还原前后结构示意

解包后可能是：

```text
OUTPUT/
  app-config.json
  game.js
  subContext.js
  workers.js
  其他资源文件...
```

还原后可能变成：

```text
OUTPUT/
  game.json
  js/
    main.js
    player.js
    scene.js
  openDataContext/
    index.js
  workers/
    worker.js
  images/
  audio/
  其他资源文件...
```

具体目录取决于 `define()` 里注册的模块名。

## 16. 小游戏还原的本质

小游戏打包代码可以理解为一个模块表：

```js
define("a.js", function(require, module, exports) {
  // a.js 原始内容
});

define("b.js", function(require, module, exports) {
  // b.js 原始内容
});
```

项目做的事情是把 `define()` 替换成：

```js
function define(name, func) {
  save(name, extractFunctionBody(func))
}
```

因此小游戏代码还原的本质是：

```text
执行模块注册代码，而不是执行业务逻辑；
截获模块名和模块函数；
把模块函数体重新保存为独立 JS 文件。
```

## 17. 这种还原方式的局限

### 17.1 不能恢复原始工程全部信息

可以还原模块内容和路径，但通常不能恢复：

- 原始注释。
- 原始变量名。
- TypeScript 源码。
- 构建前目录结构。
- webpack/rollup/vite 原始配置。

如果打包前已经压缩混淆，输出结果也仍然会带有混淆痕迹。

### 17.2 强依赖 define 结构

`decompileGameJS()` 有这样的判断：

```ts
if (!code.includes('define(') || !code.includes('function(require, module, exports)')) return
```

如果微信未来改变小游戏模块包装格式，或者某些游戏使用了特殊打包方式，当前逻辑可能无法拆出模块。

### 17.3 执行沙箱不是完整微信运行时

`createVM()` 只是提供足够的假环境，目标是让模块注册过程跑起来。

它不保证完整模拟：

- 渲染引擎。
- 游戏主循环。
- WebGL/Canvas 行为。
- 真实微信 API。
- 网络请求。
- 文件系统。

### 17.4 只能拆出注册过的模块

如果某些代码没有通过 `define()` 注册，或者注册过程被条件判断阻断，它就可能不会被 `_parseJsDefine()` 捕获。

## 18. 推荐阅读顺序

如果只关心小游戏，建议按这个顺序读：

1. `src/interface/unpack-wxapkg.ts`

   看如何从包里释放文件，以及如何判断 `game.js` 存在。

2. `src/decompilation-controller.ts`

   看如何从 `packInfo.appType === 'game'` 进入 `GameDecompilation`。

3. `src/interface/game-decompilation.ts`

   看小游戏完整还原流程。

4. `src/utils/get-pack-codeInfo.ts`

   看小游戏读取哪些关键文件。

5. `src/interface/base-decompilation.ts`

   重点看 `_parseJsDefine()` 和 `decompileAppWorkers()`。

6. `src/utils/create-vm.ts`、`src/utils/wx-dom.ts`

   理解为什么需要沙箱和假微信环境。

## 19. 一句话总结

小游戏代码还原的核心是：

```text
小游戏包解开后，game.js / subContext.js / workers.js 里保留了 define(name, factory) 形式的模块注册信息。
Wedecode 在沙箱里执行这些打包文件，用自己的 define() 截获模块名和模块函数，
再从函数体中抽出代码、格式化、写回独立 .js 文件，
同时把 app-config.json 整理成 game.json，并补回 workers 配置。
```

所以小游戏方向最值得关注的不是 WXML/WXSS 逆向，而是：

```text
define 拦截
函数体抽取
沙箱执行
worker 配置回写
```
