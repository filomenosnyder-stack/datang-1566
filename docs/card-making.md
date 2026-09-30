# 超糖（taberna）制卡 · 全流程与格式

> 从「和亲」「大明王朝1566」「雍正王朝」「现代韩国模拟器」「庄园主模拟器」「白霜（地雷表妹）」六张卡里攒出来的。
> 每一条都是踩过坑才写下的，不是推测。

---

## 〇 · 一句话

**一张卡 = 一份 JSON。** 里面有四样东西：**卡面**（`description`，一整页 HTML）、**三份提示词**（`systemPrompt` / `prePrompt` / `outputFormat`）、**世界书**（`embeddedWorldBook`）、**对话界面样式**（`assets.customCss`）。

前两样是给**人**看的，后两样是给**模型**看的。**配对关系错了，卡就是瘸的。**

---

## 一 · 产物格式

### 1.1 顶层只有 4 个键

```json
{ "version": 2, "ns": "taberna", "exportedAt": "…", "braindance": { … } }
```

### 1.2 `braindance` 正好 12 个键（不多不少）

```
title  summary  description  language  experienceType  systemPrompt
jailbreakEnabled  prePrompt  outputFormat  cssCompatMode  assets  embeddedWorldBook
```

⚠ **少一个键或多一个键，平台会拒收整张卡。** 构建脚本必须校验这一点。

### 1.3 `assets` 只有两个键

```json
{ "cover": { "desktop": "https://…" }, "customCss": "…" }
```

### 1.4 `embeddedWorldBook` 每条 11 个字段

```json
{
  "content": "条目正文（已剥掉 markdown）",
  "enabled": true,
  "logic": 2,
  "triggerArea": 1,
  "triggerAreas": [3, 4],
  "scanDepth": 2,
  "insertionPosition": 2,
  "insertionPositions": [2],
  "probability": 100,
  "insertionOrder": 整数，全局唯一,
  "triggerExpression": "关键词A OR 关键词B OR 关键词C"
}
```

---

## 二 · 工程目录

```
<卡名>（制卡素材）/
  build.mjs                  组装 + 校验，唯一入口
  src/
    meta.json                title / summary / cover / assetBase / artDir
    systemPrompt.md          世界观 + 铁律 + 文风 + 禁止项
    prePrompt.md             开局协议（玩家看不到）
    outputFormat.md          思维链 / 正文 / 局面 / 记忆区 四段
    worldbook/*.md           世界书源文件（可读的 markdown）
    frontend/
      index.html             → description
      custom.css             → assets.customCss
      assets/*.jpg           本地图（构建时替换成绝对 URL）
  <卡名>.json                构建产物
```

⚠ **源文件保持可读的 markdown，规范化只在构建时做。** 别在源文件里手写纯文本版——改不动。

---

## 三 · 世界书条目格式

```markdown
=== 【条目名】后缀
trigger: 关键词A OR 关键词B OR 关键词C
order: 582
---
正文……
```

**解析正则**（`build.mjs` 里那条，别改）：

```js
const HEAD_RE = /^=== 【(.*?)】([^\n]*)\ntrigger: (.*?)\norder: (\d+)\n---\n([\s\S]*)$/;
// 切分用： text.split(/\n(?==== 【)/)
```

⚠⚠ **三个必踩的坑：**

1. **名字必须带 `【】`。** 写成 `=== 海河市` 会被整条静默跳过（我就这么丢过 11 条）。
2. **JS 正则里没有 `\Z`** —— 写 `\Z` 是字面量 Z，会吃掉每个文件的最后一条。用 `$`。
3. **换行必须归一化。** 不归一化的话 `^…\n` 这类正则会整片失配，**词条静默消失**。

---

## 四 · 触发词规则（平台硬约束）

| 规则 | 说明 |
|---|---|
| **不能是单字** | 「药」「躁」「哭」都会被拒 → 改「什么药」「躁期」「她哭」 |
| **不能含空格、连字符、顿号、括号** | 含了就拒收**整张卡**，不是拒收那一条 |
| **同一关键词最多 2 个条目** | 第 3 条就超 |
| **按空白拆分** | `trigger: A OR B OR C`，用 ` OR ` 分隔 |

