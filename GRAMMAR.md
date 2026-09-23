# 防二传方案语法速查表

> 函数 · 方法 · 参数 · 正则 · 文件路径 · 常量

---

## 一、零宽字符常量

| 常量 | Unicode | 含义 | 转义写法 |
|------|---------|------|----------|
| Z0 | U+200B | 二进制 0 | `"\u200B"` |
| Z1 | U+200C | 二进制 1 | `"\u200C"` |
| PRE | U+200D U+FEFF | 水印前缀 | `"\u200D\uFEFF"` |
| SUF | U+FEFF U+200D | 水印后缀 | `"\uFEFF\u200D"` |

---

## 二、核心函数

| 函数名 | 参数 | 返回值 | 作用 |
|--------|------|--------|------|
| `encodeStealth` | text | 零宽字符串 | 文本编码成零宽字符 |
| `decodeStealth` | source | 字符串数组 | 零宽字符还原成文本 |
| `processDocXml` | xml, fullMark, shortMark | 新xml | 向文本节点插水印 |
| `injectDoc` | arrayBuffer, readerQQ, token, authorName, authorQQ | Uint8Array | 完整注入五层水印 |
| `analyzeDocx` | zip, filename | 结果对象 | 单个docx溯源分析 |
| `traceFile` | file | 结果数组 | 入口：区分zip和docx |

---

## 三、水印明文格式

| 类型 | 格式 |
|------|------|
| 完整水印 | `AUTH:作者名:作者QQ\|TRACE:读者QQ\|VERIFIED:1\|T:令牌短码` |
| 短水印 | `Q读者QQ` |

---

## 四、注入位置（文件路径）

| 路径 | 内容 | 注入方式 |
|------|------|----------|
| `word/document.xml` | 正文 | 文本节点 + 段落尾部隐藏Run |
| `word/header*.xml` | 页眉 | 文本节点 |
| `word/footer*.xml` | 页脚 | 文本节点 |
| `word/endnotes.xml` | 尾注 | 文本节点 |
| `word/footnotes.xml` | 脚注 | 文本节点 |
| `word/comments*.xml` | 批注 | 文本节点 |
| `word/_author_auth.dat` | 作者归属 | 新建文件 |
| `word/_dist_trace.dat` | 分发追踪 | 新建文件 |
| `docProps/custom.xml` | 自定义属性 | 改写XML |

---

## 五、自定义数据文件格式

| 文件 | 内容 |
|------|------|
| `word/_author_auth.dat` | `ORIGIN_AUTHOR=作者名`<br>`AUTHOR_QQ=作者QQ` |
| `word/_dist_trace.dat` | `DIST_TARGET_QQ=读者QQ`<br>`MAIL_VERIFIED=1`<br>`VERIFY_TOKEN=令牌`<br>`VERIFY_TIME=时间` |

---

## 六、自定义属性名

| 属性名 | 含义 |
|--------|------|
| `DistTargetQQ` | 这份文档发给了谁 |
| `OriginAuthorQQ` | 原创作者是谁 |

---

## 七、正则表达式

| 用途 | 正则 |
|------|------|
| 匹配文本节点 | `/<w:t(\s[^>]*)?>([\s\S]*?)<\/w:t>/g` |
| 匹配零宽水印 | `/PRE([\u200B\u200C]+)SUF/g` |
| 匹配完整水印 | `/AUTH:(.*?):(\d+)\|TRACE:(\d+)(?:\|VERIFIED:(\d))?/` |
| 匹配短水印 | `/^Q(\d{5,12})$/` |
| 匹配追踪QQ | `/DIST_TARGET_QQ=(\d+)/` |
| 匹配作者名 | `/ORIGIN_AUTHOR=(.*)/` |
| 匹配作者QQ | `/AUTHOR_QQ=(\d+)/` |
| 匹配自定义属性 | `/name="DistTargetQQ"[^>]*>[^<]*<vt:lpwstr>(\d+)</` |

---

## 八、接口路径

| 方法 | 路径 | 作用 |
|------|------|------|
| POST | `/api/public/qq-code/send` | 发送邮箱验证码 |
| POST | `/api/public/qq-code/verify` | 校验验证码，返回token |
| POST | `/api/public/doc/seal` | 托管密钥，返回docId+pageKey |
| POST | `/api/public/doc/unseal` | 凭token取回密钥 |

---

## 九、加密参数

| 项目 | 值 |
|------|------|
| 算法 | AES-256-GCM |
| 密钥长度 | 256 位 |
| IV 长度 | 12 字节 |
| 密钥导出格式 | raw |
| 加密对象 | JSON字符串（文档列表） |
| 打包压缩 | DEFLATE |

---

## 十、溯源结果对象

| 字段 | 类型 | 含义 |
|------|------|------|
| `filename` | string | 文件名 |
| `authorName` | string | 原作者名 |
| `authorQQ` | string | 原作者QQ |
| `leakerQQ` | string | 泄露者QQ |
| `verified` | boolean | 是否邮箱验证 |
| `hits` | string[] | 证据链数组 |

---

## 十一、证据链文字

| 命中位置 | 提示文字 |
|----------|----------|
| `_dist_trace.dat` | 颢核分发目标 |
| `MAIL_VERIFIED=1` | QQ邮箱验证通过 |
| `_author_auth.dat` | 原创归属 |
| XML零宽水印 | 正文隐形水印(文件名) |
| 短水印 | 短水印(文件名) |
| `custom.xml` | 自定义属性(DistTargetQQ) |

---

## 十二、命令行

| 命令 | 作用 |
|------|------|
| `anti-redistribute trace 文件.docx` | 溯源单个文件 |
| `anti-redistribute trace 文件.zip --json` | 批量溯源输出JSON |
| `anti-redistribute seal ./目录 --author "名字" --qq 号码` | 生成分发页 |

---

## 十三、依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| JSZip | 3.10.1 | docx/zip 解析与打包 |
| Web Crypto API | 浏览器原生 | AES-GCM 加解密 |
| TextEncoder/TextDecoder | 浏览器原生 | UTF-8 编解码 |
| Node.js | ≥ 16 | 命令行工具运行环境 |
