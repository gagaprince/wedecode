# wxapkg 解密逻辑详解

本文档专门解释当前项目中 `wxapkg` 解密逻辑的来源、调用链、关键代码和算法含义。

相关源码：

- `src/interface/unpack-wxapkg.ts`
- `src/utils/decrypt-wxapkg.ts`
- `src/utils/common.ts`
- `src/decompilation-controller.ts`

## 1. 解密发生在什么时候

解密不是一个独立命令，而是在“解包”开始前自动触发。

整体流程是：

```text
DecompilationController.singlePackMode()
  -> UnpackWxapkg.unpackWxapkg()
    -> 读取 wxapkg 文件为 Buffer
    -> 判断是否需要解密
    -> 如果需要，调用 decryptWxapkg()
    -> 解密后的 Buffer 继续进入文件索引解析
```

入口代码在 `src/decompilation-controller.ts`：

```ts
private async singlePackMode(wxapkgPath: string, outputPath: string): Promise<void> {
  const packInfo = await UnpackWxapkg.unpackWxapkg(
    wxapkgPath,
    outputPath,
    this.config.wxid
  )
  ...
}
```

这里会把用户传入的 `--wxid` 继续传给解包器。`wxid` 是解密所需的重要参数。

## 2. 如何判断包是否需要解密

判断逻辑在 `src/utils/decrypt-wxapkg.ts`：

```ts
export function needsDecryption(data: Buffer): boolean {
  if (data.length < 14) return false;

  const firstMark = data.readUInt8(0);
  const lastMark = data.readUInt8(13);

  // 如果不是有效的 wxapkg 标记，说明需要解密
  return firstMark !== 0xbe || lastMark !== 0xed;
}
```

含义：

- 正常可解析的 `wxapkg`，第 `0` 字节应该是 `0xbe`。
- 正常可解析的 `wxapkg`，第 `13` 字节应该是 `0xed`。
- 如果这两个位置不符合预期，就认为它可能是微信本地缓存里的加密包。

这个判断本质上是在检查 `wxapkg` 的魔数特征。

为什么是这两个字节？

因为后续解包逻辑读取文件索引时，也依赖这个头部格式：

```ts
const headerBuffer = __APP_BUF__.subarray(0, 18)
let firstMark = headerBuffer.readUInt8(0);
let lastMark = headerBuffer.readUInt8(13);

if (firstMark !== 0xbe || lastMark !== 0xed) {
  ...
}
```

也就是说，解密后的数据必须变回这种格式，后面的 `genFileList()` 才能继续解析包内文件。

## 3. 解密前如何获得 wxid

解密逻辑需要 `wxid`。这里的 `wxid` 实际上通常就是小程序 appid，比如：

```text
wx1234567890abcdef
```

项目里有两个来源：

1. 用户通过命令行传入：

   ```text
   --wxid wx1234567890abcdef
   ```

2. 从包路径中自动提取。

相关代码在 `src/interface/unpack-wxapkg.ts`：

```ts
if (needsDecryption(__APP_BUF__)) {
  const wxid = externalWxid || UnpackWxapkg.extractWxid(inputPath);
  if (wxid) {
    printLog(`\n ▶ 检测到加密包，正在解密... (wxid: ${colors.blue(wxid)})`, {isStart: true});
    __APP_BUF__ = decryptWxapkg(wxid, __APP_BUF__);
  } else {
    console.log(`检测到加密包，但无法从路径中提取 wxid`);
    process.exit(0);
  }
}
```

自动提取逻辑：

```ts
private static extractWxid(filePath: string): string | null {
  const normalizedPath = path.normalize(filePath);
  const parts = normalizedPath.split(path.sep);
  for (const part of parts) {
    if (isWxAppid(part)) {
      return part;
    }
  }
  return null;
}
```

它会把路径按目录切开，然后找符合 appid 规则的目录名。

appid 判断逻辑在 `src/utils/common.ts`：

```ts
export function isWxAppid(str: string) {
  const reg = /^wx[0-9a-f]{14,16}$/i
  str = str.trim()
  return str.length >= 16 && str.length <= 18 && reg.test(str)
}
```

