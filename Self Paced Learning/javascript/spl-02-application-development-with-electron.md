---
title: spl-02-application-development-with-electron
description: Course on building portable desktop applications with Electron and JavaScript
published: true
date: 2026-03-07T00:00:00.000Z
tags: javascript, electron, desktop, learning
editor: markdown
dateCreated: 2026-03-07T00:00:00.000Z
---

# Module 2: Application Development with Electron

## 📋 Course Overview

This module builds on Module 1 (JavaScript and Node.js) to teach **desktop application development with Electron**. Electron lets you write cross-platform desktop apps using the same web technologies you already know — HTML, CSS, and JavaScript — while giving full access to the operating system through Node.js.

Electron is the technology behind VS Code, Slack, Discord, Figma, and many other production-grade desktop applications. The same app you build here runs unmodified on macOS, Windows, and Linux.

### What You'll Learn

- How Electron's two-process architecture works (main process and renderer process)
- Setting up an Electron project from scratch
- Creating and managing application windows
- Preload scripts and the Context Bridge security model
- Inter-Process Communication (IPC) — the backbone of every Electron app
- Native OS integrations: file dialogs, the shell, Tray icons, and notifications
- Building application menus and context menus
- Structuring a real, maintainable Electron application
- Packaging and distributing your app as a standalone executable

### Prerequisites

- **Module 1 completed** — especially:
  - `async`/`await` and Promises
  - Classes and `this` binding
  - CommonJS modules and `__dirname`
  - EventEmitter (`on`, `emit`) pattern
  - Destructuring in function parameters

### The Application You'll Build

Throughout this module you will incrementally build a **Cluster Monitor** desktop application — a system dashboard that displays live CPU, memory, and uptime data from your local machine and can save snapshots to disk. This app demonstrates every major Electron concept in a practical context.

---

## ⚙️ Part 1: Environment Setup

### 1.1 Install Prerequisites

You need Node.js (v18 LTS or later) and npm. If you completed Module 1, these are already installed.

```bash
node --version   # should be v18.x.x or later
npm --version    # should be 9.x.x or later
```

### 1.2 Create the Project

```bash
mkdir cluster-monitor && cd cluster-monitor
npm init -y
```

Install Electron as a dev dependency. It is a large download (~300 MB) that bundles both Chromium and Node.js.

```bash
npm install --save-dev electron
```

Add a start script to `package.json`:

```json
{
  "name": "cluster-monitor",
  "version": "1.0.0",
  "description": "Homelab cluster monitor desktop app",
  "main": "src/main.js",
  "scripts": {
    "start": "electron ."
  },
  "devDependencies": {
    "electron": "^29.0.0"
  }
}
```

> **Why `"main": "src/main.js"`?** Electron reads `package.json` to find the entry point for the main process. This path must be relative to the project root.

### 1.3 Project Structure

We will build up to this structure over the course of the module:

```
cluster-monitor/
├── package.json
└── src/
    ├── main.js              ← Main process entry point
    ├── preload.js           ← Preload script (bridge between main and renderer)
    └── renderer/
        ├── index.html       ← Renderer process HTML
        ├── renderer.js      ← Renderer process JavaScript
        └── styles.css       ← Styles
```

Create the directories now:

```bash
mkdir -p src/renderer
```

### 1.4 Verify the Installation

Create a minimal `src/main.js` to confirm everything is working:

```javascript
// src/main.js
const { app, BrowserWindow } = require("electron");
const path = require("path");

function createWindow() {
  const win = new BrowserWindow({ width: 800, height: 600 });
  win.loadFile(path.join(__dirname, "renderer/index.html"));
}

app.whenReady().then(createWindow);
```

Create a minimal `src/renderer/index.html`:

```html
<!DOCTYPE html>
<html>
<head><title>Cluster Monitor</title></head>
<body>
  <h1>Electron is working!</h1>
</body>
</html>
```

Run it:

```bash
npm start
```

A native desktop window should open. Close it with the OS window close button.

---

## ✅ Part 2: Electron Architecture

Understanding Electron's two-process model is the most important conceptual step. Everything else follows from it.

### 2.1 The Two-Process Model

Every Electron app runs as **two distinct processes** that communicate over a message-passing channel:

```
┌─────────────────────────────────────────────────────────────┐
│  MAIN PROCESS (Node.js)                                      │
│  src/main.js                                                 │
│                                                              │
│  • Full Node.js API (fs, os, net, child_process, ...)       │
│  • Creates and manages BrowserWindow instances              │
│  • Handles OS-level events (app lifecycle, menus, trays)    │
│  • Listens for IPC messages from renderers                  │
│  • One per application                                       │
└───────────────────────┬─────────────────────────────────────┘
                        │  IPC (Inter-Process Communication)
                        │  ipcMain / ipcRenderer
                        │
┌───────────────────────┴─────────────────────────────────────┐
│  RENDERER PROCESS (Chromium)                                 │
│  src/renderer/index.html + renderer.js                      │
│                                                              │
│  • Standard browser environment (DOM, fetch, CSS, ...)      │
│  • NO direct Node.js access (by design — security)         │
│  • One per BrowserWindow (multiple windows = multiple        │
│    renderer processes)                                       │
│  • Communicates with main via contextBridge (preload.js)    │
└─────────────────────────────────────────────────────────────┘
```

**The key insight:** The renderer is a sandboxed browser. It cannot call `require("fs")` or `require("os")`. Any operation that needs Node.js (reading files, calling native APIs, accessing the network at a low level) must be done in the **main process** and the result sent back to the renderer via IPC.

### 2.2 Why This Matters

This separation exists for security. Web content is untrusted by default — if the renderer could call `require("child_process")`, a malicious web page loaded in your app could take over the host machine. The main process acts as a gatekeeper, exposing only the specific operations it explicitly allows through the preload script.

```
Renderer (untrusted)  →→  preload.js (bridge)  →→  Main (trusted)
                           contextBridge              Node.js APIs
                           ipcRenderer.invoke         ipcMain.handle
```

### 2.3 The Three Files You Always Write

| File | Process | Role |
|------|---------|------|
| `src/main.js` | Main | App lifecycle, window management, native APIs, IPC handlers |
| `src/preload.js` | Neither (injected) | Exposes a safe, limited API to the renderer |
| `src/renderer/renderer.js` | Renderer | DOM manipulation, UI logic, calls preload API |

The preload script is injected by Electron into the renderer before any renderer JavaScript runs. It has access to **both** the Node.js `require` function AND the browser's `window` object, making it the only place where the two worlds can safely meet.

---

## ✅ Part 3: The Main Process

`src/main.js` is where your application starts. It is a regular Node.js script with access to Electron's main-process APIs.

### 3.1 The App Lifecycle

The `app` object is an EventEmitter that fires events as the application moves through its lifecycle.

```javascript
// src/main.js
const { app, BrowserWindow } = require("electron");
const path = require("path");

// Called when Electron has finished initialisation and is ready to create windows.
// Some APIs can only be used after this event fires.
app.whenReady().then(() => {
  createWindow();

  // macOS: re-create the window when the dock icon is clicked and no windows are open
  app.on("activate", () => {
    if (BrowserWindow.getAllWindows().length === 0) {
      createWindow();
    }
  });
});

// Quit when all windows are closed (except on macOS, where apps stay open until Cmd+Q)
app.on("window-all-closed", () => {
  if (process.platform !== "darwin") {
    app.quit();
  }
});
```

### 3.2 BrowserWindow

`BrowserWindow` creates a native OS window that hosts a Chromium renderer process.

