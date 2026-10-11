# English-to-Japanese Email Translator — Setup Guide

> **College Minor Project** — Step-by-step instructions to get the project running locally.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Clone the Repository](#2-clone-the-repository)
3. [Create a Virtual Environment](#3-create-a-virtual-environment)
4. [Install Dependencies](#4-install-dependencies)
5. [Download the HuggingFace Model](#5-download-the-huggingface-model)
6. [Configure the Model Path](#6-configure-the-model-path)
7. [Create the `.env` File](#7-create-the-env-file)
8. [Gmail App Password Setup](#8-gmail-app-password-setup)
9. [NVIDIA API Key Setup](#9-nvidia-api-key-setup)
10. [Verification & Module Testing](#10-verification--module-testing)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Prerequisites

Make sure the following are installed on your system before proceeding:

| Requirement | Minimum Version | Notes |
|---|---|---|
| **Python** | 3.9+ | [python.org](https://www.python.org/downloads/) |
| **pip** | 23+ | Comes with Python |
| **Git** | Any | [git-scm.com](https://git-scm.com/) |
| **CUDA** *(optional)* | 11.8+ | Only if using an NVIDIA GPU for faster inference |
| **RAM** | 8 GB+ | 16 GB recommended for the translation model |
| **Disk Space** | ~5 GB | For the model weights |

> **Tip:** Run `python --version` and `pip --version` in your terminal to confirm they are installed and on your PATH.

---

## 2. Clone the Repository

```bash
git clone <YOUR_REPOSITORY_URL>
cd "email translator"
```

Replace `<YOUR_REPOSITORY_URL>` with the actual GitHub/GitLab URL of the project.

---

## 3. Create a Virtual Environment

It is strongly recommended to use a virtual environment to avoid dependency conflicts.

```bash
# Create the virtual environment
python -m venv venv

# Activate it — Windows (PowerShell)
.\venv\Scripts\Activate.ps1

# Activate it — Windows (Command Prompt)
.\venv\Scripts\activate.bat

# Activate it — macOS / Linux
source venv/bin/activate
```

You should see `(venv)` at the beginning of your terminal prompt once activated.

---

## 4. Install Dependencies

### 4a. Upgrade pip first

```bash
pip install --upgrade pip
```

### 4b. Install PyTorch

Choose the correct command based on whether you have a CUDA-capable GPU:

**CPU only (no GPU):**
```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
```

**CUDA 11.8 (NVIDIA GPU):**
```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
```

**CUDA 12.1 (NVIDIA GPU):**
```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
```

> **Note:** Visit [pytorch.org/get-started](https://pytorch.org/get-started/locally/) to get the exact command for your system configuration.

### 4c. Install the remaining dependencies

```bash
pip install \
    transformers \
    langchain-core \
    langchain-nvidia-ai-endpoints \
    python-dotenv \
    sentencepiece \
    sacremoses
```

Or if a `requirements.txt` file is present in the repo:

```bash
pip install -r requirements.txt
```

---

## 5. Download the HuggingFace Model

The translation module uses an **NLLB-style sequence-to-sequence model**. Download it once and save it locally.

### Option A — Download via Python script (recommended)

Save the following as `download_model.py` in the project root, then run `python download_model.py`:

```python
from transformers import AutoModelForSeq2SeqLM, AutoTokenizer

MODEL_NAME = "facebook/nllb-200-distilled-600M"   # adjust if a different checkpoint is used
SAVE_PATH  = r"C:\Users\kumar\OneDrive\Desktop\colg\college-minor-project-\model"

print("Downloading tokenizer...")
tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME)
tokenizer.save_pretrained(SAVE_PATH)

print("Downloading model weights...")
model = AutoModelForSeq2SeqLM.from_pretrained(MODEL_NAME)
model.save_pretrained(SAVE_PATH)

print(f"Model saved to: {SAVE_PATH}")
```

```bash
python download_model.py
```

### Option B — Manual download from HuggingFace Hub

1. Go to [huggingface.co/facebook/nllb-200-distilled-600M](https://huggingface.co/facebook/nllb-200-distilled-600M) (or the specific checkpoint used by the project).
2. Click **Files and versions** and download all files.
3. Place them inside:
   ```
   C:\Users\kumar\OneDrive\Desktop\colg\college-minor-project-\model
   ```

> **Important:** The folder must contain at minimum: `config.json`, `tokenizer.json` (or `sentencepiece.bpe.model`), and the model weights (`pytorch_model.bin` or `model.safetensors`).

---

## 6. Configure the Model Path

The model path is **hardcoded** in `translate.py`. If you are running on a different machine or changed the download location, update it:

1. Open `translate.py`.
2. Find the line that sets the model path (look for `C:\\Users\\kumar\\...`).
3. Replace it with the **absolute path** to your downloaded model folder.

**Example:**
```python
# Before
MODEL_PATH = r"C:\Users\kumar\OneDrive\Desktop\colg\college-minor-project-\model"

# After — use YOUR actual path
MODEL_PATH = r"C:\Users\DELL\Documents\email translator\model"
```

> **Warning:** Use a raw string (`r"..."`) or double-backslashes (`\\`) on Windows to avoid path escape issues.

---

## 7. Create the `.env` File

Sensitive credentials are loaded from a `.env` file. **Never commit this file to Git.**

Create a file named `.env` in the project root directory with the following content:

```
# .env — do NOT share or commit this file

# NVIDIA NIM API key (for write_email.py)
NVIDIA_API_KEY=your_nvidia_api_key_here

# Gmail credentials (for email_sender.py)
GMAIL_ADDRESS=your.email@gmail.com
GMAIL_APP_PASSWORD=your_16_character_app_password
```

Replace the placeholder values with your actual credentials (see Sections 8 & 9 below).

---

## 8. Gmail App Password Setup

`email_sender.py` uses Gmail's SMTP server. Google requires an **App Password** (not your regular Gmail password) when 2-Step Verification is enabled.

### Step-by-step

1. Go to your Google Account: [myaccount.google.com](https://myaccount.google.com)
2. Navigate to **Security** → **How you sign in to Google**.
3. Enable **2-Step Verification** if it is not already on.
4. After enabling 2FA, go back to **Security** and search for **"App passwords"**.
5. Click **App passwords** (you may be prompted to sign in again).
6. Under "Select app", choose **Mail**; under "Select device", choose **Windows Computer** (or Other — type a name like `EmailTranslator`).
7. Click **Generate**.
8. Copy the **16-character password** shown (spaces can be ignored — remove them when pasting).
9. Paste it as the value of `GMAIL_APP_PASSWORD` in your `.env` file.

> **Caution:** Store the App Password securely. If compromised, revoke it immediately from Google Account → Security → App passwords.

---

## 9. NVIDIA API Key Setup

`write_email.py` uses NVIDIA's hosted LLM endpoints via `langchain-nvidia-ai-endpoints`.

### Step-by-step

1. Go to [integrate.api.nvidia.com](https://integrate.api.nvidia.com).
2. Sign in or create a free **NVIDIA Developer** account.
3. Navigate to **API Keys** in the dashboard sidebar.
4. Click **Generate API Key** (or **+ New Key**).
5. Copy the key — it begins with `nvapi-...`.
6. Paste it as the value of `NVIDIA_API_KEY` in your `.env` file.

> **Note:** The free tier provides a limited number of tokens per month. Monitor your usage on the NVIDIA dashboard to avoid hitting the quota limit during a demo or evaluation.

---

## 10. Verification & Module Testing

Run each module independently to confirm everything is set up correctly before doing an end-to-end test.

### 10a. Verify Python environment and imports

```bash
python -c "import torch; import transformers; import langchain_core; print('All imports OK')"
```

Expected output:
```
All imports OK
```

### 10b. Verify PyTorch and CUDA (optional)

```bash
python -c "import torch; print('Torch:', torch.__version__); print('CUDA available:', torch.cuda.is_available())"
```

### 10c. Test the translation module

```bash
python translate.py
```

The script should load the model (this may take 30–90 seconds on the first run) and print a translated Japanese sentence.

### 10d. Test the email writing module

```bash
python write_email.py
```

Confirm the NVIDIA endpoint responds with a generated email draft. If you see an authentication error, check that `NVIDIA_API_KEY` is correctly set in `.env`.

### 10e. Test the email sending module

```bash
python email_sender.py
```

> **Important:** Before running, make sure `GMAIL_ADDRESS` and `GMAIL_APP_PASSWORD` are correctly set in `.env`, and that the recipient address in the script is a test address you control.

---

## 11. Troubleshooting

---

### CUDA not found / model running very slowly

**Cause:** PyTorch was installed without CUDA support, or no NVIDIA GPU is present.

**Fix:**
```bash
# Uninstall current torch
pip uninstall torch torchvision torchaudio

# Reinstall with your CUDA version (e.g., 11.8)
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
```

Verify:
```bash
python -c "import torch; print(torch.cuda.is_available())"  # should print: True
```

If you don't have a GPU, the model will run on CPU — it is slower but fully functional.

---

### OSError: Model path not found / No such file or directory

**Cause:** The model was not downloaded, or the path in `translate.py` is incorrect.

**Fix:**
1. Re-run `download_model.py` (see Section 5).
2. Verify the path in `translate.py` exactly matches the folder where the model files were saved (see Section 6).
3. On Windows, double-check that backslashes are escaped (`\\`) or that a raw string (`r"..."`) is used.

---

### SMTPAuthenticationError / Authentication error (Gmail)

**Cause:** Wrong Gmail credentials or 2-Step Verification not enabled.

**Fix:**
1. Confirm **2-Step Verification** is enabled on the sending Google account.
2. Re-generate the App Password from [myaccount.google.com](https://myaccount.google.com) → Security → App passwords.
3. Ensure there are **no spaces** in the `GMAIL_APP_PASSWORD` value in `.env`.
4. If using a Google Workspace (GSuite) account, ask your admin to allow SMTP relay or less-secure app access.

---

### AuthenticationError: 401 from NVIDIA API

**Cause:** Invalid or expired NVIDIA API key.

**Fix:**
1. Log in to [integrate.api.nvidia.com](https://integrate.api.nvidia.com) and generate a new key.
2. Update `NVIDIA_API_KEY` in `.env`.
3. Ensure no extra whitespace or newline characters were accidentally included in the key value.

---

### ModuleNotFoundError: No module named 'sentencepiece'

**Cause:** The NLLB tokenizer requires `sentencepiece`, which is a separate package.

**Fix:**
```bash
pip install sentencepiece sacremoses
```

---

### .env variables not loading / KeyError on os.environ

**Cause:** `python-dotenv` is not installed, or the `.env` file is in the wrong directory.

**Fix:**
```bash
pip install python-dotenv
```

Make sure `.env` is in the **project root** (same folder as your Python scripts), and that your scripts call:

```python
from dotenv import load_dotenv
load_dotenv()
```

Place `load_dotenv()` **before** any `os.getenv(...)` calls.

---

## Project Structure (Reference)

```
email translator/
├── translate.py          # NLLB seq2seq translation (English -> Japanese)
├── write_email.py        # LLM-powered email drafting via NVIDIA NIM
├── email_sender.py       # Gmail SMTP sender
├── download_model.py     # One-time model download helper (create manually)
├── .env                  # Secrets — DO NOT commit
├── .gitignore            # Should include .env and model/
├── requirements.txt      # Python dependencies
├── SETUP.md              # This file
└── model/                # Downloaded HuggingFace model (local copy)
```

---

*For questions or issues, open a GitHub issue or contact the project maintainer.*