所以路径里如果有这样的目录名，就可以自动解密：

```text
.../wx1234567890abcdef/__APP__.wxapkg
```

如果路径里没有 appid，用户就需要显式传 `--wxid`。

## 4. 解密算法从哪里来

这个算法不是微信官方公开文档提供的，而是社区通过逆向微信客户端和样本验证沉淀出来的规则。

微信客户端要运行本地缓存的小程序，就必须知道如何把加密包还原为可读的 `wxapkg`。逆向人员通常会通过这些方式分析：

- 对比加密包和已解密包的二进制差异。
- 逆向 PC/Mac 微信客户端中读取小程序缓存的逻辑。
- 搜索非常有特征的字符串，例如 `saltiest`、`the iv: 16 bytes`。
- 观察 AES、PBKDF2、XOR 等解密调用。
- 用大量样本验证解密后是否能得到合法 `wxapkg` 头和文件索引表。

当前项目只是把这套已知规则实现成 Node.js 代码。

## 5. 解密函数总览

核心函数在 `src/utils/decrypt-wxapkg.ts`：

```ts
export function decryptWxapkg(wxid: string, encryptedData: Buffer): Buffer {
  const salt = "saltiest";
  const iv = Buffer.from("the iv: 16 bytes");

  // 使用 PBKDF2 派生密钥
  const dk = crypto.pbkdf2Sync(wxid, salt, 1000, 32, "sha1");

  // 创建 AES-CBC 解密器
  const decipher = crypto.createDecipheriv(
    "aes-256-cbc",
    new Uint8Array(dk),
    new Uint8Array(iv),
  );
  decipher.setAutoPadding(false);

  // 解密前 1024 字节 (跳过前 6 字节)
  const encryptedPart = encryptedData.subarray(6, 6 + 1024);
  const updateResult = decipher.update(new Uint8Array(encryptedPart));
  const finalResult = decipher.final();
  const decryptedPart = Buffer.concat([
    new Uint8Array(updateResult),
    new Uint8Array(finalResult),
  ]);

  // XOR 解密剩余数据
  const xorKey = wxid.length >= 2 ? wxid.charCodeAt(wxid.length - 2) : 0x66;
  const remainingData = encryptedData.subarray(6 + 1024);
  const xorDecrypted = Buffer.alloc(remainingData.length);
  for (let i = 0; i < remainingData.length; i++) {
    xorDecrypted[i] = remainingData[i] ^ xorKey;
  }

  // 合并解密数据 (取前 1023 字节 + XOR 解密部分)
  return Buffer.concat([
    new Uint8Array(decryptedPart.subarray(0, 1023)),
    new Uint8Array(xorDecrypted),
  ]);
}
```

这个函数可以拆成五步：

```text
1. 准备 AES 参数
2. 用 wxid 派生 AES key
3. AES 解密头部 1024 字节
4. XOR 解密剩余数据
5. 拼接成解密后的 wxapkg Buffer
```

## 6. 第一步：准备 AES 参数

```ts
const salt = "saltiest";
const iv = Buffer.from("the iv: 16 bytes");
```

这里有两个固定参数：

- `salt`：用于 PBKDF2 派生密钥。
- `iv`：AES-CBC 的初始化向量。

`"the iv: 16 bytes"` 的长度正好是 16 字节，符合 AES-CBC 对 IV 长度的要求。

这两个值不是项目作者随意设计的，而是已知微信包解密规则的一部分。

## 7. 第二步：用 wxid 派生 AES key

```ts
const dk = crypto.pbkdf2Sync(wxid, salt, 1000, 32, "sha1");
```

这行代码使用 Node.js 的 `crypto.pbkdf2Sync()` 派生密钥。

参数含义：

```text
password    = wxid
salt        = "saltiest"
iterations  = 1000
key length  = 32 字节
digest      = sha1
```

为什么 key 长度是 32 字节？

因为后面使用的是 `aes-256-cbc`。AES-256 需要 256 bit 密钥：

```text
256 bit = 32 byte
```

所以 PBKDF2 输出 32 字节。

