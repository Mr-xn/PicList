# PicList-Core Fix for Feishu/Lark Image Watermark Issue

This directory contains a fix for the issue where watermarks are not applied to images copied from Feishu/Lark (and similar services) that provide URLs without file extensions.

## Problem

Images from URLs like `https://my.feishu.cn/space/api/box/stream/download/asynccode/?code=xxx` don't get watermarks applied because the URL has no file extension.

## Solution

The fix detects the image type from the HTTP `Content-Type` header when the URL doesn't have a file extension.

## Files

- `common.ts.fixed` - Fixed version of `piclist/src/utils/common.ts` with Content-Type detection
- Diff shows changes needed in the PicList-Core repository

## How to Apply

### Option 1: Submit PR to PicList-Core (Recommended)

1. Fork https://github.com/Kuingsmile/PicList-Core
2. Apply the changes from `common.ts.fixed` to `src/utils/common.ts`
3. Submit a Pull Request

### Option 2: Local Patch (Temporary)

If you need the fix immediately before it's merged into PicList-Core:

1. Install patch-package:
   ```bash
   yarn add -D patch-package postinstall-postinstall
   ```

2. Add to package.json scripts:
   ```json
   {
     "scripts": {
       "postinstall": "patch-package"
     }
   }
   ```

3. After `yarn install`, manually apply the changes from `common.ts.fixed`:
   - Copy the `getExtensionFromContentType()` function (lines 186-207)
   - Replace the `getURLFile()` function with the updated version (lines 209-265)
   - Target file: `node_modules/piclist/dist/utils/common.js`

4. Create patch:
   ```bash
   npx patch-package piclist
   ```

## Changes Summary

The fix adds:

1. `getExtensionFromContentType()` - Maps MIME types to file extensions
2. Modified `getURLFile()` - Captures HTTP headers and uses Content-Type for extension detection

See `../docs/FIX_FEISHU_WATERMARK_ISSUE_CN.md` for detailed explanation in Chinese.
