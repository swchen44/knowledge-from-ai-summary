---
name: article-to-personal-kb
description: 讀取網頁文章（含需登入的頁面）或 YouTube 影片，翻譯成台灣繁體中文 Obsidian markdown 格式，連同圖片一起上傳到 GitHub 個人知識庫。輸入一個 URL，自動完成全流程。
disable-model-invocation: true
argument-hint: <url>
---

將 $ARGUMENTS 存入個人知識庫，完整執行以下步驟。

> [!important] **URL 類型偵測**：請先判斷 `$ARGUMENTS` 的類型，再選擇對應流程：
> - **GitHub Repo URL**（如 `https://github.com/owner/repo`）→ 執行 **程式碼分析流程**（步驟 A1–A5），然後跳到步驟 3
> - **YouTube URL** → 步驟 1（YouTube 分支）→ 步驟 2 跳過 → 步驟 3 開始
> - **一般網頁文章** → 步驟 1（文章分支）→ 步驟 2 → 步驟 3 開始

---

## 🔧 程式碼分析流程（GitHub Repo 專用）

> 僅當 `$ARGUMENTS` 為 GitHub Repo URL 時執行此區塊，完成後跳至步驟 3。

### 步驟 A1：Clone 目標 Repo 到暫存目錄

```bash
REPO_URL="$ARGUMENTS"
REPO_NAME=$(basename "$REPO_URL" .git)
TMP_CODE_DIR="/tmp/kb-code-$REPO_NAME"

# 淺層 clone（節省時間與空間）
git clone --depth=1 "$REPO_URL" "$TMP_CODE_DIR"
```

### 步驟 A2：收集 Repo 基本資訊

```bash
cd "$TMP_CODE_DIR"

# 取得語言統計
find . -name "*.py" -o -name "*.ts" -o -name "*.go" -o -name "*.rs" -o -name "*.java" \
  | grep -v node_modules | grep -v ".git" \
  | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -20

# 取得目錄結構（2層深）
find . -maxdepth 2 -not -path './.git*' -not -path './node_modules*' \
  | sort | head -60

# 讀取 README
cat README.md 2>/dev/null || cat README.rst 2>/dev/null || echo "無 README"

# 讀取主要設定檔
cat pyproject.toml 2>/dev/null
cat package.json 2>/dev/null
cat go.mod 2>/dev/null
cat Cargo.toml 2>/dev/null
cat pom.xml 2>/dev/null

# 統計程式碼行數
find . -name "*.py" -o -name "*.ts" -o -name "*.go" -o -name "*.rs" -o -name "*.java" \
  | grep -v node_modules | grep -v ".git" \
  | xargs wc -l 2>/dev/null | tail -1

# 讀取 CHANGELOG 或 release notes（了解演進歷史）
cat CHANGELOG.md 2>/dev/null | head -100
```

### 步驟 A3：深入分析核心程式碼

識別並閱讀關鍵檔案：

```bash
cd "$TMP_CODE_DIR"

# 找入口點（entry points）
find . -name "main.py" -o -name "main.go" -o -name "index.ts" -o -name "index.js" \
       -o -name "app.py" -o -name "server.py" -o -name "main.rs" \
  | grep -v node_modules | grep -v ".git" | head -5

# 找核心模組（排除 test/docs）
find . -name "*.py" -o -name "*.go" -o -name "*.ts" \
  | grep -v test | grep -v spec | grep -v docs | grep -v node_modules | grep -v ".git" \
  | head -20

# 找 benchmark / perf 相關測試
find . -name "*bench*" -o -name "*benchmark*" -o -name "*perf*" \
  | grep -v ".git" | head -10

# 找架構設計文件
find . -name "ARCHITECTURE*" -o -name "DESIGN*" -o -name "RFC*" \
  -o -name "ADR*" -o -path "*/docs/architecture*" \
  | grep -v ".git" | head -10
```

逐一閱讀上面找到的重要檔案，理解程式邏輯。

### 步驟 A4：查詢 GitHub API 取得補充資訊

