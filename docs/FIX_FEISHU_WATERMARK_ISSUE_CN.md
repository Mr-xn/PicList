# 修复飞书图片无法添加水印的问题

## 问题描述

从飞书（Feishu/Lark）等服务复制图片到 Typora，再上传到 PicList 时，水印无法正常添加。这是因为这些服务提供的图片 URL 没有文件扩展名，例如：

```
https://my.feishu.cn/space/api/box/stream/download/asynccode/?code=xxxxxxxxx
```

## 根本原因

问题出在 `piclist` 库（PicList-Core）的 `getURLFile` 函数（`src/utils/common.ts`）：

1. 从 URL 获取图片时，函数尝试从以下位置提取文件扩展名：
   - URL 参数 `wx_fmt`（用于微信图片）
   - 使用 `path.extname()` 从 URL 路径名提取
   
2. 对于飞书 URL，两种方法都无法获取扩展名，导致扩展名为空字符串 `''`

3. 在后续的图片处理流程（`src/core/Lifecycle.ts`）中，`isNeedAddWatermark()` 函数检查文件扩展名是否在支持的图片格式列表中

4. 空字符串 `''` 不在图片格式列表中，因此跳过了水印处理

## 解决方案

修复方法是在 PicList-Core 的 `getURLFile` 函数中添加 Content-Type 头部检测。当 URL 没有文件扩展名时，函数现在会：

1. 捕获 HTTP 响应头
2. 提取 `Content-Type` 头（例如 `image/jpeg`、`image/png`）
3. 将 Content-Type 映射到相应的文件扩展名
4. 使用此扩展名进行水印和压缩处理

## 实现

### 修改文件：`piclist/src/utils/common.ts`

添加一个辅助函数将 Content-Type 映射到文件扩展名：

```typescript
/**
 * 将 Content-Type 映射到文件扩展名
 */
const getExtensionFromContentType = (contentType: string): string => {
  const mimeToExt: Record<string, string> = {
    'image/jpeg': '.jpg',
    'image/jpg': '.jpg',
    'image/png': '.png',
    'image/gif': '.gif',
    'image/webp': '.webp',
    'image/bmp': '.bmp',
    'image/svg+xml': '.svg',
    'image/x-icon': '.ico',
    'image/vnd.microsoft.icon': '.ico',
    'image/avif': '.avif',
    'image/heic': '.heic',
    'image/heif': '.heif'
  }
  // 提取基础 content type（移除 charset 等参数）
  const baseType = contentType.split(';')[0].trim().toLowerCase()
  return mimeToExt[baseType] || ''
}
```

修改 `getURLFile` 函数以捕获响应头并在扩展名缺失时使用 Content-Type：

```typescript
export const getURLFile = async (url: string, ctx: IPicGo): Promise<IPathTransformedImgInfo> => {
  url = handleUrlEncode(url)
  let timeoutId: NodeJS.Timeout
  const requestFn = new Promise<IPathTransformedImgInfo>((resolve, reject) => {
    ;(async () => {
      try {
        // 捕获完整的响应对象，而不只是数据
        const resp = await ctx
          .request({
            method: 'get',
            url,
            resolveWithFullResponse: true,
            responseType: 'arraybuffer',
          })
        const res = resp.data as Buffer
        const headers = resp.headers || {}  // 获取响应头
        clearTimeout(timeoutId)
        const urlPath = new URL(url).pathname
        let extname = ''
        try {
          const urlParams = new URL(url).searchParams
          extname = urlParams.get('wx_fmt') || path.extname(urlPath) || ''
        } catch (_e) {
          extname = path.extname(urlPath) || ''
        }
        // 新增：如果从 URL 中未找到扩展名，尝试从 Content-Type 头获取
        if (!extname && headers['content-type']) {
          extname = getExtensionFromContentType(headers['content-type'])
        }
        if (!extname.startsWith('.') && extname) {
          extname = `.${extname}`
        }
        resolve({
          buffer: res,
          fileName: path.basename(urlPath),
          extname,
          success: true,
        })
      } catch (error: any) {
        clearTimeout(timeoutId)
        resolve({
          success: false,
          reason: `request ${url} error, ${error?.message ?? ''}`,
        })
      }
    })().catch(reject)
  })
  const timeoutPromise = new Promise<IPathTransformedImgInfo>((resolve): void => {
    timeoutId = setTimeout(() => {
      resolve({
        success: false,
        reason: `request ${url} timeout`,
      })
    }, 30000)
  })
  return Promise.race([requestFn, timeoutPromise])
}
```

## 测试

测试此修复：

1. 从飞书复制图片到 Typora
2. 在 Typora 中右键点击图片并选择"上传图片"
3. 验证 PicList 成功上传图片**并应用了水印**
4. 与下载图片后从访达上传进行对比 - 两种方式现在都应该正确应用水印

## 下一步

此修复应作为 Pull Request 提交到 PicList-Core 仓库：
- 仓库：https://github.com/Kuingsmile/PicList-Core
- 文件：`src/utils/common.ts`
- 函数：`getURLFile`

合并并发布到 npm 后，PicList 可以更新其 `piclist` 依赖以包含此修复。
