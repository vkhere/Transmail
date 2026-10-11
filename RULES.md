# RULES.md — English-to-Japanese Email Translator

> College Minor Project · Python · NVIDIA AI + HuggingFace + SMTP

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Coding Conventions](#2-coding-conventions)
3. [Architectural Rules](#3-architectural-rules)
4. [Security Rules](#4-security-rules)
5. [Translation Rules](#5-translation-rules)
6. [Email Sending Rules](#6-email-sending-rules)
7. [LLM Prompt Rules](#7-llm-prompt-rules)
8. [Contribution Rules](#8-contribution-rules)

---

## 1. Project Overview

This project consists of three Python scripts that form a pipeline:

| Script | Role | Key Technology |
|---|---|---|
| `write_email.py` | Generate formal professional emails from topic prompts | NVIDIA AI API · Google DiffusionGemma 26B · LangChain `ChatNVIDIA` |
| `translate.py` | Translate generated English emails to Japanese | Local HuggingFace seq2seq model (NLLB-style) |
| `email_sender.py` | Send the final HTML/plain-text email via SMTP | Python `smtplib` · `email.mime` |

---

## 2. Coding Conventions

### 2.1 Style & Formatting

- Follow **PEP 8** for all Python code.
- Use **4 spaces** for indentation — never tabs.
- Maximum line length: **100 characters**.
- Use **snake_case** for variables and function names; **PascalCase** for class names.
- All files must end with a **single newline character**.

### 2.2 Imports

- Group imports in this order, separated by blank lines:
  1. Standard library (`os`, `smtplib`, `email`, etc.)
  2. Third-party libraries (`langchain_nvidia_ai_endpoints`, `transformers`, etc.)
  3. Local/project modules
- Never use wildcard imports (`from module import *`).

### 2.3 Type Hints & Docstrings

- All public functions **must** include type hints for parameters and return values.
- All modules, classes, and public functions **must** have docstrings (Google-style preferred).

```python
def generate_email(topic: str, max_tokens: int = 512) -> str:
    """Generate a formal professional email from a topic prompt.

    Args:
        topic: A short description of the email's subject matter.
        max_tokens: Maximum token budget for the generated response.

    Returns:
        A formatted professional email as a plain-text string.
    """
```

### 2.4 Error Handling

- Always use **specific exception types** — never bare `except:` clauses.
- Log errors with context (what was attempted, what failed) before raising or returning.
- Use `try/except/finally` for any I/O operations (file, network, SMTP).

### 2.5 Constants & Configuration

- Define module-level constants in `UPPER_SNAKE_CASE`.
- Keep all tuneable parameters (model names, token limits, SMTP ports) as named constants at the top of each file or in a shared `config.py` — never as magic numbers inline.

---

## 3. Architectural Rules

### 3.1 Script Responsibilities (Single Responsibility Principle)

- **`write_email.py`** — ONLY responsible for LLM-based email generation. Must not handle translation or SMTP logic.
- **`translate.py`** — ONLY responsible for English→Japanese translation. Must not call the LLM or send emails.
- **`email_sender.py`** — ONLY responsible for SMTP dispatch. Must not generate or translate content.

### 3.2 Inter-Script Communication

- Scripts communicate via **plain-text strings** (or files). Avoid tight coupling through shared global state.
- If orchestrating the full pipeline, use a separate `main.py` or a shell script that calls the three scripts in sequence.

### 3.3 Model Loading

- The HuggingFace model in `translate.py` must be loaded **once** at module level (or inside a guarded `if __name__ == "__main__":` block), not re-loaded on every function call.
- Use lazy loading with a module-level `_model` and `_tokenizer` variable if the script is imported as a library.

### 3.4 Dependencies

- All third-party dependencies must be listed in `requirements.txt` with **pinned versions**.
- Keep the `requirements.txt` up to date whenever a new package is introduced.

---

## 4. Security Rules

> [!CAUTION]
> Violating these rules may expose credentials or compromise accounts. There are no exceptions.

### 4.1 Never Hardcode Credentials

- **NEVER** hardcode API keys, passwords, email addresses, or any secret directly in source code.
- All secrets must be loaded exclusively from environment variables via a **`.env` file** and the `python-dotenv` library.

```python
# ✅ Correct
from dotenv import load_dotenv
import os

load_dotenv()
NVIDIA_API_KEY = os.getenv("NVIDIA_API_KEY")
GMAIL_APP_PASSWORD = os.getenv("GMAIL_APP_PASSWORD")
SENDER_EMAIL = os.getenv("SENDER_EMAIL")
```

```python
# ❌ Wrong — never do this
NVIDIA_API_KEY = "nvapi-xxxxxxxxxxxxxxxxxxxxxxxx"
GMAIL_APP_PASSWORD = "mysecretpassword123"
```

### 4.2 `.env` File Rules

- The `.env` file **must** be listed in `.gitignore` and must never be committed to version control.
- Provide a `.env.example` file with placeholder values to document required variables:

```dotenv
# .env.example
NVIDIA_API_KEY=your_nvidia_api_key_here
SENDER_EMAIL=your_email@gmail.com
GMAIL_APP_PASSWORD=your_gmail_app_password_here
RECIPIENT_EMAIL=recipient@example.com
```

### 4.3 Gmail: Use App Password, NOT Your Real Password

- Gmail SMTP **requires an App Password**, not your actual Google account password.
- Steps to generate an App Password:
  1. Enable **2-Step Verification** on your Google account.
  2. Go to **Google Account → Security → App Passwords**.
  3. Generate a password for "Mail" / "Other".
  4. Store the generated 16-character password as `GMAIL_APP_PASSWORD` in `.env`.
- **Never** disable 2-Step Verification to use your real password with SMTP.

### 4.4 Validate Environment Variables at Startup

- Each script must validate that required environment variables are present at startup and fail fast with a clear error message if any are missing.

```python
if not NVIDIA_API_KEY:
    raise EnvironmentError(
        "NVIDIA_API_KEY is not set. Add it to your .env file."
    )
```

---

## 5. Translation Rules

### 5.1 Model

- Use an **NLLB-style** (No Language Left Behind) HuggingFace seq2seq model for English→Japanese translation.
- Recommended model: `facebook/nllb-200-distilled-600M` (balance of quality and local performance).
- Store the model name as a constant:

```python
TRANSLATION_MODEL_NAME = "facebook/nllb-200-distilled-600M"
SOURCE_LANG = "eng_Latn"
TARGET_LANG = "jpn_Jpan"
```

### 5.2 Maximum Token Length

- **Always** enforce a maximum input token length of **512 tokens** during tokenization.
- Truncate inputs that exceed this limit rather than raising an error.

```python
inputs = tokenizer(
    text,
    return_tensors="pt",
    max_length=512,
    truncation=True,
)
```

### 5.3 Forced BOS Token for Target Language

- **Always** set `forced_bos_token_id` when calling `model.generate()` to ensure the model outputs the correct target language (Japanese).
- This is mandatory — omitting it may produce output in the wrong language.

```python
target_lang_id = tokenizer.convert_tokens_to_ids(TARGET_LANG)

translated_tokens = model.generate(
    **inputs,
    forced_bos_token_id=target_lang_id,
    max_new_tokens=512,
)
```

### 5.4 Output Handling

- Decode translated tokens with `skip_special_tokens=True`.
- Strip leading/trailing whitespace from the decoded string before returning.

### 5.5 Long Email Handling

- If an email body exceeds 512 tokens, split it into logical paragraphs, translate each paragraph independently, and rejoin with newlines.
- Do not split mid-sentence.

---

## 6. Email Sending Rules

### 6.1 Always Use STARTTLS

- **Always** use `smtplib.SMTP` with **STARTTLS** (`port 587`) — never use plain SMTP (`port 25`) or raw SSL (`port 465` via `smtplib.SMTP_SSL`) unless explicitly required.
- Call `server.starttls()` **before** `server.login()`.

```python
import smtplib

SMTP_HOST = "smtp.gmail.com"
SMTP_PORT = 587  # STARTTLS

with smtplib.SMTP(SMTP_HOST, SMTP_PORT) as server:
    server.ehlo()
    server.starttls()      # ← Must come before login
    server.ehlo()
    server.login(SENDER_EMAIL, GMAIL_APP_PASSWORD)
    server.sendmail(SENDER_EMAIL, RECIPIENT_EMAIL, msg.as_string())
```

### 6.2 Multipart Email Format

- All outgoing emails **must** use `MIMEMultipart("alternative")` with both:
  - A `text/plain` fallback part.
  - A `text/html` primary part.
- Attach the `text/plain` part **first**, then the `text/html` part (per RFC 2046 — mail clients display the last part they understand).

### 6.3 Authentication Error Handling

- Catch `smtplib.SMTPAuthenticationError` explicitly and provide a user-friendly message that:
  - Confirms the credentials were rejected.
  - Reminds the user to use a **Gmail App Password**, not their real password.
  - Suggests re-checking the `.env` file.

```python
except smtplib.SMTPAuthenticationError:
    print(
        "[ERROR] SMTP authentication failed.\n"
        "  • Ensure you are using a Gmail App Password, not your real password.\n"
        "  • Verify SENDER_EMAIL and GMAIL_APP_PASSWORD in your .env file."
    )
```

- Also catch `smtplib.SMTPException` as a general fallback for other SMTP errors.
- Use a `with` statement (context manager) for the SMTP connection so it is always closed properly, even on error.

### 6.4 Email Headers

- Always set `From`, `To`, `Subject`, and `Date` headers explicitly.
- Use `email.utils.formatdate(localtime=True)` for the `Date` header.

---

## 7. LLM Prompt Rules

### 7.1 Model & Client

- Use **Google DiffusionGemma 26B** via the NVIDIA AI API, accessed through LangChain's `ChatNVIDIA` client.
- Store the model identifier as a constant:

```python
NVIDIA_MODEL_ID = "google/diffusion-gemma-26b"  # Update if the endpoint name changes
```

### 7.2 Formal Tone — Mandatory

- **All prompts** sent to the LLM must explicitly instruct the model to produce **formal, professional business English**.
- The system prompt or instruction must include clear directives such as:
  - "Write in a formal, professional tone."
  - "Use polite and respectful language appropriate for professional correspondence."
  - "Avoid slang, contractions, and casual language."

### 7.3 Prompt Structure

Use a consistent prompt template with a clear system role and user instruction:

```python
SYSTEM_PROMPT = (
    "You are a professional email writing assistant. "
    "Always write in formal, polite, and professional business English. "
    "Do not use casual language, slang, or contractions."
)

USER_PROMPT_TEMPLATE = (
    "Write a formal professional email about the following topic:\n\n"
    "{topic}\n\n"
    "Include a subject line, greeting, body, and sign-off."
)
```

### 7.4 Output Parsing

- Parse the LLM response to extract the **Subject** and **Body** separately before passing to `translate.py` and `email_sender.py`.
- If the LLM does not return a parsable subject line, default to a generic subject derived from the topic.

### 7.5 Temperature & Parameters

- Use a **low temperature** (e.g., `0.2`–`0.4`) for consistent, professional output.
- Set `max_tokens` to a reasonable limit (e.g., `1024`) to avoid runaway generation.

---

## 8. Contribution Rules

### 8.1 Version Control

- Use **Git** for version control.
- Commit messages must follow the **Conventional Commits** format:
  - `feat:` — new feature
  - `fix:` — bug fix
  - `docs:` — documentation changes
  - `refactor:` — code restructuring without behavior change
  - `chore:` — maintenance tasks (dependency updates, etc.)

### 8.2 Branch Naming

- Feature branches: `feat/<short-description>`
- Bug fix branches: `fix/<short-description>`
- Documentation branches: `docs/<short-description>`

### 8.3 Files That Must Never Be Committed

Add the following to `.gitignore`:

```gitignore
# Secrets
.env

# Python artifacts
__pycache__/
*.py[cod]
*.pyo
.venv/
venv/
env/

# HuggingFace model cache
*.bin
*.safetensors
models/

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db
```

### 8.4 Testing

- Before submitting any change, manually verify the full pipeline:
  1. `write_email.py` produces a well-formed formal email.
  2. `translate.py` produces a valid Japanese translation.
  3. `email_sender.py` successfully delivers the email (check recipient inbox and spam folder).
- Unit test any pure utility functions (text splitting, token counting, header parsing) if feasible.

### 8.5 README

- The project `README.md` must document:
  - Setup instructions (create `.env` from `.env.example`, install `requirements.txt`).
  - How to run each script individually and as a pipeline.
  - How to obtain an NVIDIA API key and a Gmail App Password.

---

*Last updated: 2026-10-11*