```bash
# 從 URL 解析 owner/repo
GITHUB_OWNER=$(echo "$REPO_URL" | sed 's|https://github.com/||' | cut -d'/' -f1)
GITHUB_REPO=$(echo "$REPO_URL" | sed 's|https://github.com/||' | cut -d'/' -f2)

# 取得 repo 元資料（stars, forks, description, topics）
curl -s "https://api.github.com/repos/$GITHUB_OWNER/$GITHUB_REPO" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
print('Stars:', d.get('stargazers_count'))
print('Forks:', d.get('forks_count'))
print('Language:', d.get('language'))
print('Description:', d.get('description'))
print('Topics:', d.get('topics'))
print('Created:', d.get('created_at'))
print('Updated:', d.get('updated_at'))
print('License:', d.get('license', {}).get('name') if d.get('license') else 'None')
"

# 取得最新 releases
curl -s "https://api.github.com/repos/$GITHUB_OWNER/$GITHUB_REPO/releases?per_page=3" \
  | python3 -c "
import sys, json
releases = json.load(sys.stdin)
for r in releases[:3]:
    print(r.get('tag_name'), '-', r.get('name'))
    print(r.get('body', '')[:300])
    print('---')
"

# 取得 issues / discussions 中的 benchmark 討論（選用）
curl -s "https://api.github.com/repos/$GITHUB_OWNER/$GITHUB_REPO/issues?labels=performance,benchmark&state=all&per_page=5" \
  | python3 -c "
import sys, json
issues = json.load(sys.stdin)
for i in issues[:5]:
    print('#', i.get('number'), i.get('title'))
"
```

### 步驟 A5：合成分析報告，填入筆記模板

根據以上收集的資訊，撰寫完整的程式碼分析筆記，格式如下：

```markdown
---
title: "{Repo 名稱} — 程式碼深度分析"
date: {repo 建立或最新 release 日期}
category: CodeAnalysis
tags:
  - #code-analysis
  - #{主要程式語言}
  - #{領域，如 ai/framework、tools/cli、web/backend}
source: "{GitHub Repo URL}"
source_type: code
author: "{GitHub Owner/Org}"
status: notes
links:
  - "[[相關筆記1]]"
  - "[[相關筆記2]]"
github_stars: {star 數}
github_language: {主要語言}
---

## 摘要（Summary）
一段說明：這個 repo 是什麼、解決什麼問題、為何重要。

## Why — 為什麼存在？
> 這個專案要解決的根本問題是什麼？現有方案的哪些痛點促使它被創造？

- **核心動機**：...
- **取代/改善什麼**：...
- **目標用戶**：...

## What — 是什麼？
> 這個專案的功能邊界與核心能力。

- **主要功能**：
  - 功能一
  - 功能二
- **不做什麼（Non-goals）**：...
- **技術棧（Tech Stack）**：{語言、框架、主要依賴}

## How — 如何運作？

> [!important] 本節必須包含至少 **2 種 ASCII 圖表**，用 code block 呈現，讓讀者不看程式碼也能快速理解系統全貌。

### 系統架構圖（System Architecture）

用 ASCII art 描繪模組邊界與依賴關係：

```
{範例格式 — 請依實際架構替換}

┌─────────────────────────────────────────┐
│                  CLI / API              │
└──────────────────┬──────────────────────┘
                   │
       ┌───────────▼───────────┐
       │      Core Engine      │
       │  ┌─────┐  ┌────────┐  │
       │  │ A   │→│   B    │  │
       │  └─────┘  └────────┘  │
       └───────────┬───────────┘
                   │
       ┌───────────▼───────────┐
       │      Storage / IO     │
       └───────────────────────┘
```

### 執行流程圖（Execution Flowchart）

用 ASCII flowchart 描述主要執行路徑與分支：

```
{範例格式}

 Start
   │
   ▼
[讀取輸入] ──失敗──► [回報錯誤] ──► End
   │成功
   ▼
[處理步驟 A]
   │
   ├─ 條件 X ──► [路徑 X]
   │                  │
   └─ 條件 Y ──► [路徑 Y]
                      │
                      ▼
                  [合併結果]
                      │
                      ▼
                    End
```

### 時序圖（Sequence Diagram）

描述元件間的呼叫順序（適用於涉及多元件互動的系統）：

```
{範例格式}

 Client        Core         External API
   │             │                │
   │──請求──────►│                │
   │             │──查詢──────────►│
   │             │◄──回應─────────│
   │◄──結果──────│                │
   │             │                │
