# knowledge-from-ai-summary — Claude Skills Marketplace

A Claude Code plugin marketplace for personal knowledge base skills.

## Installation

### Step 1: Add marketplace

```
/plugin marketplace add swchen44/knowledge-from-ai-summary
```

### Step 2: Install skill

```
/plugin install article-to-personal-kb@knowledge-from-ai-summary
```

## Skills

| Skill | Description |
|-------|-------------|
| [article-to-personal-kb](skills/article-to-personal-kb) | 讀取網頁文章（含需登入的頁面）或 YouTube 影片，翻譯成台灣繁體中文 Obsidian markdown 格式，連同圖片一起上傳到 GitHub 個人知識庫。輸入一個 URL，自動完成全流程。 |

## Usage

After installing, use the skill with:

```
/article-to-personal-kb <url>
```

Supports:
- General web articles (including paywalled/login-required pages via Chrome CDP)
- YouTube videos (transcript extraction + summary)
- GitHub repositories (code analysis)

## License

MIT