```javascript
const { BrowserWindow } = require("electron");
const path = require("path");

function createWindow() {
  const win = new BrowserWindow({
    width:  1000,
    height: 700,
    minWidth: 600,
    minHeight: 400,

    // Renderer security settings — always set these
    webPreferences: {
      preload: path.join(__dirname, "preload.js"), // path to your preload script
      contextIsolation: true,   // ✅ required — isolates renderer from Node.js world
      nodeIntegration:  false,  // ✅ required — renderer cannot call require()
      sandbox:          true,   // ✅ recommended — additional process sandboxing
    },

    // Window chrome
    titleBarStyle: "hiddenInset", // macOS: traffic-light buttons inside content
    // frame: false,              // completely frameless window (you draw your own chrome)

    // Show the window only after it has loaded its content to avoid a white flash
    show: false,
  });

  // Load the HTML entry point
  win.loadFile(path.join(__dirname, "renderer/index.html"));

  // Show the window once the DOM is ready
  win.once("ready-to-show", () => win.show());

  // Open DevTools during development
  if (process.env.NODE_ENV === "development") {
    win.webContents.openDevTools();
  }

  return win;
}
```

#### BrowserWindow Events

```javascript
win.on("close", (event) => {
  // event.preventDefault() stops the window from closing — useful for "unsaved changes" dialogs
});

win.on("closed", () => {
  // Window has been closed and the BrowserWindow object is destroyed
  // Dereference any held references to this window here
});

win.on("focus",  () => { /* window gained focus */ });
win.on("blur",   () => { /* window lost focus  */ });
win.on("resize", () => {
  const [width, height] = win.getSize();
  console.log(`Resized to ${width}x${height}`);
});
```

#### Multiple Windows

```javascript
let mainWindow = null;
let settingsWindow = null;

function openSettingsWindow() {
  if (settingsWindow) {
    settingsWindow.focus(); // bring existing window to front
    return;
  }

  settingsWindow = new BrowserWindow({
    width: 500,
    height: 400,
    parent: mainWindow,  // modal relationship on macOS/Windows
    modal: true,
    webPreferences: {
      preload: path.join(__dirname, "preload.js"),
      contextIsolation: true,
      nodeIntegration: false,
    },
  });

  settingsWindow.loadFile(path.join(__dirname, "renderer/settings.html"));

  // Clean up the reference when the window is closed
  settingsWindow.on("closed", () => {
    settingsWindow = null;
  });
}
```

### 3.3 Main Process Boilerplate

Here is a complete, production-ready starting template for `src/main.js`:

```javascript
// src/main.js
"use strict";

const { app, BrowserWindow, ipcMain } = require("electron");
const path = require("path");
const os   = require("os");
const fs   = require("fs/promises");

// ─── State ────────────────────────────────────────────────────────────────────
let mainWindow = null;

// ─── Window Factory ──────────────────────────────────────────────────────────
function createWindow() {
  mainWindow = new BrowserWindow({
    width:  1000,
    height: 700,
    show:   false,
    webPreferences: {
      preload:          path.join(__dirname, "preload.js"),
      contextIsolation: true,
      nodeIntegration:  false,
      sandbox:          true,
    },
  });

  mainWindow.loadFile(path.join(__dirname, "renderer/index.html"));
  mainWindow.once("ready-to-show", () => mainWindow.show());
  mainWindow.on("closed", () => { mainWindow = null; });
}

// ─── IPC Handlers (registered before window creation) ────────────────────────
ipcMain.handle("get-system-info", async () => {
  return {
    hostname: os.hostname(),
    platform: os.platform(),
    arch:     os.arch(),
    cpus:     os.cpus().length,
    memory: {
      total: os.totalmem(),
      free:  os.freemem(),
    },
    uptime: os.uptime(),
  };
});

// ─── App Lifecycle ────────────────────────────────────────────────────────────
app.whenReady().then(() => {
  createWindow();

  app.on("activate", () => {
    if (BrowserWindow.getAllWindows().length === 0) createWindow();
  });
});

app.on("window-all-closed", () => {
  if (process.platform !== "darwin") app.quit();
});

// ─── Global error handling ────────────────────────────────────────────────────
process.on("uncaughtException",    (err) => console.error("Uncaught:", err));
process.on("unhandledRejection",   (err) => console.error("Unhandled rejection:", err));
```

---

## ✅ Part 4: Preload Scripts and the Context Bridge

The preload script is the most important security boundary in modern Electron. **Every API you want the renderer to use must be explicitly declared here.**

### 4.1 What a Preload Script Does

The preload script runs in a **privileged context** — it has access to Node.js APIs, but also to the browser's `window` object. The `contextBridge` module lets you safely expose a controlled API surface to the renderer.

```javascript
// src/preload.js
"use strict";

const { contextBridge, ipcRenderer } = require("electron");

// contextBridge.exposeInMainWorld(apiName, api) injects an object into
// window.apiName in the renderer. The renderer cannot access anything
// from Node.js EXCEPT what you explicitly expose here.
contextBridge.exposeInMainWorld("electronAPI", {

  // One-way: renderer → main (fire and forget)
  sendMessage: (channel, data) => ipcRenderer.send(channel, data),

  // Two-way: renderer asks main for data, main replies
  getSystemInfo: () => ipcRenderer.invoke("get-system-info"),

  // One-way: main → renderer (main pushes updates to renderer)
  onUpdate: (callback) => {
    // ipcRenderer.on listener — wrap in a clean function to avoid
    // exposing the full `event` object to the renderer
    const handler = (_event, data) => callback(data);
    ipcRenderer.on("live-update", handler);

    // Return a cleanup function the renderer can call to unsubscribe
    return () => ipcRenderer.removeListener("live-update", handler);
  },

});
```

After this, in the renderer you call `window.electronAPI.getSystemInfo()` — just like any other browser API. The renderer has zero knowledge that Node.js is involved.

### 4.2 What You Can and Cannot Expose

```javascript
// ✅ SAFE — expose specific, intentional operations
contextBridge.exposeInMainWorld("electronAPI", {
  readConfig:  ()       => ipcRenderer.invoke("read-config"),
  saveConfig:  (config) => ipcRenderer.invoke("save-config", config),
  openFile:    ()       => ipcRenderer.invoke("open-file-dialog"),
  onLogLine:   (cb)     => {
    const h = (_, line) => cb(line);
    ipcRenderer.on("log-line", h);
    return () => ipcRenderer.removeListener("log-line", h);
  },
});

// ❌ DANGEROUS — never do these
contextBridge.exposeInMainWorld("dangerousAPI", {
  // Exposing require gives the renderer access to ALL of Node.js
  require: require,

  // Exposing ipcRenderer directly gives the renderer unrestricted IPC access
  ipcRenderer: ipcRenderer,

  // Exposing exec lets the renderer run arbitrary shell commands
  exec: require("child_process").exec,
});
```

### 4.3 Validating Channel Names

When using `ipcRenderer.send` or `.invoke`, always validate channel names in the main process handler. A renderer should only be able to trigger operations you intended.

```javascript
// src/preload.js — whitelist allowed channels
const VALID_CHANNELS = ["get-system-info", "read-config", "save-config", "open-file"];

contextBridge.exposeInMainWorld("electronAPI", {
  invoke: (channel, ...args) => {
    if (!VALID_CHANNELS.includes(channel)) {
      throw new Error(`Blocked IPC call to unknown channel: ${channel}`);
    }
    return ipcRenderer.invoke(channel, ...args);
  },
});
```