```

### 關鍵設計決策（Key Design Decisions）

> [!note] 設計模式（Design Pattern）
> 說明使用的核心設計模式與原因。

1. **決策一**：{描述} — {原因}
2. **決策二**：{描述} — {原因}

### 資料流（Data Flow）
1. 步驟一
2. 步驟二
3. 步驟三

### 關鍵程式碼（Key Code Snippets）

{摘錄最能說明設計理念的程式碼片段，完整保留不省略}

## 架構師觀點（Architect's View）

### ✅ 優點（Strengths）

| 面向 | 評估 | 說明 |
|------|------|------|
| 可維護性（Maintainability） | ⭐⭐⭐⭐⭐ | ... |
| 可擴展性（Scalability） | ⭐⭐⭐⭐ | ... |
| 測試覆蓋（Test Coverage） | ⭐⭐⭐ | ... |
| 文件品質（Documentation） | ⭐⭐⭐⭐ | ... |
| 依賴管理（Dependency Management） | ⭐⭐⭐⭐ | ... |

> [!tip] 值得學習的設計
> 描述最值得借鏡的架構決策或程式碼品質。

### ⚠️ 缺點與風險（Weaknesses & Risks）

> [!warning] 已知缺陷
> 列出架構層面的問題或技術債（Technical Debt）。

- **問題一**：{描述} — 影響：{影響}
- **問題二**：{描述} — 影響：{影響}

### 🔮 改進建議（Improvement Suggestions）
1. 建議一
2. 建議二

## 效能基準（Benchmark）

> [!info] 資料來源
> 說明 benchmark 數據來源（官方文件、issue、第三方評測）。

| 場景 | 此專案 | 競品 A | 競品 B |
|------|--------|--------|--------|
| 操作一 | {數值} | {數值} | {數值} |
| 操作二 | {數值} | {數值} | {數值} |

{若無公開 benchmark 數據，說明效能特性與預期瓶頸}

## 快速上手（Quick Start）

```bash
{最小可執行範例}
```

## 我的心得（My Takeaways）
我從這個 codebase 學到什麼？哪些設計可以應用到自己的專案？

## 相關連結（Related）
- [[RELATED-NOTE-1]] — 連結理由
- [[RELATED-NOTE-2]] — 連結理由
- [[RELATED-NOTE-3]] — 連結理由

## References
- [GitHub Repo]({URL})
- [官方文件]({docs URL if available})
```

> [!important] 步驟 A5 結尾：將筆記存到 `/tmp/kb-article.md`，圖片（若有 repo banner/截圖）存到 `/tmp/kb-assets/`，然後繼續步驟 3。

---

## 🌐 語言規則（Language Rule）：台灣繁體中文 + 保留英文專有名詞

> [!important] 所有筆記內容一律以**台灣繁體中文**撰寫，遵守以下規則：

**規則一：中文翻譯 + 括號保留英文原文**

| ✅ 正確寫法 | ❌ 錯誤寫法 |
|-----------|-----------|
| 知識圖譜（Knowledge Graph） | 知識圖譜 |
| 回饋循環（feedback loop） | feedback loop |
| 分叉模式（fork mode） | fork mode |
| 提示工程（Prompt Engineering） | prompt engineering |
| 上下文視窗（Context Window） | context window |

**規則二：工具名稱、品牌名稱直接使用英文**（不翻譯）
- 工具 / 軟體：Obsidian、Claude、GitHub、yt-dlp、Whisper
- 程式語言 / 框架：Python、JavaScript、YAML、Markdown
- 縮寫廣為人知者：API、CLI、SDK、LLM、AI

**規則三：雙向連結（Wikilink）檔名維持英文全大寫**
`[[CLAUDE-MEMORY-ENGINE]]` — 檔名格式不變，可在正文以中文描述

**規則四：引用原文時保留原語言**，區塊引用或程式碼中的英文不需翻譯。

**規則五：程式碼範例完整保留，不得省略**
- 原文中的程式碼區塊（code block）必須**完整複製**，不得用註解（comment）或摘要取代
- 程式碼內容保持英文原文，不翻譯
- 若原文有多個範例，每個都要完整收錄

---

## 步驟 1：讀取文章內容

**優先**：用 agent-browser 透過 Chrome CDP（port 9222）讀取，可存取需要登入的頁面：