**自查脚本**（构建时跑）：

```python
kw = Counter()
for e in entries:
    for k in e.triggerExpression.split(' OR '):
        kw[k] += 1
        if len(k) < 2: 报错
        if re.search(r'[\s\-—·,，、（）()「」]', k): 报错
assert max(kw.values()) <= 2
```

⚠ **改人名时一定要全库搜旧名。** 我改过一次「吉尔→戈弗雷」，只改了 `wb_danger`，忘了 `wb_outside`，结果**同一个人劈成两个名字**，触发词还指着错的那个。

---

## 五 · 平台硬约束

| | |
|---|---|
| **`<script>` 不会被剥** | ⚠ 这条**我错判过一整轮**。大明卡里有 16KB 脚本在正常跑。JS 随便用。 |
| **`description` 是 HTML 片段** | 结尾是 `</div>`，**没有 `</body>` / `</html>`** —— 注入脚本直接追加，别 `replace('</body>')` |
| **世界书渲染为纯文本** | 条目里的 `**` 会变成垃圾字符 → 构建时剥（`plain()`） |
| **表格转纯文本** | `\|` 分隔的表格要转成 `列 ｜ 列`，分隔行删掉 |
| **图片必须绝对 URL** | 相对路径全部变破图 |
| **同一个 URL 换内容 = 缓存不变** | ⚠⚠ **平台/浏览器按 URL 缓存**。改图必须**换文件名**（`cover.jpg` → `cover-v2.jpg`），否则永远给你看旧的 |
| **平台会存一份 description 副本** | 你把卡导进去那一刻，它抄一份。之后改 json 它不知道 —— **必须重新导入** |

---

## 六 · 前端（`description`）

### 6.1 页签：纯 CSS radio + `:checked ~`

```html
<div class="w" id="wRoot">
<style>…</style>
<input class="w-radio" type="radio" name="wtab" id="wt-ta" checked>
…
<div class="w-body">
  <header class="w-cover">…</header>
  <nav class="w-nav"><div class="w-nav-in">
    <label for="wt-ta"><b>壹</b>她</label>…
  </div></nav>
  <section class="w-panel" id="p-ta">…</section>
</div>
<script>…</script>
</div>
```

```css
.w-radio{position:absolute;width:1px;height:1px;opacity:0;pointer-events:none;margin:0}
#wt-ta:checked~.w-body #p-ta{display:block}
#wt-ta:checked~.w-body label[for="wt-ta"]{color:#fff;border-bottom-color:var(--key)}
```

⚠ **radio 必须是 `.w-body` 的前置兄弟**，否则 `~` 够不到，页签永远点不动。

⚠ 两条线（甲/乙）那种开关同理，但 **不要另搞一套 `data-line` 机制** —— 我写过一版 `data-line="soldier"` 写死在根节点上、全文没有任何代码改它，结果**乙线永远不显示**。**统一走 radio。**

### 6.2 复制按钮（照抄大明）

```js
function copyText(t){
  function fb(){
    var x=document.createElement("textarea");
    x.value=t; x.style.position="fixed"; x.style.left="-9999px";
    document.body.appendChild(x); x.select();
    var ok=document.execCommand("copy"); document.body.removeChild(x);
    say(ok?"已复制":"复制失败 —— 手动选中", ok);
  }
  if(navigator.clipboard&&navigator.clipboard.writeText)
    navigator.clipboard.writeText(t).then(function(){say("已复制",true);}, fb);
  else fb();
}
```

### 6.3 开局表单

- 用 **`<textarea>`** 不用 `<input>` —— textarea 的内容**全选复制时会被带走**，input 的 value 不会
- 逐字段格子（`.w-cell`），**复制逻辑扫全部格子**而不是写死字段名 —— 以后加格子不用改代码
- ⚠ **填格要同步执行**，不要 `setTimeout(…,0)` —— 否则点完选项马上点复制会读到旧值

### 6.4 布局（三条我犯过的错）