### 4.4 The TypeScript Pattern (for reference)

Even if you write plain JavaScript, it is worth knowing the TypeScript preload pattern because you will see it in documentation:

```typescript
// In TypeScript projects, declare the API shape in a .d.ts file
// so the renderer gets full type-checking:
//
// interface Window {
//   electronAPI: {
//     getSystemInfo: () => Promise<SystemInfo>;
//     openFile:      () => Promise<string | null>;
//   }
// }
```

---

## ✅ Part 5: IPC — Inter-Process Communication

IPC is how the main process and renderer process talk to each other. There are two main patterns: **invoke/handle** (request-response) and **send/on** (one-way).

### 5.1 Invoke / Handle — Request-Response

This is the most common pattern. The renderer asks for data; the main process responds. Think of it as an RPC call over a private channel.

```javascript
// ─── Main process: register a handler ────────────────────────────────────────
// src/main.js
const { ipcMain } = require("electron");
const os          = require("os");
const fs          = require("fs/promises");
const path        = require("path");

// ipcMain.handle(channel, asyncHandler)
// The handler receives (event, ...args) and can return a value or throw.
// Returning a value sends it back to the renderer as the resolved Promise value.
// Throwing sends it back as a rejected Promise.

ipcMain.handle("get-system-info", async (event) => {
  // event.sender is the WebContents of the renderer that called invoke()
  // You can check the sender URL for additional security:
  // if (!event.sender.getURL().startsWith("file://")) throw new Error("Untrusted sender");

  return {
    hostname: os.hostname(),
    platform: os.platform(),
    cpus:     os.cpus().length,
    memory: {
      total: os.totalmem(),
      free:  os.freemem(),
    },
    uptime: os.uptime(),
  };
});

ipcMain.handle("read-file", async (event, { filePath, encoding = "utf8" }) => {
  // Parameter destructuring — exactly the pattern from Module 1
  const absPath = path.resolve(filePath);
  return fs.readFile(absPath, encoding);
});

ipcMain.handle("write-file", async (event, { filePath, content }) => {
  await fs.mkdir(path.dirname(filePath), { recursive: true });
  await fs.writeFile(filePath, content, "utf8");
  return { success: true };
});
```

```javascript
// ─── Preload: expose the handler ─────────────────────────────────────────────
// src/preload.js
const { contextBridge, ipcRenderer } = require("electron");

contextBridge.exposeInMainWorld("electronAPI", {
  getSystemInfo: ()               => ipcRenderer.invoke("get-system-info"),
  readFile:      (filePath)       => ipcRenderer.invoke("read-file", { filePath }),
  writeFile:     (filePath, content) => ipcRenderer.invoke("write-file", { filePath, content }),
});
```

```javascript
// ─── Renderer: call the exposed API ──────────────────────────────────────────
// src/renderer/renderer.js
async function loadSystemInfo() {
  try {
    // window.electronAPI is injected by preload.js
    const info = await window.electronAPI.getSystemInfo();

    document.getElementById("hostname").textContent = info.hostname;
    document.getElementById("cpu-count").textContent = `${info.cpus} CPUs`;

    const usedMemGB = ((info.memory.total - info.memory.free) / 1e9).toFixed(1);
    const totalMemGB = (info.memory.total / 1e9).toFixed(1);
    document.getElementById("memory").textContent = `${usedMemGB} / ${totalMemGB} GB`;

    const uptimeDays = Math.floor(info.uptime / 86400);
    const uptimeHrs  = Math.floor((info.uptime % 86400) / 3600);
    document.getElementById("uptime").textContent = `${uptimeDays}d ${uptimeHrs}h`;
  } catch (err) {
    console.error("Failed to load system info:", err);
  }
}

// Call on startup
loadSystemInfo();
```

### 5.2 Send / On — One-Way (Fire and Forget)

Use `ipcRenderer.send` when the renderer notifies the main process of an action and doesn't need a response.

```javascript
// src/preload.js — expose one-way send
contextBridge.exposeInMainWorld("electronAPI", {
  logAction: (action) => ipcRenderer.send("log-action", { action, ts: Date.now() }),
});

// src/main.js — listen for one-way messages
const { ipcMain } = require("electron");
ipcMain.on("log-action", (event, { action, ts }) => {
  console.log(`[${new Date(ts).toISOString()}] Action: ${action}`);
});
```

### 5.3 Main → Renderer Push Updates

The main process can push data to the renderer at any time using `webContents.send()`. This is perfect for live updates (polling, file watching, etc.).

```javascript
// src/main.js — push live updates from main to renderer
const { BrowserWindow } = require("electron");

function startLiveUpdates(win) {
  setInterval(() => {
    const snapshot = {
      ts:        Date.now(),
      freeMem:   os.freemem(),
      totalMem:  os.totalmem(),
      uptime:    os.uptime(),
    };

    // Send to the specific window's renderer process
    if (win && !win.isDestroyed()) {
      win.webContents.send("live-update", snapshot);
    }
  }, 2000); // push every 2 seconds
}

// Call this after creating the window
app.whenReady().then(() => {
  createWindow();
  startLiveUpdates(mainWindow);
});
```

```javascript
// src/preload.js — expose the listener
contextBridge.exposeInMainWorld("electronAPI", {
  onLiveUpdate: (callback) => {
    const handler = (_event, data) => callback(data);
    ipcRenderer.on("live-update", handler);
    // Return a cleanup function
    return () => ipcRenderer.removeListener("live-update", handler);
  },
});
```

```javascript
// src/renderer/renderer.js — subscribe to live updates
const unsubscribe = window.electronAPI.onLiveUpdate((snapshot) => {
  const usedMB = Math.round((snapshot.totalMem - snapshot.freeMem) / 1e6);
  document.getElementById("live-mem").textContent = `${usedMB} MB used`;
});

// Clean up when the page unloads
window.addEventListener("beforeunload", unsubscribe);
```

### 5.4 Concurrent IPC Calls

Just as in Node.js, you can fire multiple IPC calls concurrently with `Promise.all`:

```javascript
// src/renderer/renderer.js
async function loadDashboard() {
  // Fire both requests simultaneously instead of waiting for each in sequence
  const [systemInfo, config] = await Promise.all([
    window.electronAPI.getSystemInfo(),
    window.electronAPI.readConfig(),
  ]);

  renderSystemInfo(systemInfo);
  applyConfig(config);
}
```

### 5.5 IPC Patterns — Summary

| Pattern | Direction | API | Use Case |
|---------|-----------|-----|----------|
| Request/response | Renderer → Main | `ipcRenderer.invoke` + `ipcMain.handle` | Get data, trigger operations with a result |
| Fire and forget | Renderer → Main | `ipcRenderer.send` + `ipcMain.on` | Log events, notify without caring about result |
| Push update | Main → Renderer | `webContents.send` + `ipcRenderer.on` | Real-time data (timers, file watches, network events) |
| Broadcast | Main → All windows | `BrowserWindow.getAllWindows().forEach(w => w.webContents.send(...))` | App-wide state changes |

---

## ✅ Part 6: Native OS Integrations

Electron exposes native OS features through a set of modules available only in the main process. These are registered as IPC handlers so the renderer can trigger them.

### 6.1 File Dialogs