> [!important] 前提條件：Chrome 必須以 remote debugging mode 啟動。若 CDP 連線失敗，請告知用戶用以下指令重開 Chrome（macOS 範例）：
> ```bash
> /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
>   --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-debug
> ```
> Chrome 開啟後，用戶登入目標網站，再回來繼續執行。

```bash
agent-browser --cdp 9222 open "$ARGUMENTS"
agent-browser --cdp 9222 wait --load networkidle
agent-browser --cdp 9222 get text body
```

**退而求其次**（CDP 失敗時）：
```bash
defuddle parse "$ARGUMENTS" --md
```

### YouTube 影片特別處理

若 URL 為 YouTube，**同時執行**以下兩步取得影片元資訊與逐字稿：

**步驟 Y1：取得影片元資訊（標題、頻道、上傳日期、時長）**
```bash
yt-dlp --skip-download --print "%(title)s|%(uploader)s|%(upload_date)s|%(duration_string)s" "$ARGUMENTS"
```

**步驟 Y2：取得逐字稿（transcript）— 依序嘗試以下方法，成功即停止**

**方法一：youtube-transcript-api（推薦，最穩定）**
```bash
# 確認已安裝
pip3 install youtube-transcript-api -q 2>/dev/null || true

# 從 URL 取出 VIDEO_ID 並傳入 Python
VIDEO_ID=$(python3 -c "
import sys, re
url = '$ARGUMENTS'
m = re.search(r'(?:v=|youtu\.be/)([^&?/]+)', url)
print(m.group(1) if m else '')
")
echo "VIDEO_ID: $VIDEO_ID"

python3 -c "
from youtube_transcript_api import YouTubeTranscriptApi
video_id = '$VIDEO_ID'
api = YouTubeTranscriptApi()
transcripts = api.list(video_id)
for t in transcripts:
    print(f'Available: {t.language} ({t.language_code}) generated={t.is_generated}')
t = transcripts.find_transcript(['zh-TW', 'zh-Hant', 'zh-Hans', 'zh', 'en'])
data = t.fetch()
full_text = '\n'.join([s.text for s in data])
print(full_text)
"
```

> [!important] API 版本注意：`youtube-transcript-api` 新版（≥0.7）已移除 `YouTubeTranscriptApi.get_transcript()` 類別方法，必須用 `api = YouTubeTranscriptApi()` 實例化後呼叫 `api.list(video_id)`。

**方法二：yt-dlp 下載字幕**（youtube-transcript-api 失敗時）
```bash
# 先列出可用字幕語言代碼
yt-dlp --list-subs "$ARGUMENTS" 2>&1 | grep -E "zh|en"

# 使用正確的語言代碼（如 zh-TW、zh-Hant-zh-TW）下載
yt-dlp --write-auto-sub --sub-lang "zh-TW,zh-Hant-zh-TW,zh-Hans-zh-TW,en" \
  --convert-subs srt --skip-download -o "/tmp/kb-video" "$ARGUMENTS"

# 讀取下載的字幕
cat /tmp/kb-video.*.srt 2>/dev/null || cat /tmp/kb-video.*.vtt 2>/dev/null
```

**方法三：無字幕 — Whisper AI 自動轉錄**
```bash
yt-dlp -f "bestaudio" -o "/tmp/kb-audio.%(ext)s" "$ARGUMENTS"
whisper /tmp/kb-audio.* --language Chinese --output_format txt
```

**方法四：Google NotebookLM**（無程式碼）
前往 https://notebooklm.google.com/ → 新增 YouTube 來源 → 複製摘要輸出

---

## 步驟 2：下載文章圖片

讀取頁面後，擷取文章本文中的圖片（排除 icon、avatar、廣告等非內容圖片）：

```bash
agent-browser --cdp 9222 eval --stdin <<'EOF'
JSON.stringify(
  Array.from(document.querySelectorAll(
    'article img, .article img, [data-testid="post-content"] img, .pw-post-body-paragraph img, figure img'
  ))
  .filter(img => img.width > 200 && img.src && !img.src.includes('avatar') && !img.src.includes('icon'))
  .map(img => ({ src: img.src, alt: img.alt }))
)
EOF
```

下載圖片到暫存目錄：
```bash
mkdir -p /tmp/kb-assets/
curl -L -o /tmp/kb-assets/{圖片檔名} "{圖片URL}"
```

圖片檔名：保留原檔名；若無副檔名則加 `.jpg`。

