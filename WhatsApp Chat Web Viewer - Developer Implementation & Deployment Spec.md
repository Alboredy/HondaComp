# Developer Specification: Static WhatsApp Chat Viewer (Android-Only Workflow)

## 1. Objective & Architecture
Build and deploy a zero-dependency, static web application that renders an exported Android WhatsApp chat (`chat.txt` + media attachments) as an interactive, mobile-first chat interface. 

* **Constraint:** All repository creation, file preparation, version control, and deployment steps must be executable strictly from an **Android device** without a desktop OS or build tools (no Node.js, Python, or FFmpeg).
* **Architecture:** Pure client-side HTML5, CSS3, and Vanilla JavaScript (`index.html`) fetching `chat.txt` and relative media assets directly from the root directory.

---

## 2. File Structure & Asset Preparation
1. **Export Source Data:**
   * In WhatsApp (Android): Open target chat $\rightarrow$ **Menu ($\vdots$)** $\rightarrow$ **More** $\rightarrow$ **Export chat** $\rightarrow$ **Include media**.
   * Extract the resulting `.zip` archive on the Android device.
2. **Normalize Filenames:**
   * Rename the exported transcript file (e.g., `WhatsApp Chat with [Name].txt`) to **`chat.txt`**.
   * Keep all media files (`.jpg`, `.png`, `.webp`, `.opus`, `.mp4`, `.pdf`) in the same root directory as `chat.txt` with their original WhatsApp filenames intact.
3. **Repository Size & Limit Validation:**
   * Ensure total repository size remains under **1 GB** (GitHub Pages soft limit).
   * Ensure no individual media file (e.g., long `.mp4` videos) exceeds **100 MB** (GitHub hard file size limit). Remove or compress oversized `.mp4` files prior to commit.

---

## 3. Application Requirements (`index.html`)
Create a single-file `index.html` in the root directory implementing the following specifications:

### A. Parser Engine
* Fetch `chat.txt` asynchronously via `fetch('chat.txt')`.
* Strip invisible Unicode directional markers (`\u200e`, `\u200f`) injected by WhatsApp.
* Parse standard Android timestamp formats (both 24-hour and 12-hour AM/PM):
  * **User Messages:** `^(\d{1,2}\/\d{1,2}\/\d{2,4},\s\d{1,2}:\d{2}(?::\d{2})?(?:\s?[APap][Mm])?)\s-\s([^:]+):\s(.*)$`
  * **System Notifications:** `^(\d{1,2}\/\d{1,2}\/\d{2,4},\s\d{1,2}:\d{2}(?::\d{2})?(?:\s?[APap][Mm])?)\s-\s(.*)$`
  * **Multi-line Messages:** Append lines that do not match timestamp regexes to the previous message object with a `\n` delimiter.

### B. Media Attachment Rendering
Detect Android attachment strings matching `^([^\s]+?\.(ext))\s*\(file attached\)(.*)$` and render inline elements using `encodeURI(filename)`:
* **Images (`.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`):** Render `<img>` with `loading="lazy"` and an interactive full-screen lightbox modal on click.
* **Voice Memos / Audio (`.opus`, `.ogg`, `.mp3`, `.m4a`):** Render native HTML5 `<audio controls preload="none">`.
* **Videos (`.mp4`, `.mov`):** Render `<video controls preload="metadata">`.
* **Documents (`.pdf`, `.docx`, `.xlsx`, etc.):** Render a downloadable anchor link (`target="_blank"`).
* **Captions:** Preserve and render any text appearing after `(file attached)` beneath the media element.

### C. UI/UX & Performance
* **WhatsApp Mobile Layout:** Centered container (`max-width: 650px`), sticky top header (`#075e54`), incoming bubbles (`#ffffff` left-aligned), outgoing bubbles (`#d9fdd3` right-aligned), and system badges (`#ffeecd` centered).
* **Perspective Switcher:** Dynamically populate a `<select>` dropdown with all unique senders parsed from `chat.txt`, allowing the viewer to toggle which participant appears on the right-hand ("sent") side.
* **Real-Time Search:** Filter rendered messages by keyword or sender name.
* **DOM Performance:** Batch-insert message nodes using `DocumentFragment` to prevent UI thread blocking on large chat histories.

