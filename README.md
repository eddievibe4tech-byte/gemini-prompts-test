```markdown
# ⚡ Gemini Prompt Auto-Pipeline (Android 14+ / ColorOS)

專為 Android 裝置設計的極簡自動化管線。透過 GitHub Actions 雲端自動將模組化的 Few-Shot 提示詞編譯為單一 Master Prompt，並由手機端的開源工具 **HTTP Shortcuts** 接收分享文字或 YouTube 網址，0.2 秒無縫喚醒手機版 **Gemini App** 並自動填入完整分析指令。

---

## 📁 倉庫目錄結構

```text
.
├── .github/
│   └── workflows/
│       └── build-prompt.yml       # 自動組裝並發布的 CI 流程
├── 00-system.md                   # 核心人設、{{INPUT_CONTENT}} 錨點與任務規範
├── few-shots/                     # 獨立情境範例庫（依檔名順序自動拼接）
│   ├── 01-habit.md                # 模組 A：習慣拖延與自律心理
│   ├── 02-business.md             # 模組 B：避開網紅洗腦與割韭菜
│   ├── 03-geopolitics.md          # 模組 C：時事利益角力比喻
│   └── 04-english.md              # 模組 D：道地美語火鍋沾醬法
├── 99-framework.md                # 三層分析邏輯架構結尾規範
├── dist/
│   └── master_prompt.txt          # [自動生成] 手機 HTTP Shortcuts 抓取的最終產物
└── README.md

```

---

## 🛠️ 第一階段：GitHub 倉庫設定

### 1. 啟用 Actions 寫入權限（關鍵）

為讓 GitHub Actions 能自動將組裝好的 `dist/master_prompt.txt` commit 回倉庫：

1. 進入倉庫頁面，點選 **Settings** $\rightarrow$ **Actions** $\rightarrow$ **General**。
2. 滑到底部找到 **Workflow permissions**。
3. 勾選 **Read and write permissions** 並點擊 **Save**。

### 2. 取得 Raw 檔案網址

當第一次 push 觸發 Action 生成 `dist/master_prompt.txt` 後：

1. 點進該檔案，點擊右上角的 **Raw** 按鈕。
2. 複製網址列的 Raw URL（格式如下）：
```text
[https://raw.githubusercontent.com/](https://raw.githubusercontent.com/)<你的 GitHub 帳號>/<倉庫名稱>/main/dist/master_prompt.txt

```


*(若為私人倉庫，後續請在 HTTP Shortcuts 加上 Authorization Header)*

---

## 📱 第二階段：手機端 HTTP Shortcuts 設定

### 1. 建立捷徑

1. 開啟 **HTTP Shortcuts** App，點擊右下角「＋」新增。
2. 類型選擇：**HTTP 請求（HTTP Request）**。
3. **基本設定**：
* **捷徑名稱**：`⚡ 送給 Gemini 分析`
* **方法（Method）**：`GET`
* **URL**：貼上剛剛複製的 `dist/master_prompt.txt` Raw 網址。



### 2. 註冊系統分享面板

點進 **「觸發與執行設定（Trigger & Execution Settings）」**：

* 勾選 **「在啟動器上顯示為應用程式捷徑」**。
* 勾選 **「允許從分享對話方塊接收檔案」**（或接收文字）。

### 3. 設定無感背景切換

點進 **「回應與顯示（Response Handling）」**：

* **回應顯示方式（Response display）**：改選為 **「不顯示（None）」**。

### 4. 配置處理腳本（Scripting）

* **點進「執行後腳本（Success Script / Response Script）」貼上：**

```javascript
// 1. 抓取分享文字或剪貼簿備援
let sharedText = getSharedText() || getClipboard();
if (!sharedText) {
    showToast("⚠️ 未偵測到輸入內容！");
    abort();
}

// 2. 取得 GitHub 下載的最新組裝 Prompt
let masterTemplate = response.body;

// 3. 快取機制（斷網時自動回退）
if (masterTemplate && masterTemplate.length > 50) {
    setGlobalVariable("CACHED_MASTER_PROMPT", masterTemplate);
} else {
    masterTemplate = getGlobalVariable("CACHED_MASTER_PROMPT");
}

if (!masterTemplate) {
    showToast("❌ 首次使用需連網同步 GitHub 模板！");
    abort();
}

// 4. 將內文注入模板錨點
let payload = masterTemplate.replace("{{INPUT_CONTENT}}", sharedText);

// 5. 喚醒手機版 Gemini App 並填入對話框
sendIntent({
    target: "activity",
    action: "android.intent.action.SEND",
    type: "text/plain",
    package: "com.google.android.apps.bard",
    extra: {
        "android.intent.extra.TEXT": payload
    },
    flags: [268435456] // FLAG_ACTIVITY_NEW_TASK
});

```

* **點進「執行失敗腳本（Failure Script）」貼上（離線保險）：**

```javascript
// 斷網或無收訊時，自動調用本地快取直通 Gemini
let sharedText = getSharedText() || getClipboard();
let cachedPrompt = getGlobalVariable("CACHED_MASTER_PROMPT");

if (cachedPrompt && sharedText) {
    let payload = cachedPrompt.replace("{{INPUT_CONTENT}}", sharedText);
    sendIntent({
        target: "activity",
        action: "android.intent.action.SEND",
        type: "text/plain",
        package: "com.google.android.apps.bard",
        extra: {
            "android.intent.extra.TEXT": payload
        },
        flags: [268435456]
    });
} else {
    showToast("⚠️ 連線失敗且無本地快取，請先連網同步一次！");
}

```

---

## 🚀 日常使用流程

1. **反白文章**：在任何網頁或 App 反白選取長文，點選選單中的「分享」$\rightarrow$「⚡ 送給 Gemini 分析」。
2. **YouTube 影片**：在 YouTube App 點選「分享」$\rightarrow$「⚡ 送給 Gemini 分析」。
3. **成果**：系統約 0.2 秒自動切換至 **Gemini App** 前景，輸入框已填滿完整的 Prompt 與分析內容，直接點擊送出即可。

---

## 🔄 日常如何增修 Few-Shot？

* 在手機打開 **GitHub 官方 App** 或在 **Termux** 中進到 `few-shots/` 目錄。
* 直接新增如 `05-crypto.md` 或修改現有範例，Commit 並 push 到 `main` 分支。
* GitHub Actions 會在數秒內自動合併組裝。手機端下一次分享時，無需做任何操作，便會自動同步並套用最新版提示詞！

```

<FollowUp>
<Suggestion>若 GitHub 倉庫設為 Private，HTTP Shortcuts 該如何配置 Personal Access Token？</Suggestion>
<Suggestion>想在分享 YouTube 網址時自動附上「請抓取這部影片的逐字稿重點」該如何擴充判斷？</Suggestion>
<Suggestion>如何將此捷徑加入反白選取文字時的浮動選單（Process Text），實現免按分享直達？</Suggestion>
</FollowUp>

```