---

## 步驟 3：決定分類、檔名與路徑

### 分類選擇

> [!important] 分類依據**內容主題**，而非來源格式。YouTube 影片、部落格文章、論文都應依內容歸入對應主題資料夾。若現有分類都不適合，**直接新增一個語意明確的分類資料夾**，不需詢問。

| 分類 | 使用時機 |
|------|---------|
| `AI/` | AI 框架、LLM、代理人（Agent）、提示工程（Prompt Engineering）、機器學習（ML） |
| `Career/` | 職涯發展、職場策略、利害關係人管理（Stakeholder Management）、升遷、求職 |
| `Productivity/` | 工作流程（Workflow）、個人系統、GTD、時間管理、習慣 |
| `DevTools/` | 開發者工具、CLI、SDK、編輯器 |
| `Finance/` | 投資、經濟學（Economics）、金融科技（Fintech） |
| `Design/` | UI/UX、產品設計、視覺思考（Visual Thinking） |
| `Science/` | 物理、生物、天文、普通科學 |
| `Research/` | 學術論文、技術深入探討 |
| `Books/` | 書籍摘要與重點整理 |
| `OpenSource/` | 開源專案（Open Source Project）與社群 |
| `Security/` | 資訊安全（Cybersecurity）、隱私（Privacy） |
| `CodeAnalysis/` | GitHub Repo 程式碼深度分析、架構評估、Benchmark |

若內容不符合上列任何分類，根據主題自行建立新資料夾，例如 `Leadership/`、`Marketing/`、`Health/` 等。

### 檔名規則

格式：`{YYYY-MM-DD}-{原文標題大寫加連字號}.md`

- **日期**：使用文章或影片的**發布日期**（不是今天的日期），從以下來源依序尋找：
  1. 文章頁面中明確標示的發布日期（如 "Mar 7, 2026"、"2026-03-07"）
  2. HTML 的 `<time>` 元素或 `datetime` 屬性
  3. 文章 URL 中的日期（如 `/2026/03/07/`）
  4. YouTube 影片的上傳日期
  5. 若完全找不到，才使用今天日期，並在 frontmatter 加上 `date_uncertain: true`

**用 agent-browser 擷取發布日期的標準指令**（在步驟 1 讀完頁面後執行）：
```bash
agent-browser --cdp 9222 eval \
  'document.querySelector("meta[property=\"article:published_time\"]")?.content
   || document.querySelector("time[datetime]")?.getAttribute("datetime")
   || document.querySelector("meta[name=\"date\"]")?.content'
```
- **標題**：只保留英數字和連字號，去掉特殊符號，全部大寫
- 例：`2026-03-07-CLAUDE-SKILL-EVAL-FRAMEWORK-3-SKILLS-ONE-AFTERNOON-REAL-DATA.md`

> [!warning] 日期來源必須是**原文發布日期**，不可使用執行此 skill 的當天日期

### 圖片存放路徑

`{分類}/assets/{YYYY-MM-DD}-{簡短標題}/`

例：`AI/assets/2026-03-07-SKILL-EVAL/`

---

## 步驟 4：撰寫 Obsidian Markdown 筆記

### 前置資料（Frontmatter）格式（Bases 相容）

```yaml
---
title: "筆記完整標題（台灣繁體中文）"
date: YYYY-MM-DD        # 原文或影片的發布日期，非執行 skill 的當天日期
category: AI
tags:
  - tag1
  - tag2
  - tag3
source: "https://original-url.com"
source_type: article   # article | video | paper | tool | book | podcast | code
author: "原作者姓名"
status: notes          # notes | reviewed | complete
links:
  - "[[RELATED-NOTE-1]]"
  - "[[RELATED-NOTE-2]]"
---
```

影片筆記（`source_type: video`）額外欄位：
```yaml
channel: "頻道名稱（Channel Name）"
duration: "00:00"
transcript_method: youtube-transcript-api   # youtube-transcript-api | yt-dlp | whisper | notebooklm | manual
```

### 正文結構

