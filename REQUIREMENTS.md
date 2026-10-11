# 📧 English-to-Japanese Email Translator — Requirements

> **College Minor Project** | Python-based AI email translation pipeline using HuggingFace Transformers + NVIDIA AI Endpoints

---

## 🐍 Python Version

| Requirement | Version |
|-------------|---------|
| **Minimum** | Python 3.9 |
| **Recommended** | Python 3.10 or 3.11 |

> [!IMPORTANT]
> Python 3.12+ may have compatibility issues with some versions of `torch` and `transformers`. Stick to **3.10 or 3.11** for the smoothest experience.

---

## 📦 pip Dependencies (`requirements.txt`)

Create a `requirements.txt` file in the project root with the following:

```txt
# Core ML / Inference
torch>=2.1.0
transformers>=4.38.0
sentencepiece>=0.1.99          # Required by NLLB tokenizer
sacremoses>=0.0.53             # Tokenization utility for multilingual models
accelerate>=0.26.0             # Speeds up model loading (optional but recommended)

# LangChain ecosystem
langchain-core>=0.1.30
langchain-nvidia-ai-endpoints>=0.1.0

# Environment & config
python-dotenv>=1.0.0

# Standard library (no install needed — included with Python)
# smtplib
# email.mime.text
# email.mime.multipart
```

Install all dependencies with:

```bash
pip install -r requirements.txt
```

---

## 🖥️ System Requirements

### Minimum (CPU-only)
| Component | Requirement |
|-----------|-------------|
| RAM | 8 GB |
| Disk Space | ~3 GB (model + dependencies) |
| OS | Windows 10/11, Ubuntu 20.04+, macOS 12+ |
| Python | 3.9 – 3.11 |

### Recommended (GPU-accelerated)
| Component | Requirement |
|-----------|-------------|
| GPU | NVIDIA GPU with **4 GB+ VRAM** |
| CUDA | **11.8 or 12.1** |
| cuDNN | Matching your CUDA version |
| RAM | 16 GB |
| Disk Space | ~3 GB |

> [!TIP]
> GPU inference with CUDA is **5–10× faster** than CPU for the NLLB model. If you have an NVIDIA GPU, install the CUDA-enabled version of PyTorch (see below).

### Install PyTorch with CUDA support