```javascript
// src/main.js
const { dialog } = require("electron");
const fs = require("fs/promises");

// Open file dialog — returns the chosen file path(s) or undefined if cancelled
ipcMain.handle("open-file-dialog", async (event, options = {}) => {
  const result = await dialog.showOpenDialog(mainWindow, {
    title:       options.title ?? "Open File",
    buttonLabel: "Open",
    filters: options.filters ?? [
      { name: "All Files", extensions: ["*"] },
    ],
    properties: ["openFile"],  // or "openDirectory", "multiSelections"
  });

  if (result.canceled) return null;
  return result.filePaths[0];   // single file path as a string
});

// Save file dialog
ipcMain.handle("save-file-dialog", async (event, options = {}) => {
  const result = await dialog.showSaveDialog(mainWindow, {
    title:          options.title ?? "Save File",
    defaultPath:    options.defaultPath ?? "output.json",
    buttonLabel:    "Save",
    filters: options.filters ?? [
      { name: "JSON Files", extensions: ["json"] },
      { name: "All Files",  extensions: ["*"]    },
    ],
  });

  if (result.canceled) return null;
  return result.filePath;
});

// Message box (alert / confirm)
ipcMain.handle("show-message-box", async (event, options) => {
  const result = await dialog.showMessageBox(mainWindow, {
    type:     options.type    ?? "info",     // "none", "info", "error", "question", "warning"
    title:    options.title   ?? "Message",
    message:  options.message ?? "",
    detail:   options.detail  ?? "",
    buttons:  options.buttons ?? ["OK"],
    defaultId: 0,
    cancelId:  1,
  });
  return result.response;  // index of the clicked button
});
```

```javascript
// src/preload.js
contextBridge.exposeInMainWorld("electronAPI", {
  openFileDialog:   (opts)          => ipcRenderer.invoke("open-file-dialog", opts),
  saveFileDialog:   (opts)          => ipcRenderer.invoke("save-file-dialog", opts),
  showMessageBox:   (opts)          => ipcRenderer.invoke("show-message-box", opts),
});
```

```javascript
// src/renderer/renderer.js
async function saveSnapshot() {
  const snapshot = await window.electronAPI.getSystemInfo();

  const filePath = await window.electronAPI.saveFileDialog({
    title:       "Save Snapshot",
    defaultPath: `snapshot-${Date.now()}.json`,
    filters:     [{ name: "JSON", extensions: ["json"] }],
  });

  if (!filePath) return;  // user cancelled

  await window.electronAPI.writeFile(filePath, JSON.stringify(snapshot, null, 2));

  await window.electronAPI.showMessageBox({
    type:    "info",
    title:   "Saved",
    message: "Snapshot saved successfully.",
    detail:  filePath,
  });
}
```

### 6.2 The Shell Module

`shell` provides OS-level file and URL operations that open in the user's default application.

```javascript
// src/main.js
const { shell } = require("electron");

// Open a URL in the default browser
ipcMain.handle("open-external", async (event, url) => {
  // Validate the URL before opening it — never pass unchecked renderer input to shell
  const parsed = new URL(url);
  if (!["http:", "https:"].includes(parsed.protocol)) {
    throw new Error(`Blocked: unsafe protocol ${parsed.protocol}`);
  }
  await shell.openExternal(url);
});

// Reveal a file in Finder/Explorer
ipcMain.handle("show-in-folder", async (event, filePath) => {
  shell.showItemInFolder(filePath);
});

// Open a file in its default application
ipcMain.handle("open-path", async (event, filePath) => {
  const err = await shell.openPath(filePath);
  if (err) throw new Error(err);
});
```

### 6.3 The Tray Icon

A tray icon lives in the system taskbar/menu bar and is useful for background applications or quick access.

```javascript
// src/main.js
const { Tray, Menu, nativeImage } = require("electron");
const path = require("path");

let tray = null;   // keep a reference — if garbage collected, the tray disappears

function createTray() {
  // On macOS, template images (black on transparent) are recommended
  // On Windows, use a 16×16 or 32×32 .ico file
  const icon = nativeImage.createFromPath(
    path.join(__dirname, "../assets/tray-icon.png")
  );

  tray = new Tray(icon);
  tray.setToolTip("Cluster Monitor");

  const contextMenu = Menu.buildFromTemplate([
    { label: "Show Window", click: () => mainWindow?.show() },
    { type: "separator" },
    {
      label: "Take Snapshot",
      click: async () => {
        const info = { .../* gather info */ };
        // Trigger the snapshot save flow via the main window
        mainWindow?.webContents.send("trigger-snapshot");
      },
    },
    { type: "separator" },
    { label: "Quit",        click: () => app.quit() },
  ]);

  tray.setContextMenu(contextMenu);

  // Single-click to show window (Windows/Linux)
  tray.on("click", () => {
    if (mainWindow) {
      mainWindow.isVisible() ? mainWindow.hide() : mainWindow.show();
    }
  });
}
```

### 6.4 System Notifications

```javascript
// src/main.js
const { Notification } = require("electron");

ipcMain.handle("show-notification", (event, { title, body }) => {
  if (Notification.isSupported()) {
    new Notification({ title, body }).show();
  }
});
```

---

## ✅ Part 7: Application Menus and Context Menus

### 7.1 The Application Menu

The application menu is the native menu bar at the top of the screen (macOS) or the window (Windows/Linux).

```javascript
// src/main.js
const { Menu } = require("electron");

function buildMenu() {
  const isMac = process.platform === "darwin";

  const template = [
    // macOS: first menu item is always the app name menu
    ...(isMac ? [{
      label: app.name,
      submenu: [
        { role: "about" },
        { type: "separator" },
        { role: "services" },
        { type: "separator" },
        { role: "hide" },
        { role: "hideOthers" },
        { role: "unhide" },
        { type: "separator" },
        { role: "quit" },
      ],
    }] : []),

    // File menu
    {
      label: "File",
      submenu: [
        {
          label:       "Save Snapshot…",
          accelerator: "CmdOrCtrl+S",
          click:       () => mainWindow?.webContents.send("trigger-snapshot"),
        },
        { type: "separator" },
        isMac ? { role: "close" } : { role: "quit" },
      ],
    },

    // View menu
    {
      label: "View",
      submenu: [
        { role: "reload" },
        { role: "forceReload" },
        { role: "toggleDevTools" },
        { type: "separator" },
        { role: "resetZoom" },
        { role: "zoomIn" },
        { role: "zoomOut" },
        { type: "separator" },
        { role: "togglefullscreen" },
      ],
    },

    // Custom menu
    {
      label: "Monitor",
      submenu: [
        {
          label:   "Refresh Now",
          accelerator: "CmdOrCtrl+R",
          click:   () => mainWindow?.webContents.send("refresh-data"),
        },
        {
          label:   "Auto-Refresh",
          type:    "checkbox",
          checked: true,
          click:   (menuItem) => {
            mainWindow?.webContents.send("set-auto-refresh", menuItem.checked);
          },
        },
      ],
    },

    // Window menu
    {
      label: "Window",
      submenu: [
        { role: "minimize" },
        { role: "zoom" },
        ...(isMac ? [
          { type: "separator" },
          { role: "front" },
        ] : [
          { role: "close" },
        ]),
      ],
    },
  ];

  return Menu.buildFromTemplate(template);
}

// Apply the menu after the app is ready
app.whenReady().then(() => {
  Menu.setApplicationMenu(buildMenu());
  createWindow();
});
```

### 7.2 Context Menus (Right-Click)

Context menus are triggered by the renderer and built in the main process.