| 症状 | 真实原因 |
|---|---|
| 「边框停航母」 | `object-fit:cover` 把**方图**拉去铺满横条 → 裁掉了头。改 `width:100%;height:auto` |
| 「旁边一大排空」 | 页签栏铺满全宽、标签挤左边。改：内容 `max-width` 居中 + 标签 `flex:1` 平分 |
| 「点了没反应」 | 结构没错，是**反馈太细**（只变个边框颜色）。点一下要**看得见的东西变** |

⚠⚠ **改前端必须先在真实宽度下渲染截图看一眼再交。** 我连着三轮栽在这。

---

## 七 · `customCss`（对话界面样式）

⚠⚠ **这不是卡面样式，是聊天界面的样式。** 它 style 的是 `.markdown-body` 和三个折叠块。

**配对关系**：`outputFormat` 吐什么 `data-type`，`customCss` 就得画什么。**少一边，聊天里就是裸的。**

```css
.markdown-body details[data-type="cot"]{…}      /* 思维链 */
.markdown-body details[data-type="status"]{…}   /* 局面 */
.markdown-body details[data-type="memory"]{…}   /* 记忆区 */
```

**铁律：**
- ⚠ **全写字面量，不用 `var(--x)`。** 平台给每个选择器加前缀，`:root` 上的变量够不到 → 写了就是空颜色
- 想用变量就定义在 `.markdown-body` 上（大明是这么干的），不要 `:root`
- `<table>` 的 th/td 颜色要**显式声明**，引擎会强制表格为黑

---

## 八 · `outputFormat`（四段）

```
第一段 · 思维链   <details data-type="cot">      默认收起
第二段 · 正文      裸文本，800~1000 字以上
第三段 · 局面   <details data-type="status">   默认收起
第四段 · 记忆区  <details data-type="memory">   默认收起
```

⚠ **默认收起 = 不写 `open`。** 要默认展开才写 `open`。

**思维链该有哪些节**（可复用骨架）：
`▍〇 先读账本`（防「回到开局」的闸门）→ `▍一 现在是什么时候` → `▍二 盘面` → `▍三 算账` → `▍四 谁不高兴了` → … → `▍九 落笔`

---

## 九 · 生图（BotCF）

```
Base URL  https://botcf.com/v1
POST      /v1/images/generations
body      {"model":"gpt-image-2","prompt":"…","n":1,"size":"1024x1536"}
返回      data[0].b64_json
```

⚠⚠ **Key 要剥掉前缀。** 用户给的是 `consolesk-XXXX`，直接当 Bearer 用会 **401 Invalid token**，要用 `consolesk-` **后面那段**。

⚠ **一批别超过 3 张** —— 每张 30–60 秒，超了 Bash 超时。

**画风**：用户要**动漫化**（轻小说 key visual：干净赛璐璐 + 清晰线稿 + 大眼 + 好看的脸），**不是**水彩写实、**不是**手绘脏感。

```
Japanese anime illustration, light novel key visual style, clean crisp line art,
cel shading with soft gradients, delicate pretty features, soft warm rim lighting,
detailed anime painted background, no 3D render, no plastic gloss, no photorealism
```

- ⚠ **写「村姑」别写脏** —— 不写 dirt / chapped / rough hands
- ⚠ **也别滑向幼态** —— 写 19 岁容易被画成 14 岁，要加 `clearly adult young woman, mature pretty face, adult figure, not a child`
- ⚠ **不要抠历史考据** —— 衣服好看就行

---

## 十 · 导出给用户

⚠⚠ **用户那份 JSON 是从平台 round-trip 回来的**，不是你的构建产物：

- 标题**被用户改过**
- 封面**被换成他的图床**（miha.wiki）
- 描述里的图**被平台重传成短链**

**所以绝对不能直接覆盖。** 写一个导出脚本，只换内容、保留元信息：

```python
out = json.loads(json.dumps(user_card))          # 以用户那份为底
ob  = out['braindance']
ob['description'] = 把构建产物里的 GitHub 图 URL 按索引映射回 miha.wiki
ob['systemPrompt'] = new['systemPrompt']
ob['prePrompt']    = new['prePrompt']
ob['outputFormat'] = new['outputFormat']
ob['embeddedWorldBook'] = new['embeddedWorldBook']
ob['assets']['customCss'] = new['assets']['customCss']
# title / summary / assets.cover 一律不动
shutil.copy2(f, f+'.bak-'+时间戳)                 # 先备份
```

