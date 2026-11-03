# 09 真实验收：证明页面真的出现在界面里

构建通过、门禁全绿，**都不等于界面里能看到它**。要证明，得真的把它跑起来。

分三层，逐层加码。

## 第一层：boot payload 里有这一行

起一个带 overlay 的实例：

```sh
DSH_HOME=/tmp/dsh-hello-test pnpm dsh web --patch apps/web/tests/hello-page.overlay.yml --port 3099 --no-open
```

> `DSH_HOME` 必须指到一个**可写**目录。默认 `~/.dsh` 在沙箱外，会报 `EROFS: read-only file system`。

然后带 token 访问并跟随重定向（303 → 200）：

```sh
TOKEN_URL="http://127.0.0.1:3099/?token=<token>"
curl -sL -c /tmp/cj -b /tmp/cj -o boot.html "$TOKEN_URL"
grep -o '.\{0,120\}dsh-experimental-hello-page.\{0,120\}' boot.html
```

能看到这样的记录就说明**插件已被真实 profile 加载**：

```json
{"id":"@deepseek-ai/dsh-experimental-hello-page","url":"plugins/??@deepseek-ai/dsh-experimental-hello-page/client.js&rev=08c5e0cecb5d","rev":"08c5e0cecb5d"}
```

## 第二层：模块路由能返回 bundle

注意模块 URL 是**批量形式**（`plugins/??a&b&c`），不是 `/plugins/<name>/client.js`：

```sh
curl -s -b /tmp/cj -o mod.js -w "%{http_code}\n" \
  "http://127.0.0.1:3099/plugins/??@deepseek-ai/dsh-experimental-hello-page/client.js&rev=<rev>"
```

200 且内容是我们产出的 bundle，就说明**浏览器拿得到代码**。

## 第三层：真浏览器点一遍

用仓库自带的 Playwright（`apps/web` 的 devDependency）：

```js
import { chromium } from 'playwright'

const browser = await chromium.launch({ args: ['--no-sandbox', '--disable-dev-shm-usage'] })
const page = await browser.newPage({ viewport: { width: 1440, height: 900 } })
await page.goto(TOKEN_URL, { waitUntil: 'load', timeout: 40000 })
await page.waitForTimeout(7000)

// 关掉首启弹窗（见下面的坑）
await page.keyboard.press('Escape')

await page.locator('button[aria-label="Hello"]').click()
console.log(await page.locator('h1').allTextContents())      // ["Hello page"]
await page.getByRole('button', { name: 'Increment', exact: true }).click()
console.log(await page.getByText(/^Count:/).first().textContent())  // "Count: 1"
```

### 三个必踩的坑

**1. 浏览器要装在沙箱可写的位置**

```sh
PLAYWRIGHT_BROWSERS_PATH=<工作区>/node_modules/.cache/ms-playwright \
  pnpm exec playwright install chromium
```

默认装到 `~/.cache/ms-playwright`，在沙箱外，会失败。

**2. 缺系统库**

chromium 起不来，报 `libnspr4.so: cannot open shared object file`。用 `ldd` 精确列出缺哪些：

```sh
ldd node_modules/.cache/ms-playwright/chromium_headless_shell-*/chrome-headless-shell-linux64/chrome-headless-shell | grep "not found"
```

Ubuntu 22.04 对应一条命令（**不要用 t64 后缀**，那是 24.04）：

```sh
sudo apt-get install -y libnspr4 libnss3 libasound2 libatk1.0-0 libatk-bridge2.0-0 \
  libatspi2.0-0 libgbm1 libxkbcommon0 libxcomposite1 libxdamage1 libxfixes3 libxrandr2
```

**3. 全屏遮罩吃掉点击**

`locator.click()` 超时，但元素明明可见——因为上层有遮罩。Playwright 只说 "click timeout"，完全不提遮挡。

用命中检测定位：

```js
const hit = await page.evaluate(() => {
  const b = document.querySelector('button[aria-label="Hello"]')
  const r = b.getBoundingClientRect()
  const el = document.elementFromPoint(r.x + r.width / 2, r.y + r.height / 2)
  return el ? el.tagName + '.' + el.className : null
})
```

本次遇到两个：

1. **首启预览弹窗**（"DSH Harness 0.2 还在预览"）—— 全屏 `_mask_*` 遮罩
2. **API Key 登录框** —— 关掉第一个之后才露出来

两个都能用 `Escape` 关掉。注意它们只在**全新 profile 首次启动**时出现。

## 另外两个环境细节

- **`/tmp` 在每次 bash 调用之间不保留**（沙箱独立临时区）。跨命令的临时文件要放在工作区内。
- 字体缓存不可写会刷 `Fontconfig error`，设 `XDG_CACHE_HOME` 到工作区即可，不影响渲染。

## 验收结果长这样才算过

```json
{
  "h1": ["Hello page"],
  "description": true,
  "countBefore": "Count: 0",
  "countAfter1": "Count: 1",
  "countAfter2": "Count: 2",
  "errors": []
}
```