这里的 `wxid` 是整个解密过程的核心。appid 不对，派生出的 key 就不对，AES 解出来的数据也就不是合法 `wxapkg`。

## 8. 第三步：创建 AES-256-CBC 解密器

```ts
const decipher = crypto.createDecipheriv(
  "aes-256-cbc",
  new Uint8Array(dk),
  new Uint8Array(iv),
);
decipher.setAutoPadding(false);
```

含义：

- 算法：`aes-256-cbc`
- 密钥：上一步 PBKDF2 派生出的 `dk`
- IV：固定字符串 `"the iv: 16 bytes"`
- 关闭自动 padding

为什么关闭自动 padding？

这里处理的是微信包的固定分段格式，不是常见的“加密一段文本再自动补齐 padding”的模式。如果开启自动 padding，Node.js 可能会尝试按 PKCS padding 规则处理末尾，反而破坏真实数据。

## 9. 第四步：AES 解密头部 1024 字节

```ts
const encryptedPart = encryptedData.subarray(6, 6 + 1024);
const updateResult = decipher.update(new Uint8Array(encryptedPart));
const finalResult = decipher.final();
const decryptedPart = Buffer.concat([
  new Uint8Array(updateResult),
  new Uint8Array(finalResult),
]);
```

这里有两个关键点：

1. 跳过前 6 字节。

   ```ts
   encryptedData.subarray(6, 6 + 1024)
   ```

   加密包的前 6 字节不是最终解密后 `wxapkg` 的正文，所以解密时从偏移 `6` 开始。

2. 只用 AES 解密 1024 字节。

   微信这个加密格式不是整个文件都用 AES。它只对靠前的一段使用 AES，后面的数据使用更简单的 XOR。

可以把文件看成这样：

```text
加密包 encryptedData

[0..5]              加密包额外头部，解密结果里不保留
[6..1029]           AES-256-CBC 加密段，共 1024 字节
[1030..end]         XOR 加密段
```

## 10. 第五步：XOR 解密剩余数据

```ts
const xorKey = wxid.length >= 2 ? wxid.charCodeAt(wxid.length - 2) : 0x66;
const remainingData = encryptedData.subarray(6 + 1024);
const xorDecrypted = Buffer.alloc(remainingData.length);
for (let i = 0; i < remainingData.length; i++) {
  xorDecrypted[i] = remainingData[i] ^ xorKey;
}
```

XOR 是“异或”运算。规则是：

```text
0 ^ 0 = 0
0 ^ 1 = 1
1 ^ 0 = 1
1 ^ 1 = 0
```

它有一个重要特性：

```text
原始数据 ^ key = 加密数据
加密数据 ^ key = 原始数据
```

所以只要知道同一个 `key`，再异或一次就能还原。

这里的 key 是：

```ts
wxid.charCodeAt(wxid.length - 2)
```

也就是 `wxid` 倒数第二个字符的字符编码。

例如：

```text
wxid = wx1234567890abcdef
倒数第二个字符 = e
xorKey = 'e'.charCodeAt(0) = 101
```

然后对剩余每个字节执行：

```ts
xorDecrypted[i] = remainingData[i] ^ xorKey;
```

这一步会把 AES 段之后的大部分包内容还原出来。

## 11. 第六步：拼接解密结果

```ts
return Buffer.concat([
  new Uint8Array(decryptedPart.subarray(0, 1023)),
  new Uint8Array(xorDecrypted),
]);
```

这里把两段解密结果拼起来：

```text
AES 解密结果的前 1023 字节
+ XOR 解密后的剩余数据
= 解密后的 wxapkg Buffer
```

注意，代码 AES 解密时取了 `1024` 字节，但最后只使用 `1023` 字节：

```ts
decryptedPart.subarray(0, 1023)
```

这是微信加密包格式里的一个历史细节。很多同类解包工具也会这么处理。最终拼出来的数据应该以正常 `wxapkg` 包头开头：

```text
byte[0]  = 0xbe
byte[13] = 0xed
```

## 12. 解密后的数据如何继续解包

解密后的 Buffer 会回到 `UnpackWxapkg.unpackWxapkg()`：