**图床映射**：构建产物是 GitHub URL，用户的是 miha.wiki，**按出现顺序一一对应**。

**独立世界书**：平台「切世界书」要的是**单独的数组文件**，不是卡里的 `embeddedWorldBook`。两种格式都要出：

| 格式 | 字段 |
|---|---|
| 十字段 | `keys`(数组) / `content` / `enabled` / `logic` / `triggerArea` / `triggerAreas` / `insertionPosition` / `insertionPositions` / `insertionOrder` / `probability` |
| 39 键 | `group` / `match_type` / `key`(`_or_A@wb@B@wb@C`) / `key_region` / `value_type` / `value` / `value_configs` / `value_region` / `sort` / `depth` / `probability` / `enable` |

---

## 十一 · 验证（不验证等于没做）

**1 · 构建校验**（写进 `build.mjs`，不过就不出卡）
- 顶层 4 键、`braindance` 12 键
- `insertionOrder` 全局唯一
- 无单字触发词、无非法字符、无关键词 >2 次
- `description` 里有 `<style>`、有绝对 URL、`assets/` 全被替换
- `outputFormat` 的 `data-type` ⟷ `customCss` 一一配对
- 世界书正文无残留 `**` / `|` / `#`
- `JSON.parse` 通过

**2 · 浏览器实测**（headless Chrome）

```bash
chrome --headless=new --disable-gpu --no-sandbox \
       --window-size=1400,900 --virtual-time-budget=8000 \
       --dump-dom file:///…/_t.html
```

抓结果的手法：**把 `navigator.clipboard` 用 `Object.defineProperty` 换成假对象**，把复制出来的文本写进 `document.body` 的 `data-` 属性，再 grep。

⚠ **这张卡的 HTML 没有 `<head>` / `<title>`，`document.title` 写不进去** —— 必须往 body 写属性。

⚠ **截图空白 ≠ 没渲染**，可能是**纯黑**，也可能是入场动画被冻在 `opacity:0`。要显式 `animation:none`。**解 PNG 像素**比肉眼可靠。

**3 · 逐项功能测**
页签每个都点一遍 → 每个只显示自己那一页；两条线都切一遍；复制按钮每条线都吐对；空字段正确跳过。

---

## 十二 · 文风（用户骂出来的）

**❌ 最招人烦的是「给行为下抒情结论」** —— 列完事实再补一句解释它什么意思。

被抓出来的原句：
> 「她哪天开始抬头看你了，说明村里人开始不怕你了 —— 那不是好事」
> 「她是村里人对你的态度的温度计」
> 「她不是不难过，是把难过省下来干活了」
> 「灰溪不是一块好地，是一块还没被用对的地」

⚠⚠ **判据：删掉这句，信息量不变 → 就是 AI 腔，删。**

**❌ 另一窝（脸上不能写"有戏"）**
「眼下有青影」「眼底闪过一丝」「眼里的光暗下去」「眼神复杂」「欲言又止」

> ⚠ **脸上能写的只有一个东西：她做了什么。** 别写眼睛怎么"有戏"，写她看哪儿、看了多久、看没看你。

**❌ 自己跟自己打架**
我在 `systemPrompt` 里手写「她眼底闪过一丝……这种句子一个都不许有」，转头在卡面图注里写了「眼下的青影」。**禁令是我写的，犯的也是我。**

**❌ 现代词** —— 系统 / 面板 / 任务 / 数值 / 等级 / 技能 / 手机 / 公司 / 老板 / 上班 / 小时 / 分钟 / 公里
⚠ 但**禁用词清单本身会触发"出现现代词"的告警** —— 那是正常的，别去改。

---

## 十三 · 开放世界（如果这张卡是）

**不许做的：**
- ❌ **排节奏** —— 不写「第二轮该摸清家底」「头两三年你什么都赚不到」
- ❌ **埋引信** —— 不预设任何事"早晚要爆"
- ❌ **替玩家找事** —— 他想干什么就推演什么
- ❌ **给路线** —— 不写「三个阶段：活下去 → 转向牧业 → 加工扩大」