```javascript
// src/main.js — handle context menu request
const { Menu, MenuItem } = require("electron");

ipcMain.on("show-context-menu", (event, params) => {
  const menu = Menu.buildFromTemplate([
    {
      label: `Copy "${params.label}"`,
      click: () => event.sender.send("context-menu-action", {
        action: "copy",
        value: params.value,
      }),
    },
    { type: "separator" },
    {
      label: "Save Snapshot",
      click: () => event.sender.send("context-menu-action", {
        action: "snapshot",
      }),
    },
  ]);

  // Show the menu anchored to the BrowserWindow
  const win = BrowserWindow.fromWebContents(event.sender);
  menu.popup({ window: win });
});
```

```javascript
// src/preload.js
contextBridge.exposeInMainWorld("electronAPI", {
  showContextMenu:      (params) => ipcRenderer.send("show-context-menu", params),
  onContextMenuAction:  (cb) => {
    const h = (_, data) => cb(data);
    ipcRenderer.on("context-menu-action", h);
    return () => ipcRenderer.removeListener("context-menu-action", h);
  },
});
```

```javascript
// src/renderer/renderer.js — register right-click on a stat card
document.getElementById("memory-card").addEventListener("contextmenu", (e) => {
  e.preventDefault();
  window.electronAPI.showContextMenu({
    label: "Memory",
    value: document.getElementById("live-mem").textContent,
  });
});

window.electronAPI.onContextMenuAction(({ action, value }) => {
  if (action === "copy") {
    navigator.clipboard.writeText(value);
  }
});
```

---

## ✅ Part 8: The Renderer Process

The renderer is a standard Chromium browser context. You write plain HTML, CSS, and JavaScript — no special Electron APIs needed here, except through `window.electronAPI`.

### 8.1 HTML Entry Point

```html
<!-- src/renderer/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <!--
    Content Security Policy: restricts what can execute in the renderer.
    This policy allows:
      - Scripts from this origin only (no inline scripts, no CDN scripts)
      - Styles from this origin and inline styles
      - No external connections from the renderer (all network goes through main)
  -->
  <meta http-equiv="Content-Security-Policy"
        content="default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Cluster Monitor</title>
  <link rel="stylesheet" href="styles.css" />
</head>
<body>

  <header class="app-header">
    <h1>Cluster Monitor</h1>
    <span id="last-updated">–</span>
  </header>

  <main class="dashboard">

    <section class="card" id="system-card">
      <h2>System</h2>
      <dl>
        <dt>Hostname</dt> <dd id="hostname">–</dd>
        <dt>Platform</dt> <dd id="platform">–</dd>
        <dt>CPUs</dt>     <dd id="cpu-count">–</dd>
        <dt>Uptime</dt>   <dd id="uptime">–</dd>
      </dl>
    </section>

    <section class="card" id="memory-card">
      <h2>Memory</h2>
      <p id="live-mem" class="stat-large">–</p>
      <div class="progress-bar">
        <div id="mem-bar" class="progress-fill"></div>
      </div>
    </section>

    <section class="card" id="actions-card">
      <h2>Actions</h2>
      <button id="btn-refresh">Refresh Now</button>
      <button id="btn-snapshot">Save Snapshot</button>
    </section>

  </main>

  <!--
    Script tag at the bottom of body — DOM is fully built by the time this runs.
    Note: no type="module" here unless you configure the CSP and bundler to allow it.
  -->
  <script src="renderer.js"></script>
</body>
</html>
```

### 8.2 Renderer JavaScript

```javascript
// src/renderer/renderer.js
"use strict";

// ─── Utilities ────────────────────────────────────────────────────────────────
function formatBytes(bytes) {
  const gb = bytes / 1e9;
  return gb >= 1 ? `${gb.toFixed(1)} GB` : `${(bytes / 1e6).toFixed(0)} MB`;
}

function formatUptime(seconds) {
  const d = Math.floor(seconds / 86400);
  const h = Math.floor((seconds % 86400) / 3600);
  const m = Math.floor((seconds % 3600)  / 60);
  return `${d}d ${h}h ${m}m`;
}

// ─── DOM References ───────────────────────────────────────────────────────────
const $ = (id) => document.getElementById(id);

// ─── Render Functions ─────────────────────────────────────────────────────────
function renderSystemInfo(info) {
  $("hostname").textContent  = info.hostname;
  $("platform").textContent  = `${info.platform} / ${info.arch}`;
  $("cpu-count").textContent = `${info.cpus} logical CPUs`;
  $("uptime").textContent    = formatUptime(info.uptime);
  $("last-updated").textContent = new Date().toLocaleTimeString();
}

function renderMemory(memInfo) {
  const used   = memInfo.total - memInfo.free;
  const pct    = Math.round((used / memInfo.total) * 100);

  $("live-mem").textContent   = `${formatBytes(used)} / ${formatBytes(memInfo.total)}`;
  $("mem-bar").style.width    = `${pct}%`;
  $("mem-bar").style.background = pct > 85 ? "#e74c3c" : pct > 65 ? "#f39c12" : "#2ecc71";
}

// ─── Data Loading ─────────────────────────────────────────────────────────────
async function refresh() {
  try {
    const info = await window.electronAPI.getSystemInfo();
    renderSystemInfo(info);
    renderMemory(info.memory);
  } catch (err) {
    console.error("Refresh failed:", err);
  }
}

// ─── Live Updates ─────────────────────────────────────────────────────────────
const unsubscribe = window.electronAPI.onLiveUpdate((snapshot) => {
  renderMemory({ total: snapshot.totalMem, free: snapshot.freeMem });
  $("uptime").textContent = formatUptime(snapshot.uptime);
  $("last-updated").textContent = new Date(snapshot.ts).toLocaleTimeString();
});

// ─── Button Handlers ──────────────────────────────────────────────────────────
$("btn-refresh").addEventListener("click", refresh);

$("btn-snapshot").addEventListener("click", async () => {
  const info = await window.electronAPI.getSystemInfo();

  const filePath = await window.electronAPI.saveFileDialog({
    title:       "Save System Snapshot",
    defaultPath: `snapshot-${new Date().toISOString().slice(0, 10)}.json`,
    filters:     [{ name: "JSON Files", extensions: ["json"] }],
  });

  if (!filePath) return;

  await window.electronAPI.writeFile(
    filePath,
    JSON.stringify({ timestamp: new Date().toISOString(), ...info }, null, 2)
  );

  await window.electronAPI.showMessageBox({
    type:    "info",
    title:   "Snapshot Saved",
    message: "System snapshot written successfully.",
    detail:  filePath,
  });
});

// ─── Menu-Triggered Actions ───────────────────────────────────────────────────
// These are sent by the main process menu items
window.electronAPI.onMainMessage?.("refresh-data",    () => refresh());
window.electronAPI.onMainMessage?.("trigger-snapshot", () => $("btn-snapshot").click());

// ─── Startup ──────────────────────────────────────────────────────────────────
refresh();

window.addEventListener("beforeunload", unsubscribe);
```

### 8.3 Styles