```markdown
## 摘要（Summary）
一段說明：這是什麼、為何重要。

## 關鍵洞察（Key Insights）
- 洞察一 — 參見 [[RELATED-CONCEPT]]
- 洞察二
- 洞察三

## 詳細內容（Details）
更深入的筆記、引用、程式碼片段、圖表。

> [!note] 關鍵術語（Key Term）
> 用 callout 定義重要術語。

> [!tip] 可執行建議（Actionable Tip）
> 用 tip callout 記錄實際可以做的事。

> [!warning] 注意事項（Watch Out）
> 用 warning callout 記錄陷阱、限制或風險。

## 我的心得（My Takeaways）
我學到了什麼，以及如何應用。

## 相關連結（Related）
- [[RELATED-NOTE-1]] — 連結理由簡述
- [[RELATED-NOTE-2]] — 連結理由簡述
- [[RELATED-NOTE-3]] — 連結理由簡述

## References
- [原文]({URL})
```

### 圖片引用格式

```markdown
![圖片中文說明](assets/{YYYY-MM-DD}-{簡短標題}/{圖片檔名})
```

> [!important] 步驟 4 結尾：撰寫完成後，將 markdown 存到固定路徑 `/tmp/kb-article.md`，供後續步驟使用。

### Callout 類型參考

| Callout | 用途 |
|---------|------|
| `[!note]` | 一般筆記、觀察 |
| `[!tip]` | 可執行建議 |
| `[!warning]` | 注意事項、風險 |
| `[!info]` | 背景資訊 |
| `[!example]` | 具體範例 |
| `[!quote]` | 值得保留的直接引用 |
| `[!important]` | 重要規則或必讀資訊 |
| `[!faq]-` | 折疊式 FAQ（加 `-` 預設折疊） |

### 雙向連結（Wikilink）策略

- 在 `## 相關連結` 至少連結 **3 篇相關筆記**
- 在正文中提及已有獨立筆記的概念時即時連結：`[[CONCEPT-NAME]]`
- 若相關筆記尚未存在，仍可先寫下——會顯示為待建立的紅色連結
- 使用 `[[Note Name|顯示文字]]` 當檔名不適合直接出現在句子中

**標籤（Tag）策略**：每篇筆記 3–5 個標籤，使用巢狀標籤提升精確度：
`#ai/llm`、`#tools/cli`、`#productivity/workflows`

---

## 步驟 5：Clone / 更新本地 Repo

GitHub 資訊：
- **Repo**：`swchen44/personal-knowledge-base-from-ai`
- **Token**：環境變數 `$GITHUB_PERSONAL_ACCESS_TOKEN`
- **本地路徑**：`{執行 skill 時的工作目錄}/personal-kb-repo`（即 `$PWD/personal-kb-repo`）

> [!important] 每次執行此步驟時，**無論如何都必須先 pull 到最新版本**，再進行任何檔案複製或修改。若 pull 失敗（如 conflict），必須先解決後再繼續。

```bash
REPO_URL="https://${GITHUB_PERSONAL_ACCESS_TOKEN}@github.com/swchen44/personal-knowledge-base-from-ai.git"
LOCAL_REPO="$PWD/personal-kb-repo"

if [ -d "$LOCAL_REPO/.git" ]; then
  # 已存在：強制更新至遠端最新版本，再繼續
  echo "正在同步最新版本..."
  git -C "$LOCAL_REPO" fetch origin
  git -C "$LOCAL_REPO" pull --rebase origin main
  echo "同步完成，目前為最新版本"
else
  # 首次執行：完整 clone
  echo "首次 clone..."
  git clone "$REPO_URL" "$LOCAL_REPO"
fi
```

---

## 步驟 6：複製圖片與文章到本地 Repo

將圖片和 Markdown 文章複製到正確的資料夾結構：

```bash
LOCAL_REPO="$PWD/personal-kb-repo"

# 建立圖片目錄並複製圖片
mkdir -p "$LOCAL_REPO/{分類}/assets/{YYYY-MM-DD}-{簡短標題}/"
cp /tmp/kb-assets/* "$LOCAL_REPO/{分類}/assets/{YYYY-MM-DD}-{簡短標題}/"

# 複製 Markdown 文章
cp /tmp/kb-article.md "$LOCAL_REPO/{分類}/{YYYY-MM-DD}-{ALL-CAPS-TITLE}.md"
```

---

## 步驟 7：更新根目錄 README.md

直接在本地編輯 README.md，加入新筆記到 Recent Notes 區塊：

