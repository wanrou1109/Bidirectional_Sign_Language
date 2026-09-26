# SignBank：AI金融手語雙向翻譯系統 (Bidirectional Sign Language and Speech Recognition System)

一個為銀行服務設計的即時手語和語音辨識系統，支援聾人與銀行員工之間的無障礙溝通。

## 功能特色

- **即時手語辨識**: 使用 MediaPipe 進行手部追蹤，結合機器學習模型進行手語識別
- **語音辨識**: 基於 Whisper 模型的語音轉文字功能
- **雙向溝通**: 支援聾人客戶與銀行員工的即時對話
- **訊息編輯**: 允許修改和重新錄製辨識結果
- **使用者回饋**: 完整的服務評價系統

## 技術架構

### 前端 (Frontend)
- **React 18**: 主要 UI 框架
- **MediaPipe**: 手部關節點檢測
- **WebRTC**: 影像和音訊串流
- **React Router**: 路由管理

### 後端 (Backend)
- **Node.js + Express**: API 伺服器
- **MongoDB**: 資料儲存
- **Python Flask**: 機器學習模型服務
- **Whisper**: 語音辨識模型

## 系統需求

### 硬體
- **CPU**：8 核心以上（同時跑 TensorFlow、MediaPipe、Whisper、sentence-transformers 這些模型，核心數不夠會很吃力）
- **記憶體**：16GB 以上

### 軟體
- Node.js 16+
- **Python 3.11**（重要：`tensorflow==2.15` 不支援 3.12/3.13，裝錯版本會在 import 階段就失敗）
- MongoDB（本機需先跑在 `localhost:27017`，否則 Node 後端會啟動失敗）
- 攝影機和麥克風權限
- 現代瀏覽器 (Chrome, Firefox, Safari)

> **注意**：以下安裝步驟裡釘死的套件版本（`tf215_env.yml`）都是在 **macOS（Apple Silicon）** 上實測驗證過的，例如 `tensorflow-io-gcs-filesystem==0.37.1` 這個版本就是專門為了 macOS 上裝得起來才調整的。Windows／Linux 使用者照著裝可能會遇到某幾個套件版本對不上（yml 裡有留一些 Windows 用的替代版本註解可以參考切換）。

## 安裝與設定

專案分三個要一起跑的服務：**React 前端**（3000）、**Node/Express 後端**（8080）、**Python Flask 服務**（5050），外加一個**本機 MongoDB**（27017）。四個都要起來整個系統才會動。

### 0. clone 專案
```bash
git clone <repository-url>
cd Bidirectional_Sign_Language
```

### 1. 啟動 MongoDB
```bash
# 沒裝過的話（macOS 範例）
brew tap mongodb/brew
brew install mongodb-community

# 啟動（保持這個 terminal 開著，或用 brew services start mongodb-community 讓它常駐）
mongod --dbpath ~/data/db
```
Node 後端會連 `mongodb://localhost:27017/SignLanguageApp`，這個資料庫和 collection 會自動建立，不用手動建。

### 2. 建立 Python 環境（給 Flask 服務用）
建議在**repo 根目錄**（跟這個 README 同一層）建虛擬環境，這樣可以同時給 `永豐產學/app.py` 和語音辨識腳本共用：

```bash
# 需要 Python 3.11（沒有的話用 pyenv 裝：pyenv install 3.11.11）
python3.11 -m venv .venv
source .venv/bin/activate

pip install --upgrade pip
pip install "setuptools<81"   # openai-whisper 的舊版 setup.py 需要 pkg_resources，新版 setuptools 已移除

# 安裝 app.py 需要的套件（tensorflow、mediapipe、flask、sentence-transformers…）
pip install $(python -c "
import yaml
d = yaml.safe_load(open('永豐產學/tf215_env.yml'))
for dep in d['dependencies']:
    if isinstance(dep, dict) and 'pip' in dep:
        print(' '.join(dep['pip']))
" 2>/dev/null || echo "yaml 模組沒裝，先 pip install pyyaml 再重跑這行")

# 安裝語音辨識需要的套件（whisper、torch）
pip install --no-build-isolation -r 永豐產學/App/server/speech_recognition/requirements.txt
```

> 也可以用 conda 走一步到位：`conda env create -f 永豐產學/tf215_env.yml`，但 whisper 那份還是要另外用上面的 `--no-build-isolation` 指令裝。