```ts
__APP_BUF__ = decryptWxapkg(wxid, __APP_BUF__);

const fileList = []
fileList.splice(0, fileList.length, ...UnpackWxapkg.genFileList(__APP_BUF__))
```

`genFileList()` 会读取解密后的包头和文件索引表。

关键代码：

```ts
const headerBuffer = __APP_BUF__.subarray(0, 18)
let firstMark = headerBuffer.readUInt8(0);
let indexInfoLength = headerBuffer.readUInt32BE(5) + 18;
let lastMark = headerBuffer.readUInt8(13);
let fileCount = headerBuffer.readUInt32BE(14);
```

含义：

- 前 18 字节是包头信息。
- 第 5 到第 8 字节保存索引区长度。
- 第 14 到第 17 字节保存文件数量。
- 根据索引区内容，可以得到每个文件的路径、偏移量和大小。

如果解密正确，后面就可以正常切出包内文件：

```ts
const data = __APP_BUF__.subarray(info.off, info.off + info.size)
saveLocalFile(subRootPath, data)
```

如果解密失败，这一步通常会出现：

- 魔数不对。
- 文件数量异常。
- 文件路径乱码。
- 偏移和 size 不合理。
- 解出来的文件不是正常 JS/JSON/HTML。

## 13. 如何验证解密是否成功

可以从三个层级验证。

### 13.1 包头验证

解密后的 Buffer 应该满足：

```text
data.readUInt8(0)  === 0xbe
data.readUInt8(13) === 0xed
```

这是最基础的格式验证。

### 13.2 文件索引验证

`genFileList()` 应该能解析出合理的文件列表：

```text
fileCount > 0
文件路径是正常字符串
文件偏移 off 和大小 size 没有越界
```

### 13.3 解包结果验证

输出目录里应该能看到类似文件：

```text
app-config.json
app-service.js
app-wxss.js
page-frame.js
page-frame.html
workers.js
```

小游戏包里可能是：

```text
game.js
game.json
subContext.js
```

如果这些文件正常出现，并且内容可读，就说明解密和解包基本成功。

## 14. 为什么要用 wxid

这个算法把小程序 appid 参与进了密钥生成。

它带来的效果是：

```text
同一个加密算法
+ 不同 wxid
= 不同 AES key 和不同 XOR key
```

因此：

- A 小程序的 wxid 不能解 B 小程序的包。
- wxid 错一个字符，AES 段就会解错。
- wxid 错了，XOR key 也大概率错。

所以项目需要 `--wxid`，或者要求包路径中包含 appid。

## 15. 常见失败原因

### 15.1 路径里没有 appid

如果包路径不是这样：

```text
.../wx1234567890abcdef/__APP__.wxapkg
```

项目可能无法自动提取 `wxid`，需要手动传：

```text
wedecode ./__APP__.wxapkg --wxid wx1234567890abcdef
```

### 15.2 wxid 不正确

错误的 `wxid` 会导致：

- AES 解密失败或得到乱码。
- XOR 段还原错误。
- 后续包头和文件索引解析失败。

### 15.3 包本来就不是 wxapkg

`needsDecryption()` 只检查两个魔数字节。如果传入了完全无关文件，它也可能被判断为“需要解密”。但解密后依然无法通过 `genFileList()`。

### 15.4 微信版本格式变化

该算法对应的是社区已知的 PC/Mac 微信缓存包格式。微信客户端升级后，如果加密方式变化，当前逻辑就可能失效。

## 16. 一句话总结

当前项目的 `wxapkg` 解密逻辑是：

```text
如果包头不是正常 wxapkg 魔数，
就用 wxid 通过 PBKDF2 派生 AES-256 key，
跳过加密包前 6 字节，
AES-CBC 解密接下来的 1024 字节，
再用 wxid 倒数第二个字符作为 XOR key 解密剩余内容，
最后拼成正常 wxapkg Buffer，
交给文件索引解析器继续解包。
```

它不是项目作者自行设计的加密算法，而是根据微信客户端本地包加载行为逆向并验证出的规则。