---

## 4. Reference Implementation (`index.html`)
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>WhatsApp Chat Archive</title>
  <style>
    :root {
      --bg-color: #efeae2;
      --header-bg: #075e54;
      --sent-bg: #d9fdd3;
      --recv-bg: #ffffff;
      --text-main: #111b21;
      --text-muted: #667781;
      --sys-bg: #ffeecd;
    }
    * { box-sizing: border-box; margin: 0; padding: 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; }
    body { background-color: #111b21; display: flex; justify-content: center; height: 100vh; }
    .app { width: 100%; max-width: 650px; background-color: var(--bg-color); display: flex; flex-direction: column; height: 100vh; position: relative; box-shadow: 0 4px 20px rgba(0,0,0,0.4); }
    header { background: var(--header-bg); color: white; padding: 12px 16px; display: flex; flex-direction: column; gap: 8px; z-index: 10; box-shadow: 0 1px 3px rgba(0,0,0,0.2); }
    .header-top { display: flex; justify-content: space-between; align-items: center; }
    .header-top h1 { font-size: 1.1rem; font-weight: 600; }
    .controls { display: flex; gap: 8px; flex-wrap: wrap; }
    select, input[type="text"] { padding: 6px 10px; border-radius: 6px; border: none; font-size: 0.85rem; outline: none; }
    select { background: #054c44; color: white; max-width: 160px; }
    input[type="text"] { flex: 1; min-width: 140px; }
    #chat-box { flex: 1; overflow-y: auto; padding: 14px 12px; display: flex; flex-direction: column; gap: 6px; background-image: radial-gradient(#d5cfc7 1px, transparent 1px); background-size: 20px 20px; }
    .msg { max-width: 82%; padding: 7px 10px 6px; border-radius: 8px; position: relative; font-size: 0.94rem; line-height: 1.4; color: var(--text-main); box-shadow: 0 1px 1px rgba(0,0,0,0.12); word-wrap: break-word; }
    .msg.sent { align-self: flex-end; background: var(--sent-bg); border-top-right-radius: 0; }
    .msg.recv { align-self: flex-start; background: var(--recv-bg); border-top-left-radius: 0; }
    .msg.sys { align-self: center; background: var(--sys-bg); color: #54656f; font-size: 0.78rem; text-align: center; border-radius: 8px; max-width: 90%; padding: 5px 12px; margin: 4px 0; }
    .sender { font-size: 0.78rem; font-weight: 700; margin-bottom: 3px; color: #027eb5; }
    .sent .sender { display: none; }
    .text { white-space: pre-wrap; }
    .time { font-size: 0.68rem; color: var(--text-muted); text-align: right; margin-top: 4px; }
    .media-img, .media-video { max-width: 100%; max-height: 320px; border-radius: 6px; margin-top: 4px; display: block; cursor: pointer; }
    audio { width: 240px; max-width: 100%; height: 38px; margin-top: 4px; }
    .file-link { display: inline-block; background: rgba(0,0,0,0.06); padding: 8px 12px; border-radius: 6px; text-decoration: none; color: #027eb5; font-weight: 500; margin-top: 4px; font-size: 0.85rem; }
    #status { text-align: center; padding: 20px; color: var(--text-muted); font-size: 0.9rem; }
    #lightbox { display: none; position: fixed; inset: 0; background: rgba(0,0,0,0.9); z-index: 100; justify-content: center; align-items: center; }
    #lightbox img { max-width: 95%; max-height: 95%; object-fit: contain; }
  </style>
</head>
<body>
  <div class="app">
    <header>
      <div class="header-top">
        <h1>Chat Archive</h1>
        <span id="msg-count" style="font-size: 0.78rem; opacity: 0.85;">Loading...</span>
      </div>
      <div class="controls">
        <select id="me-select" title="Choose your perspective">
          <option value="">View as: Everyone Left</option>
        </select>
        <input type="text" id="search-input" placeholder="Search messages..." />
      </div>
    </header>
    <div id="chat-box"><div id="status">Loading chat.txt...</div></div>
  </div>
  <div id="lightbox" onclick="this.style.display='none'"><img id="lightbox-img" src="" alt="Full view" /></div>
  <script>
    const chatBox = document.getElementById('chat-box');
    const meSelect = document.getElementById('me-select');
    const searchInput = document.getElementById('search-input');
    const msgCount = document.getElementById('msg-count');
    let parsedMessages = [];
    let currentMe = "";

    const msgRegex = /^(\d{1,2}\/\d{1,2}\/\d{2,4},\s\d{1,2}:\d{2}(?::\d{2})?(?:\s?[APap][Mm])?)\s-\s([^:]+):\s(.*)$/;
    const sysRegex = /^(\d{1,2}\/\d{1,2}\/\d{2,4},\s\d{1,2}:\d{2}(?::\d{2})?(?:\s?[APap][Mm])?)\s-\s(.*)$/;

    async function init() {
      try {
        const res = await fetch('chat.txt');
        if (!res.ok) throw new Error('HTTP ' + res.status);
        const text = await res.text();
        parseChat(text);
        populateSenders();
        renderMessages();
      } catch (err) {
        document.getElementById('status').innerHTML = 
          `Error loading <b>chat.txt</b>.<br><br>Ensure the transcript is named <code>chat.txt</code> in the root directory and served over HTTP/HTTPS.`;
      }
    }

    function parseChat(rawText) {
      const lines = rawText.split(/\r?\n/);
      const senders = new Set();
      lines.forEach(line => {
        const cleanLine = line.replace(/[\u200e\u200f]/g, '');
        const match = cleanLine.match(msgRegex);
        if (match) {
          const [, time, sender, content] = match;
          senders.add(sender.trim());
          parsedMessages.push({ type: 'msg', time, sender: sender.trim(), content });
        } else if (cleanLine.match(sysRegex)) {
          const [, time, content] = cleanLine.match(sysRegex);
          parsedMessages.push({ type: 'sys', time, content });
        } else if (parsedMessages.length > 0 && cleanLine.trim() !== '') {
          parsedMessages[parsedMessages.length - 1].content += '\n' + cleanLine;
        }
      });
      window._senders = Array.from(senders);
    }

    function populateSenders() {
      window._senders.forEach(sender => {
        const opt = document.createElement('option');
        opt.value = sender;
        opt.textContent = `Right side: ${sender}`;
        meSelect.appendChild(opt);
      });
      if (window._senders.length > 0) {
        currentMe = window._senders[0];
        meSelect.value = currentMe;
      }
    }

    function formatContent(text) {
      const attachMatch = text.match(/^([^\s]+?\.(jpg|jpeg|png|webp|gif|opus|ogg|mp3|m4a|mp4|mov|pdf|docx|xlsx))\s*\(file attached\)(.*)$/is);
      if (attachMatch) {
        const fileName = attachMatch[1];
        const ext = attachMatch[2].toLowerCase();
        const caption = attachMatch[3] ? `<div class="text" style="margin-top:6px;">${escapeHtml(attachMatch[3].trim())}</div>` : '';
        if (['jpg', 'jpeg', 'png', 'webp', 'gif'].includes(ext)) {
          return `<img class="media-img" src="${encodeURI(fileName)}" loading="lazy" onclick="openLightbox(this.src)" alt="Photo" />${caption}`;
        }
        if (['opus', 'ogg', 'mp3', 'm4a'].includes(ext)) {
          return `<audio controls preload="none" src="${encodeURI(fileName)}"></audio>${caption}`;
        }
        if (['mp4', 'mov'].includes(ext)) {
          return `<video class="media-video" controls preload="metadata" src="${encodeURI(fileName)}"></video>${caption}`;
        }
        return `<a class="file-link" href="${encodeURI(fileName)}" target="_blank">📄 ${escapeHtml(fileName)}</a>${caption}`;
      }
      return `<div class="text">${escapeHtml(text)}</div>`;
    }

    function escapeHtml(str) {
      return str.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;");
    }

    function renderMessages(filter = "") {
      chatBox.innerHTML = "";
      const query = filter.toLowerCase();
      let count = 0;
      const frag = document.createDocumentFragment();
      parsedMessages.forEach(item => {
        if (query && !item.content.toLowerCase().includes(query) && !(item.sender && item.sender.toLowerCase().includes(query))) return;
        count++;
        const div = document.createElement('div');
        if (item.type === 'sys') {
          div.className = 'msg sys';
          div.textContent = item.content;
        } else {
          const isMe = item.sender === currentMe;
          div.className = `msg ${isMe ? 'sent' : 'recv'}`;
          div.innerHTML = `<div class="sender">${escapeHtml(item.sender)}</div>${formatContent(item.content)}<div class="time">${item.time}</div>`;
        }
        frag.appendChild(div);
      });
      chatBox.appendChild(frag);
      msgCount.textContent = `${count} messages`;
    }

    function openLightbox(src) {
      document.getElementById('lightbox-img').src = src;
      document.getElementById('lightbox').style.display = 'flex';
    }

    meSelect.addEventListener('change', (e) => { currentMe = e.target.value; renderMessages(searchInput.value); });
    searchInput.addEventListener('input', (e) => { renderMessages(e.target.value); });
    init();
  </script>
</body>
</html>
```

---

## 5. Android Git Workflow & Deployment

### Step A: Create Repository & Personal Access Token (Mobile Browser)
1. Navigate to `https://github.com/new` in Android Chrome and create a new repository (e.g., `whatsapp-archive`).
2. Navigate to **Settings** $\rightarrow$ **Developer settings** $\rightarrow$ **Personal access tokens** $\rightarrow$ **Tokens (classic)** $\rightarrow$ **Generate new token** with `repo` scope enabled. Copy the token.

### Step B: Commit & Push via Termux (Android)
1. Install **Termux** (from F-Droid or GitHub Releases) and run:
   ```bash
   termux-setup-storage
   pkg update && pkg install git -y
   ```
2. Navigate to the unzipped chat directory containing `index.html`, `chat.txt`, and all media files:
   ```bash
   cd ~/storage/downloads/YourExtractedChatFolder
   ```
3. Initialize Git, commit all assets, and push to GitHub (use the Personal Access Token as the password when prompted):
   ```bash
   git init
   git config user.name "Developer"
   git config user.email "dev@example.com"
   git add .
   git commit -m "Initial commit: WhatsApp chat archive"
   git branch -M main
   git remote add origin https://github.com/YOUR_USERNAME/whatsapp-archive.git
   git push -u origin main
   ```

### Step C: Hosting & Access Control Configuration
* **Option 1: Public Hosting (GitHub Pages)**
  1. Open the repository on `github.com` in Android Chrome.
  2. Go to **Settings** $\rightarrow$ **Pages**.
  3. Under **Build and deployment** $\rightarrow$ **Source**, select **Deploy from a branch**, choose `main` and `/ (root)`, and tap **Save**.
  4. Distribute the resulting `https://YOUR_USERNAME.github.io/whatsapp-archive/` URL.
* **Option 2: Private / Restricted Hosting (Cloudflare Pages + Access)**
  1. Keep the GitHub repository set to **Private**.
  2. Log into `dash.cloudflare.com` on Android Chrome $\rightarrow$ **Workers & Pages** $\rightarrow$ **Create** $\rightarrow$ **Pages** $\rightarrow$ **Connect to Git**.
  3. Select the private repository and deploy with default settings (no build command, output directory `/`).
  4. Enable **Cloudflare Access** on the deployed domain to require email OTP verification for authorized viewers.

---

## 6. Known Platform Caveats
* **Local `file://` Testing:** Opening `index.html` directly from Android local storage will fail to load `chat.txt` due to browser CORS restrictions on `fetch()`. Verify locally using a lightweight HTTP server (e.g., Acode's built-in preview server or `python -m http.server` in Termux) or directly on the deployed staging URL.
* **iOS Safari `.opus` Playback:** Android WhatsApp exports voice notes in `.opus` format. Chrome, Firefox, and Edge play `.opus` natively, whereas iOS Safari users may need to open the link in Chrome for iOS if audio playback does not initialize.