### 3. 設定 API Key
複製 `永豐產學/App/.env`，把 `OPENROUTER_API_KEY` 換成你自己申請的 key（[openrouter.ai](https://openrouter.ai) 申請）：
```
HOST=localhost
DANGEROUSLY_DISABLE_HOST_CHECK=true
OPENROUTER_API_KEY=你自己的key
```
**這個檔案不要 commit 進 git**（裡面是真實的密鑰）。

### 4. 前端 + Node 後端
```bash
cd 永豐產學/App
npm install

# 讓 Node 後端呼叫 Python 時，用剛剛建的 venv（不設的話會抓到系統的 python，裝的套件對不上）
export PYTHON_PATH=$(pwd)/../../.venv/bin/python

# 兩個一起跑（前端 3000 + 後端 8080）
npm run dev
```
或分開兩個 terminal 跑：`npm start`（前端）、`PYTHON_PATH=... npm run server`（後端）。

### 5. Python Flask 服務
```bash
cd 永豐產學   # 一定要在這層跑，程式裡有寫死相對路徑（App/.env、rag_sentence.docx）
source ../.venv/bin/activate
python app.py
```
Flask 會在 `http://localhost:5050` 監聽。**第一次執行**會從 Hugging Face 下載中文語意模型（`shibing624/text2vec-base-chinese`，約 1-2GB），需要網路，且視網速可能要幾分鐘，之後會快取起來不用再下載。

## 使用說明

### 系統流程
1. **歡迎頁面**: 點擊螢幕開始服務
2. **對話頁面**: 主要溝通介面
3. **手語辨識**: 即時手語轉文字
4. **語音辨識**: 語音轉文字
5. **回饋頁面**: 服務評價
6. **感謝頁面**: 自動返回待機

### 操作指南

#### 手語辨識
1. 點擊「手語辨識」按鈕
2. 允許攝影機權限
3. 將手部置於攝影機前
4. 系統將即時顯示辨識結果
5. 點擊停止按鈕結束辨識

#### 語音辨識
1. 點擊「語音辨識」按鈕
2. 允許麥克風權限
3. 開始說話
4. 系統將顯示轉錄結果
5. 點擊停止按鈕結束錄音

#### 訊息編輯
- 點擊訊息旁的編輯按鈕可修改內容
- 點擊重新錄製按鈕可重新進行辨識

## 聯絡資訊

如有任何問題或建議，請聯絡開發團隊。
email: rosalin200311@gmail.com

---

# SignBank: Bidirectional Sign Language and Speech Recognition System

A real-time sign language and speech recognition system built for bank services, enabling barrier-free communication between deaf customers and bank staff.

## Features

- **Real-time sign language recognition**: Hand tracking via MediaPipe combined with a machine learning model for sign recognition
- **Speech recognition**: Speech-to-text powered by the Whisper model
- **Bidirectional communication**: Real-time conversation support between deaf customers and bank staff
- **Message editing**: Edit and re-record recognition results
- **User feedback**: Full service rating system

## Tech Stack

### Frontend
- **React 18**: main UI framework
- **MediaPipe**: hand landmark detection
- **WebRTC**: video/audio streaming
- **React Router**: routing

### Backend
- **Node.js + Express**: API server
- **MongoDB**: data storage
- **Python Flask**: machine learning model service
- **Whisper**: speech recognition model

## Requirements

### Hardware
- **CPU**: 8+ cores (TensorFlow, MediaPipe, Whisper, and sentence-transformers all running together is demanding — fewer cores will struggle)
- **RAM**: 16GB+

### Software
- Node.js 16+
- **Python 3.11** (important: `tensorflow==2.15` does not support 3.12/3.13 — the wrong version will fail at import time)
- MongoDB (must be running locally on `localhost:27017`, otherwise the Node backend fails to start)
- Camera and microphone permissions
- A modern browser (Chrome, Firefox, Safari)

> **Note**: the pinned package versions in the setup steps below (`tf215_env.yml`) were tested and verified on **macOS (Apple Silicon)** — for example, `tensorflow-io-gcs-filesystem==0.37.1` was specifically chosen to make installation work on macOS. Windows/Linux users may hit a few version mismatches (the yml file has some commented-out alternate versions for Windows you can switch to).

## Setup

The project has three services that must run together: the **React frontend** (port 3000), the **Node/Express backend** (port 8080), and the **Python Flask service** (port 5050) — plus a **local MongoDB** (port 27017). All four need to be running for the system to work.

### 0. Clone the project
```bash
git clone <repository-url>
cd Bidirectional_Sign_Language
```

### 1. Start MongoDB
```bash
# If not installed yet (macOS example)
brew tap mongodb/brew
brew install mongodb-community

# Start it (keep this terminal open, or use `brew services start mongodb-community` to run it in the background)
mongod --dbpath ~/data/db
```
The Node backend connects to `mongodb://localhost:27017/SignLanguageApp`. The database and collections are created automatically — no manual setup needed.

### 2. Set up the Python environment (for the Flask service)
Create the virtual environment in the **repo root** (same level as this README) so it can be shared by both `永豐產學/app.py` and the speech recognition scripts:

```bash
# Requires Python 3.11 (install via pyenv if needed: pyenv install 3.11.11)
python3.11 -m venv .venv
source .venv/bin/activate

pip install --upgrade pip
pip install "setuptools<81"   # openai-whisper's legacy setup.py needs pkg_resources, which newer setuptools removed

# Install the packages app.py needs (tensorflow, mediapipe, flask, sentence-transformers, …)
pip install $(python -c "
import yaml
d = yaml.safe_load(open('永豐產學/tf215_env.yml'))
for dep in d['dependencies']:
    if isinstance(dep, dict) and 'pip' in dep:
        print(' '.join(dep['pip']))
" 2>/dev/null || echo "pyyaml not installed — run 'pip install pyyaml' and re-run this line")

# Install the packages speech recognition needs (whisper, torch)
pip install --no-build-isolation -r 永豐產學/App/server/speech_recognition/requirements.txt
```

> Alternatively, use conda for a one-step setup: `conda env create -f 永豐產學/tf215_env.yml` — but you'll still need the `--no-build-isolation` command above for whisper.

### 3. Configure the API key
Copy `永豐產學/App/.env` and replace `OPENROUTER_API_KEY` with your own key (sign up at [openrouter.ai](https://openrouter.ai)):
```
HOST=localhost
DANGEROUSLY_DISABLE_HOST_CHECK=true
OPENROUTER_API_KEY=your_own_key
```
**Never commit this file to git** — it contains a real secret key.

### 4. Frontend + Node backend
```bash
cd 永豐產學/App
npm install

# Point the Node backend to the venv you just created so it can call Python correctly
# (without this it falls back to the system's python, whose packages won't match)
export PYTHON_PATH=$(pwd)/../../.venv/bin/python

# Run both together (frontend on 3000 + backend on 8080)
npm run dev
```
Or run them in two separate terminals: `npm start` (frontend) and `PYTHON_PATH=... npm run server` (backend).

### 5. Python Flask service
```bash
cd 永豐產學   # must run from this directory — the code has hardcoded relative paths (App/.env, rag_sentence.docx)
source ../.venv/bin/activate
python app.py
```
Flask will listen on `http://localhost:5050`. **On first run**, it downloads a Chinese sentence-embedding model from Hugging Face (`shibing624/text2vec-base-chinese`, roughly 1-2GB) — this needs an internet connection and may take a few minutes depending on your connection speed; it's cached afterward so it won't download again.

## Usage

### System flow
1. **Welcome screen**: tap the screen to start
2. **Conversation screen**: main communication interface
3. **Sign language recognition**: real-time sign-to-text
4. **Speech recognition**: speech-to-text
5. **Feedback screen**: service rating
6. **Thank-you screen**: automatically returns to standby

### Operating guide

#### Sign language recognition
1. Tap the "Sign Language Recognition" button
2. Allow camera access
3. Position your hands in front of the camera
4. The system displays the recognition result in real time
5. Tap Stop to end recognition

#### Speech recognition
1. Tap the "Speech Recognition" button
2. Allow microphone access
3. Start speaking
4. The system displays the transcription
5. Tap Stop to end recording

#### Message editing
- Tap the edit button next to a message to modify it
- Tap the re-record button to redo recognition

## Contact

For questions or suggestions, please contact the development team.
email: rosalin200311@gmail.com