```python
python3 << 'PYEOF'
import os
LOCAL_REPO = os.path.join(os.getcwd(), "personal-kb-repo")
readme_path = f"{LOCAL_REPO}/README.md"

with open(readme_path, 'r', encoding='utf-8') as f:
    content = f.read()

new_line = "- [{NOTE-TITLE}](./{CATEGORY}/{FILENAME}.md) — {一行繁體中文說明}\n"

if "## 📌 Recent Notes" in content:
    content = content.replace(
        "## 📌 Recent Notes\n",
        "## 📌 Recent Notes\n" + new_line
    )
else:
    content += "\n## 📌 Recent Notes\n" + new_line

with open(readme_path, 'w', encoding='utf-8') as f:
    f.write(content)

print("README.md updated")
PYEOF
```

---

## 步驟 8（原步驟 8 前）：Git Commit & Push

將所有變更一次 commit 並 push 回 GitHub：

```bash
LOCAL_REPO="$PWD/personal-kb-repo"

git -C "$LOCAL_REPO" add .
git -C "$LOCAL_REPO" commit -m "Add note: {文章標題}

- Article: {分類}/{YYYY-MM-DD}-{ALL-CAPS-TITLE}.md
- Images: {分類}/assets/{YYYY-MM-DD}-{簡短標題}/
- Updated README.md Recent Notes"
git -C "$LOCAL_REPO" push
```

push 成功後顯示 GitHub 連結：`https://github.com/swchen44/personal-knowledge-base-from-ai/blob/main/{分類}/{YYYY-MM-DD}-{ALL-CAPS-TITLE}.md`

---

## ✅ 完整品質檢查清單（Quality Checklist）

### 🌐 語言規則確認
- [ ] 正文全程使用**台灣繁體中文**
- [ ] 所有翻譯成中文的專有名詞均已加括號標示英文原文
- [ ] 工具名稱、品牌名稱、程式語言保持英文原文

### 📋 前置資料（Frontmatter）確認
- [ ] 包含 `title`（台灣繁體中文標題）
- [ ] 包含 `date`、`tags`（至少 3 個）、`source`、`source_type`、`author`、`status`、`links`
- [ ] 影片筆記額外包含 `channel`、`duration`、`transcript_method`

### 🔗 知識圖譜（Knowledge Graph）確認
- [ ] 至少 **3 個雙向連結（wikilinks）**（即使目標筆記尚未存在）
- [ ] 至少 **1 個 callout** 用於關鍵洞察、術語或建議
- [ ] 包含 `## 相關連結` 區塊，附簡短連結理由

### 🖼️ 圖片確認
- [ ] 所有文章圖片已下載並上傳到 `assets/` 子目錄
- [ ] Markdown 中以相對路徑引用圖片
- [ ] 圖片 alt text 使用中文說明

### 📁 檔案結構確認
- [ ] 檔名格式：`{YYYY-MM-DD}-{ALL-CAPS-WITH-HYPHENS}.md`
- [ ] 放置於正確的分類資料夾
- [ ] 根目錄 README.md 的 Recent Notes 區塊已更新

### 📺 影片筆記（source_type: video）額外確認
- [ ] 分類依**影片內容主題**決定，而非固定放 `Videos/`
- [ ] 已記錄取得逐字稿（transcript）的方法
- [ ] `channel`、`duration`、`transcript_method` 欄位已填寫

### 💻 程式碼分析筆記額外確認
- [ ] 包含 **Why / What / How** 三個主要區塊
- [ ] **How** 區塊包含至少 **2 種 ASCII 圖表**（架構圖、流程圖、時序圖擇二以上）
- [ ] ASCII 圖表使用 code block 包住，正確使用 `┌┐└┘│─►◄▼▲` 等字元
- [ ] **架構師觀點** 有評分表格（優點）與具體問題清單（缺點）
- [ ] **Benchmark** 區塊已填寫（即使只有定性描述）
- [ ] 有 `github_stars` 與 `github_language` frontmatter 欄位
- [ ] 有 **Quick Start** 可執行範例
- [ ] 分類放在 `CodeAnalysis/` 資料夾
- [ ] `source_type: code`

---

## 步驟 9：回報結果

完成後告知：
- ✅ 文章檔名與 GitHub 路徑
- ✅ 上傳的圖片數量與路徑
- ✅ GitHub 連結
- ⚠️ 若有任何失敗的步驟請說明原因