```css
/* src/renderer/styles.css */
:root {
  --bg:       #1e1e2e;
  --surface:  #313244;
  --text:     #cdd6f4;
  --muted:    #6c7086;
  --accent:   #89b4fa;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}

* { box-sizing: border-box; margin: 0; padding: 0; }

body {
  background: var(--bg);
  color:      var(--text);
  height:     100vh;
  display:    flex;
  flex-direction: column;
}

.app-header {
  display:         flex;
  justify-content: space-between;
  align-items:     center;
  padding:         1rem 1.5rem;
  border-bottom:   1px solid var(--surface);
}

.app-header h1 { font-size: 1.25rem; color: var(--accent); }
#last-updated  { font-size: 0.8rem;  color: var(--muted);  }

.dashboard {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap:     1rem;
  padding: 1.5rem;
  flex:    1;
}

.card {
  background:    var(--surface);
  border-radius: 8px;
  padding:       1.25rem;
}

.card h2 {
  font-size:     0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color:         var(--muted);
  margin-bottom: 1rem;
}

dl { display: grid; grid-template-columns: auto 1fr; gap: 0.4rem 1rem; }
dt { color: var(--muted); font-size: 0.85rem; }
dd { font-size: 0.85rem; }

.stat-large { font-size: 1.5rem; font-weight: 600; margin-bottom: 0.75rem; }

.progress-bar  { background: var(--bg); border-radius: 4px; height: 8px; overflow: hidden; }
.progress-fill { height: 100%; border-radius: 4px; transition: width 0.5s ease, background 0.5s ease; }

button {
  display:       block;
  width:         100%;
  padding:       0.6rem 1rem;
  margin-bottom: 0.5rem;
  background:    var(--accent);
  color:         var(--bg);
  border:        none;
  border-radius: 6px;
  font-weight:   600;
  cursor:        pointer;
  transition:    opacity 0.2s;
}
button:hover  { opacity: 0.85; }
button:active { opacity: 0.7;  }
```

---

## ✅ Part 9: App Configuration and Persistence

Most desktop applications need to persist user settings between sessions. Electron provides `app.getPath()` to find appropriate OS-specific directories.

### 9.1 App Paths

```javascript
// src/main.js
const { app } = require("electron");
const path = require("path");

// app.getPath() returns OS-appropriate directories
console.log(app.getPath("userData"));
// macOS:   ~/Library/Application Support/<app-name>
// Windows: C:\Users\<user>\AppData\Roaming\<app-name>
// Linux:   ~/.config/<app-name>

console.log(app.getPath("documents")); // User's Documents folder
console.log(app.getPath("downloads")); // User's Downloads folder
console.log(app.getPath("temp"));      // OS temp directory
console.log(app.getPath("logs"));      // App log directory

// Build a path to a settings file inside userData
const CONFIG_PATH = path.join(app.getPath("userData"), "settings.json");
```

### 9.2 A Simple Config Service

```javascript
// src/config.js — a reusable config module for the main process
"use strict";

const { app } = require("electron");
const fs      = require("fs/promises");
const path    = require("path");

const CONFIG_PATH = path.join(app.getPath("userData"), "settings.json");

const DEFAULTS = {
  refreshInterval: 2000,
  autoRefresh:     true,
  theme:           "dark",
  windowBounds:    { width: 1000, height: 700, x: null, y: null },
};

let _cache = null;

async function load() {
  if (_cache) return _cache;

  try {
    const raw  = await fs.readFile(CONFIG_PATH, "utf8");
    _cache = { ...DEFAULTS, ...JSON.parse(raw) };
  } catch {
    // File doesn't exist yet — use defaults
    _cache = { ...DEFAULTS };
  }

  return _cache;
}

async function save(updates) {
  const current = await load();
  _cache = { ...current, ...updates };
  await fs.mkdir(path.dirname(CONFIG_PATH), { recursive: true });
  await fs.writeFile(CONFIG_PATH, JSON.stringify(_cache, null, 2), "utf8");
  return _cache;
}

async function get(key) {
  const config = await load();
  return config[key];
}

module.exports = { load, save, get, CONFIG_PATH };
```

```javascript
// src/main.js — register IPC handlers for config
const config = require("./config");

ipcMain.handle("read-config",  async ()          => config.load());
ipcMain.handle("save-config",  async (_, updates) => config.save(updates));

// Save window position/size on close for next launch
mainWindow.on("close", async () => {
  const bounds = mainWindow.getBounds();
  await config.save({ windowBounds: bounds });
});

// Restore window position/size on create
app.whenReady().then(async () => {
  const cfg = await config.load();
  mainWindow = new BrowserWindow({
    ...cfg.windowBounds,
    webPreferences: { /* ... */ },
  });
});
```

### 9.3 Watching for File Changes

Use Node's `fs.watch` from the main process to push file-change events to the renderer:

```javascript
// src/main.js — watch a directory and push change events
const fs = require("fs");

function watchDirectory(win, dirPath) {
  const watcher = fs.watch(dirPath, { recursive: false }, (eventType, filename) => {
    if (filename && !win.isDestroyed()) {
      win.webContents.send("file-changed", { eventType, filename, dirPath });
    }
  });

  // Clean up the watcher when the window closes
  win.on("closed", () => watcher.close());
}
```

---

## ✅ Part 10: Packaging and Distribution

Packaging turns your source code into a standalone executable that runs on any machine without Node.js or npm installed.

### 10.1 electron-builder

`electron-builder` is the most widely used tool for packaging Electron apps.

```bash
npm install --save-dev electron-builder
```

Add a build configuration to `package.json`:

```json
{
  "name": "cluster-monitor",
  "version": "1.0.0",
  "description": "Homelab cluster monitor",
  "main": "src/main.js",
  "scripts": {
    "start":     "electron .",
    "build":     "electron-builder",
    "build:mac": "electron-builder --mac",
    "build:win": "electron-builder --win",
    "build:linux": "electron-builder --linux"
  },
  "build": {
    "appId": "com.homelab.cluster-monitor",
    "productName": "Cluster Monitor",
    "copyright": "Copyright © 2026",

    "directories": {
      "output": "dist"
    },

    "files": [
      "src/**",
      "assets/**",
      "!**/*.map",
      "!**/node_modules/**"
    ],

    "mac": {
      "category": "public.app-category.utilities",
      "icon": "assets/icon.icns",
      "target": [
        { "target": "dmg", "arch": ["x64", "arm64"] },
        { "target": "zip", "arch": ["x64", "arm64"] }
      ]
    },

    "win": {
      "icon": "assets/icon.ico",
      "target": [
        { "target": "nsis",     "arch": ["x64"] },
        { "target": "portable", "arch": ["x64"] }
      ]
    },

    "linux": {
      "icon": "assets/icon.png",
      "category": "Utility",
      "target": [
        { "target": "AppImage", "arch": ["x64"] },
        { "target": "deb",      "arch": ["x64"] }
      ]
    }
  }
}
```

Build for the current platform:

```bash
npm run build
```

Output is placed in `dist/`. On macOS this produces a `.dmg` and a `.zip`; on Linux it produces an `.AppImage` and a `.deb`.

### 10.2 Code Signing

Without code signing, users on macOS will see a Gatekeeper warning. On Windows, SmartScreen will flag the installer. For a homelab tool these are acceptable; for public distribution, obtain:

- **macOS**: Apple Developer Program membership (~$99/year), then sign with `codesign` and notarize with `notarytool`.
- **Windows**: An EV Code Signing Certificate from a trusted CA.

electron-builder integrates with both — see its documentation for the `afterSign` hook.

### 10.3 Auto-Updates

`electron-updater` (part of electron-builder) handles automatic updates:

```bash
npm install electron-updater
```

```javascript
// src/main.js
const { autoUpdater } = require("electron-updater");

autoUpdater.on("update-available",    ()        => mainWindow?.webContents.send("update-available"));
autoUpdater.on("update-downloaded",   ()        => mainWindow?.webContents.send("update-downloaded"));
autoUpdater.on("error",               (err)     => console.error("AutoUpdater error:", err));

app.whenReady().then(() => {
  createWindow();
  // Check for updates after the window is shown
  setTimeout(() => autoUpdater.checkForUpdatesAndNotify(), 3000);
});
```