Visit [https://pytorch.org/get-started/locally/](https://pytorch.org/get-started/locally/) and select your CUDA version, or run:

```bash
# For CUDA 11.8
pip install torch --index-url https://download.pytorch.org/whl/cu118

# For CUDA 12.1
pip install torch --index-url https://download.pytorch.org/whl/cu121

# CPU-only fallback
pip install torch --index-url https://download.pytorch.org/whl/cpu
```

---

## 🔑 Environment Variables

Create a `.env` file in the project root. **Never commit this file to version control.**

### `.env` Template

```dotenv
# ─── NVIDIA AI Endpoints ────────────────────────────────────────────────────
# Obtain your key at: https://build.nvidia.com/
NVIDIA_API_KEY=nvapi-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# ─── Gmail SMTP Credentials ─────────────────────────────────────────────────
# Your Gmail address
SENDER_EMAIL=your_email@gmail.com

# Gmail App Password (16-character, no spaces) — NOT your regular password
# Setup instructions: see section below
SENDER_APP_PASSWORD=xxxx xxxx xxxx xxxx

# ─── Recipient (optional, can be set in code) ────────────────────────────────
RECIPIENT_EMAIL=recipient@example.com

# ─── Local Model Path ────────────────────────────────────────────────────────
# Path to the downloaded NLLB model directory
MODEL_PATH=./models/nllb-200-distilled-600M
```

Add `.env` to your `.gitignore`:

```gitignore
.env
models/
__pycache__/
*.pyc
```

---

## 🤗 Model Download Instructions

This project uses **`facebook/nllb-200-distilled-600M`** — a multilingual seq2seq model by Meta AI, supporting 200 languages including Japanese.

### Option 1 — Download via Python (recommended)

```python
from transformers import AutoModelForSeq2SeqLM, AutoTokenizer

model_name = "facebook/nllb-200-distilled-600M"
save_dir = "./models/nllb-200-distilled-600M"

print("Downloading tokenizer...")
tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.save_pretrained(save_dir)

print("Downloading model...")
model = AutoModelForSeq2SeqLM.from_pretrained(model_name)
model.save_pretrained(save_dir)

print(f"Model saved to: {save_dir}")
```

Save this as `download_model.py` and run:

```bash
python download_model.py
```

### Option 2 — Download via Hugging Face CLI

```bash
pip install huggingface_hub
huggingface-cli download facebook/nllb-200-distilled-600M --local-dir ./models/nllb-200-distilled-600M
```

### Model Details

| Property | Value |
|----------|-------|
| Model ID | `facebook/nllb-200-distilled-600M` |
| Architecture | `M2M100ForConditionalGeneration` |
| Parameters | ~600 million |
| Download Size | ~2.4 GB |
| Japanese Target Language Code | `jpn_Jpan` |
| English Source Language Code | `eng_Latn` |

---

## 📬 Gmail App Password Setup

> [!IMPORTANT]
> Google no longer allows regular passwords for SMTP. You **must** use a Gmail **App Password**.

### Steps

1. **Enable 2-Step Verification** on your Google account:
   - Go to [https://myaccount.google.com/security](https://myaccount.google.com/security)
   - Under *"How you sign in to Google"*, enable **2-Step Verification**

2. **Generate an App Password**:
   - Go to [https://myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
   - Select app: **"Mail"** | Select device: **"Windows Computer"** (or Other)
   - Click **Generate**
   - Copy the **16-character password** shown (e.g., `abcd efgh ijkl mnop`)

3. **Add to `.env`**:
   ```dotenv
   SENDER_APP_PASSWORD=abcd efgh ijkl mnop
   ```

> [!NOTE]
> The App Password works even with spaces. You can paste it as-is from Google.

---

## ⚙️ Hardware Requirements Summary

| Use Case | CPU | RAM | GPU | VRAM |
|----------|-----|-----|-----|------|
| Development / Testing | Any modern x86-64 | 8 GB | Optional | — |
| Faster Inference | Any modern x86-64 | 16 GB | NVIDIA (CUDA) | 4 GB+ |
| Production / Batch | Multi-core | 16 GB+ | NVIDIA (CUDA) | 6 GB+ |

---

## 🧩 Optional Dependencies

```txt
# Notebook / interactive development
jupyter>=1.0.0
ipykernel>=6.0.0

# Progress bars for model loading
tqdm>=4.66.0

# Logging and debugging
rich>=13.0.0          # Pretty terminal output

# Testing
pytest>=7.4.0

# Linting / formatting
black>=24.0.0
ruff>=0.3.0
```

Install optional extras:

```bash
pip install jupyter tqdm rich pytest black ruff
```

---

## 🗂️ Suggested Project Structure

```
email translator/
├── .env                          # Secret keys (never commit)
├── .gitignore
├── requirements.txt
├── REQUIREMENTS.md               # This file
├── download_model.py             # One-time model download script
├── main.py                       # Entry point
├── translator.py                 # NLLB translation logic
├── email_sender.py               # SMTP email sending logic
├── models/
│   └── nllb-200-distilled-600M/  # Downloaded model files
└── README.md
```

---

## 🚀 Quick Start Checklist

- [ ] Install Python 3.10 or 3.11
- [ ] Create and activate a virtual environment (`python -m venv venv`)
- [ ] Install dependencies: `pip install -r requirements.txt`
- [ ] Create `.env` from the template above
- [ ] Get NVIDIA API key from [build.nvidia.com](https://build.nvidia.com/)
- [ ] Set up Gmail App Password
- [ ] Download the NLLB model: `python download_model.py`
- [ ] Run the project: `python main.py`

---

*Generated for the English-to-Japanese Email Translator — College Minor Project*
