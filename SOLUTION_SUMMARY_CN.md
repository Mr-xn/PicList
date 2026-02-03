# 飞书图片水印问题分析与解决方案

## 问题总结

你提出的问题完全正确！从飞书复制到 Typora 的图片链接，形如：
```
![img](https://my.feishu.cn/space/api/box/stream/download/asynccode/?code=xxxxxxxxx)
```

这种URL**没有文件后缀名**，导致 PicList 在上传时**无法添加水印**。

## 根本原因

问题出在 PicList 依赖的核心库 `piclist` (PicList-Core) 中：

### 1. 文件扩展名检测失败

在 `piclist/src/utils/common.ts` 的 `getURLFile()` 函数中：

```typescript
// 当前代码只从这两个地方获取扩展名：
extname = urlParams.get('wx_fmt') || path.extname(urlPath) || ''
```

- **微信图片**：有 `wx_fmt` 参数
- **本地文件**：路径有扩展名
- **飞书URL**：路径是 `/space/api/box/stream/download/asynccode/`，没有扩展名！

结果：`extname = ''`（空字符串）

### 2. 水印处理被跳过

在 `piclist/src/core/Lifecycle.ts` 的图片处理流程中：

```typescript
// 判断是否需要添加水印
export const isNeedAddWatermark = (
  watermarkOptions: IBuildInWaterMarkOptions | undefined,
  fileExt: string,
): boolean => {
  fileExt = normalizeImageExt(fileExt)  // '' -> ''
  return (
    !!watermarkOptions && 
    !!watermarkOptions.isAddWatermark && 
    imageFormatList.includes(fileExt) &&  // 空字符串不在列表中！
    fileExt !== 'svg'
  )
}
```

由于空字符串不在 `imageFormatList` (`['jpg', 'jpeg', 'png', 'gif', 'webp', ...]`) 中，返回 `false`，**跳过水印处理**！

### 3. 为什么下载后再上传可以？

从浏览器下载图片后，文件有了明确的扩展名（如 `image.jpg`），所以可以正常添加水印。

## 解决方案

### 修复方法

在 `getURLFile()` 函数中，当URL没有扩展名时，从 HTTP 响应的 `Content-Type` 头检测图片类型：

```typescript
// 获取完整响应（包括headers）
const resp = await ctx.request({...})
const res = resp.data as Buffer
const headers = resp.headers || {}

// 原有逻辑：从URL获取扩展名
extname = urlParams.get('wx_fmt') || path.extname(urlPath) || ''

// 新增：如果URL没有扩展名，从Content-Type获取
if (!extname && headers['content-type']) {
  extname = getExtensionFromContentType(headers['content-type'])
  // 例如：'image/jpeg' -> '.jpg'
  //      'image/png' -> '.png'
}
```

### 映射关系

```typescript
const mimeToExt = {
  'image/jpeg': '.jpg',
  'image/png': '.png',
  'image/gif': '.gif',
  'image/webp': '.webp',
  // ... 其他格式
}
```

## 文件位置

修复代码和文档已添加到本仓库：

1. **详细文档**（中文）：
   - `docs/FIX_FEISHU_WATERMARK_ISSUE_CN.md`

2. **修复代码**：
   - `piclist-core-fix/common.ts.fixed`
   - `piclist-core-fix/README.md`

## 如何应用修复

### 方案一：提交PR到 PicList-Core（推荐）

这是最正确的方式，让所有用户受益：

1. Fork 仓库：https://github.com/Kuingsmile/PicList-Core
2. 应用 `piclist-core-fix/common.ts.fixed` 中的修改
3. 提交 Pull Request

### 方案二：临时本地补丁

**注意**：这个方法比较复杂，因为需要修改编译后的 JavaScript 代码。**推荐使用方案一**。

如果需要立即使用，可以：

1. 安装 patch-package：
   ```bash
   yarn add -D patch-package postinstall-postinstall
   ```

2. 在 `package.json` 中添加：
   ```json
   {
     "scripts": {
       "postinstall": "patch-package"
     }
   }
   ```

3. 修改 PicList-Core 源代码并重新构建，或等待官方修复

**说明**：`common.ts.fixed` 是 TypeScript 源码，需要编译为 JavaScript 后才能应用到 `node_modules/piclist/dist/utils/common.js`。直接修改编译后的 JS 文件较为困难，建议等待官方更新。

## 测试验证

应用修复后，测试步骤：

1. 从飞书复制图片到 Typora
2. 右键图片 -> 上传图片
3. 验证上传成功且**水印已添加**
4. 检查图片URL，确认是你配置的图床地址

## 技术细节

### 修改前
```typescript
const res = await ctx.request({
  method: 'get',
  url,
  resolveWithFullResponse: true,
  responseType: 'arraybuffer',
}).then(resp => resp.data as Buffer)
// ❌ 只获取数据，丢失了headers
```

### 修改后
```typescript
const resp = await ctx.request({
  method: 'get',
  url,
  resolveWithFullResponse: true,  // 保留完整响应
  responseType: 'arraybuffer',
})
const res = resp.data as Buffer
const headers = resp.headers || {}
// ✅ 保留headers，可以获取Content-Type
```

## 影响范围

此修复解决了所有**URL中没有文件扩展名**的图片服务，包括但不限于：

- ✅ 飞书/Lark
- ✅ 钉钉
- ✅ 企业微信
- ✅ 其他类似的内部图片服务

## 总结

你的分析完全正确！问题确实是：

> 飞书文档复制到 Typora 的图片连接，没有后缀，只做图片上传处理，没有对图片进行水印处理逻辑

解决方案是：**从 HTTP 响应的 Content-Type 头获取图片类型**，而不是仅依赖URL中的文件扩展名。