---

## ✅ Part 11: Security Checklist

Electron applications have full operating system access. Follow these rules on every project.

| Setting | Value | Reason |
|---------|-------|--------|
| `contextIsolation` | `true` | Renderer cannot access preload scope directly |
| `nodeIntegration` | `false` | Renderer cannot call `require()` |
| `sandbox` | `true` | Additional OS-level process restrictions |
| `webSecurity` | `true` (default) | Keep same-origin policy enforced |
| `allowRunningInsecureContent` | `false` (default) | Never load HTTP content in an HTTPS context |
| Content Security Policy | Set via `<meta>` tag | Prevents XSS in the renderer |
| Shell / exec calls | Validate all input from renderer | Never pass renderer strings directly to `exec` or `shell.openExternal` |
| URL validation | Check protocol before `shell.openExternal` | Prevents `javascript:` or `file:` URL injection |
| `event.sender` validation | Check `getURL()` in IPC handlers | Prevents spoofed IPC from injected scripts |

```javascript
// src/main.js — sender validation in every IPC handler
function isSenderTrusted(event) {
  const url = event.sender.getURL();
  // Allow only file:// origins (your own HTML files)
  // In production, also check the specific path
  return url.startsWith("file://");
}

ipcMain.handle("write-file", async (event, { filePath, content }) => {
  if (!isSenderTrusted(event)) throw new Error("Untrusted IPC sender");
  // ... rest of handler
});
```

---

## ✅ Part 12: Complete Application — Putting It All Together

Here is the final, complete `src/main.js` that integrates all the concepts from this module:

```javascript
// src/main.js — Complete Cluster Monitor
"use strict";

const { app, BrowserWindow, ipcMain, Menu, Tray, dialog,
        shell, Notification, nativeImage } = require("electron");
const path   = require("path");
const os     = require("os");
const fs     = require("fs/promises");
const config = require("./config");

// ─── State ────────────────────────────────────────────────────────────────────
let mainWindow    = null;
let tray          = null;
let updateTimer   = null;

// ─── Helpers ──────────────────────────────────────────────────────────────────
function isSenderTrusted(event) {
  return event.sender.getURL().startsWith("file://");
}

function getSystemSnapshot() {
  return {
    hostname:  os.hostname(),
    platform:  os.platform(),
    arch:      os.arch(),
    cpus:      os.cpus().length,
    memory:    { total: os.totalmem(), free: os.freemem() },
    uptime:    os.uptime(),
    timestamp: new Date().toISOString(),
  };
}

// ─── Window Management ────────────────────────────────────────────────────────
async function createWindow() {
  const cfg = await config.load();

  mainWindow = new BrowserWindow({
    width:    cfg.windowBounds.width,
    height:   cfg.windowBounds.height,
    x:        cfg.windowBounds.x  ?? undefined,
    y:        cfg.windowBounds.y  ?? undefined,
    show:     false,
    webPreferences: {
      preload:          path.join(__dirname, "preload.js"),
      contextIsolation: true,
      nodeIntegration:  false,
      sandbox:          true,
    },
  });

  mainWindow.loadFile(path.join(__dirname, "renderer/index.html"));
  mainWindow.once("ready-to-show", () => mainWindow.show());

  mainWindow.on("close", async () => {
    await config.save({ windowBounds: mainWindow.getBounds() });
  });

  mainWindow.on("closed", () => { mainWindow = null; });

  if (process.env.NODE_ENV === "development") {
    mainWindow.webContents.openDevTools();
  }
}

// ─── Live Update Loop ─────────────────────────────────────────────────────────
async function startLiveUpdates() {
  const cfg = await config.load();
  if (updateTimer) clearInterval(updateTimer);

  updateTimer = setInterval(() => {
    if (mainWindow && !mainWindow.isDestroyed()) {
      mainWindow.webContents.send("live-update", {
        ts:       Date.now(),
        totalMem: os.totalmem(),
        freeMem:  os.freemem(),
        uptime:   os.uptime(),
      });
    }
  }, cfg.refreshInterval);
}

// ─── IPC Handlers ─────────────────────────────────────────────────────────────
ipcMain.handle("get-system-info", async (event) => {
  if (!isSenderTrusted(event)) throw new Error("Untrusted sender");
  return getSystemSnapshot();
});

ipcMain.handle("read-config", async (event) => {
  if (!isSenderTrusted(event)) throw new Error("Untrusted sender");
  return config.load();
});

ipcMain.handle("save-config", async (event, updates) => {
  if (!isSenderTrusted(event)) throw new Error("Untrusted sender");
  const saved = await config.save(updates);
  // Restart live updates if interval changed
  await startLiveUpdates();
  return saved;
});

ipcMain.handle("write-file", async (event, { filePath, content }) => {
  if (!isSenderTrusted(event)) throw new Error("Untrusted sender");
  await fs.mkdir(path.dirname(filePath), { recursive: true });
  await fs.writeFile(filePath, content, "utf8");
  return { success: true };
});

ipcMain.handle("save-file-dialog", async (event, options = {}) => {
  if (!isSenderTrusted(event)) throw new Error("Untrusted sender");
  const result = await dialog.showSaveDialog(mainWindow, {
    title:       options.title       ?? "Save File",
    defaultPath: options.defaultPath ?? "output.json",
    filters:     options.filters     ?? [{ name: "All Files", extensions: ["*"] }],
  });
  return result.canceled ? null : result.filePath;
});

ipcMain.handle("show-message-box", async (event, options) => {
  if (!isSenderTrusted(event)) throw new Error("Untrusted sender");
  const result = await dialog.showMessageBox(mainWindow, {
    type:    options.type    ?? "info",
    title:   options.title   ?? "Message",
    message: options.message ?? "",
    detail:  options.detail  ?? "",
    buttons: options.buttons ?? ["OK"],
  });
  return result.response;
});

ipcMain.handle("open-external", async (event, url) => {
  if (!isSenderTrusted(event)) throw new Error("Untrusted sender");
  const parsed = new URL(url);
  if (!["http:", "https:"].includes(parsed.protocol)) throw new Error("Blocked protocol");
  await shell.openExternal(url);
});

// ─── Menu ─────────────────────────────────────────────────────────────────────
function buildMenu() {
  const isMac = process.platform === "darwin";
  const template = [
    ...(isMac ? [{ label: app.name, submenu: [
      { role: "about" }, { type: "separator" }, { role: "quit" },
    ]}] : []),
    { label: "File", submenu: [
      { label: "Save Snapshot…", accelerator: "CmdOrCtrl+S",
        click: () => mainWindow?.webContents.send("trigger-snapshot") },
      { type: "separator" },
      isMac ? { role: "close" } : { role: "quit" },
    ]},
    { label: "Monitor", submenu: [
      { label: "Refresh Now", accelerator: "CmdOrCtrl+R",
        click: () => mainWindow?.webContents.send("refresh-data") },
    ]},
    { label: "View", submenu: [
      { role: "reload" }, { role: "toggleDevTools" },
      { type: "separator" }, { role: "togglefullscreen" },
    ]},
  ];
  return Menu.buildFromTemplate(template);
}

// ─── Tray ──────────────────────────────────────────────────────────────────────
function createTray() {
  const iconPath = path.join(__dirname, "../assets/tray-icon.png");
  try {
    const icon = nativeImage.createFromPath(iconPath);
    tray = new Tray(icon.isEmpty() ? nativeImage.createEmpty() : icon);
  } catch {
    tray = new Tray(nativeImage.createEmpty());
  }

  tray.setToolTip("Cluster Monitor");
  tray.setContextMenu(Menu.buildFromTemplate([
    { label: "Show",  click: () => mainWindow?.show() },
    { label: "Quit",  click: () => app.quit() },
  ]));
  tray.on("click", () => mainWindow?.isVisible() ? mainWindow.hide() : mainWindow.show());
}

// ─── App Lifecycle ────────────────────────────────────────────────────────────
app.whenReady().then(async () => {
  Menu.setApplicationMenu(buildMenu());
  await createWindow();
  createTray();
  await startLiveUpdates();

  app.on("activate", () => {
    if (BrowserWindow.getAllWindows().length === 0) createWindow();
  });
});

app.on("window-all-closed", () => {
  if (process.platform !== "darwin") app.quit();
});

app.on("before-quit", () => {
  if (updateTimer) clearInterval(updateTimer);
});

process.on("uncaughtException",  (err) => console.error("Uncaught:",            err));
process.on("unhandledRejection", (err) => console.error("Unhandled rejection:", err));
```

