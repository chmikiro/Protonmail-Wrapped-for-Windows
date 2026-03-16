# Proton Mail Desktop App - Build Guide

Free, open-source desktop wrapper for Proton Mail.

---

## What This Is

A simple Electron wrapper around mail.proton.me with:
- Persistent login (no re-authenticating)
- System tray (minimize to tray)
- Ad/tracker blocking
- Native notifications
- External links in browser

---

## Prerequisites

- Node.js 18+
- npm or yarn
- Windows/Linux/Mac

---

## Project Structure

```
proton-mail-app/
├── package.json      # App config
├── main.js           # Main process
├── preload.js        # Security preload
└── assets/
    ├── icon.png      # App icon (256x256)
    └── tray-icon.png # Tray icon (32x32)
```

---

## Step 1: Initialize Project

```bash
mkdir proton-mail-app
cd proton-mail-app
npm init -y
npm install --save-dev electron electron-builder
```

---

## Step 2: package.json

```json
{
  "name": "proton-mail",
  "version": "1.0.0",
  "description": "Proton Mail Desktop Client",
  "main": "main.js",
  "scripts": {
    "start": "electron .",
    "build": "electron-builder build --win"
  },
  "build": {
    "appId": "com.yourname.protonmail",
    "productName": "Proton Mail",
    "directories": {
      "output": "dist"
    },
    "win": {
      "target": "nsis",
      "icon": "assets/icon.png"
    },
    "nsis": {
      "oneClick": false,
      "allowToChangeInstallationDirectory": true
    }
  },
  "devDependencies": {
    "electron": "latest",
    "electron-builder": "latest"
  }
}
```

---

## Step 3: main.js

```javascript
const { app, BrowserWindow, Tray, Menu, nativeImage, session, shell } = require('electron');
const path = require('path');

let mainWindow;
let tray;

// Block ads/trackers
const BLOCKED_DOMAINS = [
  'googlesyndication.com', 'doubleclick.net', 'googletagmanager.com',
  'google-analytics.com', 'facebook.net', 'connect.facebook.net',
  'ads.twitter.com', 'amazon-adsystem.com'
];

function isDomainBlocked(url) {
  try {
    const { hostname } = new URL(url);
    return BLOCKED_DOMAINS.some(domain => hostname.includes(domain));
  } catch { return false; }
}

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 1280, height: 900, minWidth: 800, minHeight: 600,
    icon: path.join(__dirname, 'assets/icon.png'),
    title: 'Proton Mail',
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false, contextIsolation: true,
      partition: 'persist:proton'
    }
  });

  mainWindow.loadURL('https://mail.proton.me');
  mainWindow.setMenuBarVisibility(false);

  const ses = session.fromPartition('persist:proton');
  
  ses.webRequest.onBeforeRequest({ urls: ['*://*/*'] }, (details, callback) => {
    callback({ cancel: isDomainBlocked(details.url) });
  });

  mainWindow.webContents.setWindowOpenHandler(({ url }) => {
    shell.openExternal(url);
    return { action: 'deny' };
  });

  mainWindow.on('close', (e) => {
    if (!app.isQuitting) {
      e.preventDefault();
      mainWindow.hide();
    }
  });
}

function createTray() {
  const trayIcon = nativeImage.createFromPath(path.join(__dirname, 'assets/tray-icon.png'));
  tray = new Tray(trayIcon);
  tray.setToolTip('Proton Mail');
  
  const menu = Menu.buildFromTemplate([
    { label: 'Open', click: () => { mainWindow.show(); mainWindow.focus(); } },
    { label: 'Reload', click: () => mainWindow.reload() },
    { type: 'separator' },
    { label: 'Quit', click: () => { app.isQuitting = true; app.quit(); } }
  ]);
  
  tray.setContextMenu(menu);
  tray.on('click', () => mainWindow.isVisible() ? mainWindow.hide() : mainWindow.show());
}

app.whenReady().then(() => { createWindow(); createTray(); });

const gotLock = app.requestSingleInstanceLock();
if (!gotLock) { app.quit(); }
else {
  app.on('second-instance', () => {
    if (mainWindow) { mainWindow.show(); mainWindow.focus(); }
  });
}

app.on('before-quit', () => app.isQuitting = true);
app.on('activate', () => { if (!mainWindow) createWindow(); });
```

---

## Step 4: preload.js

```javascript
const { contextBridge } = require('electron');

window.addEventListener('DOMContentLoaded', () => {
  contextBridge.exposeInMainWorld('appInfo', {
    version: process.env.npm_package_version
  });
});
```

---

## Step 5: Add Icons

Create `assets/` folder and add:
- `icon.png` - 256x256 app icon
- `tray-icon.png` - 32x32 tray icon

Get Proton icon: https://icon.horse/icon/proton.me

---

## Step 6: Run in Development

```bash
npm start
```

---

## Step 7: Build Installer

```bash
npm run build
```

Output: `dist/Proton Mail-1.0.0.exe` (Windows)

---

## Why This Exists

Proton charges €5/month for their "desktop app" which is literally just mail.proton.me in an Electron wrapper. Same features, same everything, just €60/year for a window frame.

This does the same thing. Free. Open source. No telemetry.