**必须做的：**
- ✅ **只保证世界是真的** —— 季节在走、租子在到期、人在变老
- ✅ **他不做的事，让它在背景里照常发生**

---

## 十四 · 不替玩家做任何事

```
❌ 不替他说话    ❌ 不替他选择
❌ 不替他移动    ❌ 不替他感受
```

⚠⚠ **「他的身子是他的。」** 不写「你走过去」「你推开门」「他出了门」「他转身去了」。
**有人要找他，就让他自己上门、自己在门外站着 —— 不要把他带到人跟前。**

⚠ **不许强塞 NPC。** 玩家没写在开局里的角色，**一个都不许出现** —— 世界书里有也不行，得等他自己去找。
⚠ 但**要区分「开局不许塞」和「以后可以有」** —— 我写死过一次「这张卡里没有管家」，用户回「那我他妈以后加管家呢」。

---

## 十五 · 全流程 checklist

```
□ 1  建目录 <卡名>（制卡素材）/src/{worldbook,frontend/assets}
□ 2  写 src/meta.json（title / summary / cover / assetBase / artDir）
□ 3  写 src/worldbook/*.md —— === 【名】 trigger: order: --- 格式
□ 4  写 src/systemPrompt.md（世界观 + 铁律 + 文风 + 禁止项）
□ 5  写 src/prePrompt.md（开局协议）
□ 6  写 src/outputFormat.md（四段 + 三个 data-type）
□ 7  写 src/frontend/index.html（页签 radio + 折叠块 + 复制按钮）
□ 8  写 src/frontend/custom.css（配对 data-type，全字面量）
□ 9  生图 → 压到 300KB 上下 → 传 GitHub Pages
□ 10 node build.mjs —— 校验必须全过
□ 11 headless 截图看布局 / dump-dom 测功能
□ 12 写导出脚本，保住用户的 title / summary / cover，只换内容
□ 13 另出一份独立世界书（十字段 + 39 键两种都出）
□ 14 备份 .bak-<时间戳>
□ 15 逐项复查残留（旧词、旧图名、旧数值）
```

---

## 十六 · 我踩过的坑（按疼的程度）

1. **⚠⚠ 以为自己判断对了平台限制，其实是自己的 bug。** 我把「前端点不动」误诊成「平台剥 script」，为此把整张卡改成纯 CSS。后来发现平台根本不剥 —— 大明卡里 16KB 脚本跑得好好的。**先怀疑自己。**
2. **⚠⚠ 同名换图 = 缓存不变。** 换图必须换文件名。
3. **⚠⚠ 平台存 description 副本。** 改 json 不重新导入，用户永远看旧的。
4. **⚠⚠ 覆盖用户的卡。** 他那份是 round-trip 回来的，标题封面都是他改的。**只换内容，别动元信息。**
5. **⚠ 说改了其实没改。** 我在报告里写「男爵外甥改名叫戈弗雷了」，但脚本里根本没有那一行 —— 只改了另一个文件。**报告前逐条 grep 验证。**
6. **⚠ 数字改了不连带重算。** 户数 25→24，得跟着改：口数、份地、自营地、收成、地租、合计、净剩、人日、可用工。**改一个数，全库搜它。**
7. **⚠ 自相矛盾。** 卡里禁「小时/分钟」，世界书里写了三次「两个小时」。**卡片自己的规则，卡片自己违反了。**
8. **⚠ 悬空引用。** 写「（见【继承】）」，全库只有【继承法】。
9. **⚠ 触发词撞车。** 两个不同的人共用同一个名字（吉尔 / 休 / 汤姆），触发词会指错人。
10. **⚠ 双扣。** 亩产表的「净得」已经扣过种子，出账表又列了一遍种子。
11. **⚠ 过度设计。** 硬给卡造一个"引擎"（电话号码 / 身份焦虑），用户根本不想要。
12. **⚠ 该问的时候不问，不该猜的时候瞎猜。** 用户说「改花刀」，我按自己的理解写了一整套；他说的是给**卡里 AI 看的描写**，不是生图提示词。