---

## ✅ Part 13: Practice Exercises

### Exercise 1 — IPC Round-Trip

Extend the app to show the 5 most recently modified files in the user's home directory. The file listing must happen in the main process (using `fs.readdir` and `fs.stat`), and the results must be displayed in the renderer via a new `get-recent-files` IPC channel.

Hints:
- Use `Promise.all` to stat all files concurrently
- Sort by `stat.mtime` descending
- Add the new IPC handler to `main.js` and expose it in `preload.js`

### Exercise 2 — Live CPU Load

The main process can measure CPU load by sampling `os.cpus()` at two points in time. Add CPU utilization (%) to the live-update push messages.

```javascript
// Starter: CPU sampling utility
function sampleCpuTimes() {
  return os.cpus().map(cpu => {
    const total = Object.values(cpu.times).reduce((a, b) => a + b, 0);
    return { idle: cpu.times.idle, total };
  });
}

function calcCpuPercent(before, after) {
  const diffs = before.map((b, i) => ({
    idle:  after[i].idle  - b.idle,
    total: after[i].total - b.total,
  }));
  const avg = diffs.reduce((sum, d) => sum + (1 - d.idle / d.total), 0) / diffs.length;
  return Math.round(avg * 100);
}
```

Add a CPU percentage bar to the renderer that updates with every live-update tick.

### Exercise 3 — Settings Window

Create a second `settings.html` renderer and a corresponding `openSettingsWindow()` function in `main.js`. The settings window should:
- Read the current config via `read-config` IPC
- Allow the user to change the refresh interval (500ms – 10000ms)
- Save changes via `save-config` IPC
- Be a modal sheet on macOS (use `parent` and `modal: true` options)

### Exercise 4 — Log File

Every snapshot saved by the user should also be appended to a rolling log file at `app.getPath("userData")/history.jsonl` (one JSON object per line). Add an IPC handler `read-history` that reads this file and returns the last N snapshots as a parsed array. Display the history in a collapsible section of the renderer.

### Exercise 5 — System Tray Notifications

When memory usage exceeds 85%, send a native `Notification` from the main process. Ensure the notification fires at most once per minute to avoid spam. Store the last notification timestamp in the main process and check it before sending.

---

## 📋 Key Concepts Summary

| Concept | Key Points |
|---|---|
| **Two-process model** | Main (Node.js) and Renderer (Chromium) run in separate OS processes. They communicate only via IPC. |
| **`contextIsolation: true`** | Required. Isolates the renderer's JavaScript context from the preload script. |
| **`nodeIntegration: false`** | Required. Prevents the renderer from calling `require()`. |
| **Preload script** | The only file with access to both Node.js and the browser `window`. Always use `contextBridge.exposeInMainWorld`. |
| **`ipcMain.handle` / `ipcRenderer.invoke`** | Request-response IPC. Handler returns a value; renderer gets it as a resolved Promise. |
| **`webContents.send`** | Main-to-renderer push. Used for live updates, file watch events, and menu actions. |
| **`dialog`** | Native file open/save/message dialogs. Must be called from the main process. |
| **`shell`** | Opens URLs and files in external applications. Always validate URLs before calling `openExternal`. |
| **`Tray`** | System tray icon. Hold a reference — if GC'd, the tray disappears. |
| **`app.getPath("userData")`** | OS-appropriate directory for storing user settings and app data. |
| **`Menu.buildFromTemplate`** | Declarative native menu construction. Use `role` for standard OS menu items. |
| **`electron-builder`** | Packages your app into distributable formats (dmg, exe, AppImage, deb). |
| **Sender validation** | Check `event.sender.getURL()` in IPC handlers to reject calls from unexpected origins. |
| **`path.join(__dirname, ...)`** | Always use `__dirname` to reference assets in the main process — never relative paths from cwd. |
| **`show: false` + `ready-to-show`** | Prevents a blank white window flash during startup. |

---

## 🔧 Troubleshooting

- **White flash on startup** — Add `show: false` to BrowserWindow options and call `win.show()` inside the `ready-to-show` event.
- **`window.electronAPI` is undefined** — The preload script path is wrong, or `contextIsolation` is `false`. Check that `path.join(__dirname, "preload.js")` resolves to the correct file and that `contextIsolation: true` is set.
- **IPC handler not found** — You called `ipcRenderer.invoke("channel-name")` but registered `ipcMain.handle("different-channel-name")`. Channel names must match exactly. Also ensure `ipcMain.handle` is called before the renderer sends the message.
- **`Cannot read properties of undefined (reading 'send')` on `mainWindow`** — The window was closed and `mainWindow` is `null`. Guard every `mainWindow.webContents.send(...)` call with `if (mainWindow && !mainWindow.isDestroyed())`.
- **DevTools not opening** — Call `win.webContents.openDevTools()` after `loadFile()`, or after the `ready-to-show` event.
- **Tray icon disappears** — You assigned the `Tray` object to a local variable that went out of scope and was garbage collected. Assign it to a module-level variable (e.g., `let tray = null;`).
- **Context menu doesn't appear** — `menu.popup()` must be called in the main process. Ensure you are calling `ipcRenderer.send` (not `.invoke`) from the renderer for context menu requests, and handling it with `ipcMain.on` (not `.handle`).
- **File dialog returns `undefined`** — The dialog was cancelled. Always check `if (!filePath) return;` before using the result.
- **App works in dev but crashes in production** — Asset paths are probably broken. In packaged apps, `__dirname` points inside the ASAR archive. Use `path.join(__dirname, "renderer/index.html")` — never `process.cwd()` — for all internal file references.
- **`require is not defined` in renderer** — You tried to call `require()` in `renderer.js`. The renderer cannot use Node.js. Move the code to `main.js` and expose it via IPC.
- **`electron` command not found** — Run `npx electron .` instead of `electron .`, or ensure your `npm start` script uses the local package.

---

## 🔗 What's Next

- **Module 3**: Building REST APIs with Express — routing, middleware, request parsing, and error handling
- **Module 4**: Working with Databases — PostgreSQL with `pg`, SQLite with `better-sqlite3`
- **Module 5**: Streams and Buffers — efficient handling of large files and network data
- **Module 6**: Testing — the built-in `node:test` runner, assertions, and test structure
- **Electron ecosystem** — [`electron-vite`](https://electron-vite.org/) for hot-module reloading during development; [`electron-forge`](https://www.electronforge.io/) as an alternative to electron-builder with plugin support

---

*Course version 1.0 — March 2026*
