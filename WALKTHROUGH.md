# 🗾 English-to-Japanese Email Translator — Code Walkthrough

> **Project Type:** College Minor Project  
> **Language:** Python 3  
> **Key Libraries:** LangChain, NVIDIA AI Endpoints, HuggingFace Transformers, smtplib

---

## 📐 Overall Pipeline Flow

```
┌─────────────────────┐
│    write_email.py   │
│  (LangChain + LLM)  │
│  Generates English  │
│       Email         │
└────────┬────────────┘
         │ English email text (string)
         ▼
┌─────────────────────┐
│    translate.py     │
│  (HuggingFace NLLB) │
│  English → Japanese │
└────────┬────────────┘
         │ Japanese email text (string)
         ▼
┌─────────────────────┐
│   email_sender.py   │
│  (smtplib + MIME)   │
│  Sends bilingual    │
│   HTML email        │
└─────────────────────┘
```

The three modules form a **linear pipeline**:
1. **Write** — An LLM drafts a formal English email from a topic prompt.
2. **Translate** — A local seq2seq model translates it to Japanese.
3. **Send** — An SMTP client packages both versions into a styled HTML email and delivers it.

---

## 📄 Module 1: `write_email.py`

### Full Source

```python
from dotenv import load_dotenv
from langchain_core.prompts import ChatPromptTemplate
from langchain_nvidia_ai_endpoints import ChatNVIDIA
load_dotenv()
import warnings
warnings.filterwarnings("ignore", category=UserWarning)

llm = ChatNVIDIA(
    base_url="https://integrate.api.nvidia.com/v1/",
    model="google/diffusiongemma-26b-a4b-it",
    temperature=0.5
)

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a professional email writer. Write in a formal tone."),
    ("human", "Write a short email about: {topic}"),
])

chain = prompt | llm
result = chain.invoke({"topic": "informing client for project completion."})
print(result.content)
```

### Line-by-Line Walkthrough

| Lines | Code | Explanation |
|-------|------|-------------|
| 1–3 | `from dotenv import load_dotenv` … | Imports: `load_dotenv` reads `.env` files; `ChatPromptTemplate` structures the prompt; `ChatNVIDIA` wraps NVIDIA's hosted inference API. |
| 4 | `load_dotenv()` | Loads environment variables (e.g., `NVIDIA_API_KEY`) from a `.env` file into `os.environ` so they're available at runtime. |
| 5–6 | `warnings.filterwarnings(…)` | Suppresses non-critical `UserWarning` messages from LangChain internals to keep console output clean. |
| 8–12 | `llm = ChatNVIDIA(…)` | Instantiates the NVIDIA-hosted **DiffusionGemma 26B** model. `temperature=0.5` balances creativity vs. determinism — lower values = more predictable output. |
| 14–17 | `prompt = ChatPromptTemplate…` | Builds a two-turn prompt template. The `system` role sets the persona; the `human` role contains `{topic}` — a **placeholder** replaced at invocation time. |
| 19 | `chain = prompt \| llm` | Uses the **LangChain pipe operator** (`\|`) to compose a chain: the prompt is formatted first, then passed directly to the LLM. |
| 20 | `result = chain.invoke(…)` | Runs the chain. `{"topic": "informing client for project completion."}` fills in the template placeholder. Returns an `AIMessage` object. |
| 21 | `print(result.content)` | Extracts the plain text string from the `AIMessage` and prints it. |

### 🔑 Key Concept: The LangChain Pipe Operator (`|`)

LangChain uses the `|` operator to compose **Runnables** into a chain, similar to Unix pipes:

```
input_dict  →  [ChatPromptTemplate]  →  [ChatNVIDIA LLM]  →  AIMessage
```

Each component's output becomes the next component's input. This makes chains declarative, reusable, and easy to extend (e.g., add `| output_parser` at the end).

### Example Output

```
Subject: Project Completion Notification

Dear [Client Name],

I am pleased to inform you that the project has been successfully completed
as per the agreed timeline and specifications. All deliverables have been
finalized and are ready for your review.

Please do not hesitate to reach out should you have any questions.

Sincerely,
[Your Name]
```

---

## 📄 Module 2: `translate.py`

### Conceptual Source

```python
from transformers import AutoModelForSeq2SeqLM, AutoTokenizer
import torch
from langchain_core.runnables import RunnableLambda

# Load local NLLB model and tokenizer
tokenizer = AutoTokenizer.from_pretrained("path/to/local/nllb-model")
model = AutoModelForSeq2SeqLM.from_pretrained("path/to/local/nllb-model")

def translate_to_japanese(english_text: str) -> str:
    # Tokenize input with source language tag
    inputs = tokenizer(
        english_text,
        return_tensors="pt",
        src_lang="eng_Latn"
    )

    # Get the token ID for the Japanese BCP-47 language tag
    jpn_token_id = tokenizer.convert_tokens_to_ids("jpn_Jpan")

    # Generate translation
    output_tokens = model.generate(
        **inputs,
        forced_bos_token_id=jpn_token_id,
        max_new_tokens=256
    )

    # Decode output tokens back to a string
    translation = tokenizer.batch_decode(
        output_tokens,
        skip_special_tokens=True
    )[0]

    return translation

# Wrap in a LangChain Runnable for pipeline compatibility
translate_runnable = RunnableLambda(translate_to_japanese)
```

### Line-by-Line Walkthrough

| Component | Explanation |
|-----------|-------------|
| `AutoTokenizer.from_pretrained(…)` | Loads the tokenizer that converts text ↔ token IDs. The NLLB tokenizer is multilingual and knows language codes like `eng_Latn` and `jpn_Jpan`. |
| `AutoModelForSeq2SeqLM.from_pretrained(…)` | Loads the **NLLB (No Language Left Behind)** model weights from a local directory. Seq2Seq = encoder-decoder architecture — encoder reads English, decoder writes Japanese. |
| `src_lang="eng_Latn"` | Tells the tokenizer the **source language** using BCP-47 + script code. `eng` = English, `Latn` = Latin script. |
| `convert_tokens_to_ids("jpn_Jpan")` | Looks up the integer token ID for the Japanese language tag (`jpn` = Japanese, `Jpan` = Japanese script). This ID is passed as a generation constraint. |
| `forced_bos_token_id=jpn_token_id` | Forces the **first generated token** to be the Japanese language marker, telling the decoder to produce Japanese output. This is NLLB's translation control mechanism. |
| `max_new_tokens=256` | Caps the output length to prevent runaway generation. |
| `skip_special_tokens=True` | Strips `<s>`, `</s>`, language tags, and padding from the decoded output, returning clean text. |
| `RunnableLambda(…)` | Wraps the plain Python function as a LangChain `Runnable` so it can be composed into chains with `\|`. |

### 🔑 Key Concept: Seq2Seq Translation

```
English Input
     │
     ▼
┌─────────────┐
│   Encoder   │  Reads and understands the English sentence
└──────┬──────┘
       │ context vectors (hidden states)
       ▼
┌─────────────┐
│   Decoder   │  Generates Japanese tokens one at a time,
│             │  guided by forced_bos_token_id = jpn_Jpan
└──────┬──────┘
       │
       ▼
Japanese Output
```

### Data Flow

```
"Project has been completed..."   (English string)
          │
          ▼  tokenizer(src_lang="eng_Latn")
[101, 20045, 44, 112, ...]        (input_ids tensor)
          │
          ▼  model.generate(forced_bos_token_id=jpn_token)
[250149, 3891, 74, ...]           (output token IDs)
          │
          ▼  tokenizer.batch_decode(skip_special_tokens=True)
"プロジェクトが完了しました..."       (Japanese string)
```

### Example Input/Output

| Stage | Value |
|-------|-------|
| **Input** | `"The project has been successfully completed as per the agreed timeline."` |
| **Output** | `"プロジェクトは合意されたスケジュール通りに正常に完了しました。"` |

---

## 📄 Module 3: `email_sender.py`

### Conceptual Source

```python
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText

def send_email(
    sender_email: str,
    sender_password: str,
    recipient_email: str,
    subject: str,
    japanese_body: str,
    english_body: str,
    smtp_host: str = "smtp.gmail.com",
    smtp_port: int = 587
):
    # Create a multipart/alternative email container
    msg = MIMEMultipart("alternative")
    msg["Subject"] = subject
    msg["From"] = sender_email
    msg["To"] = recipient_email

    # Plain text fallback
    plain_text = f"{japanese_body}\n\n---\n\n{english_body}"
    part_plain = MIMEText(plain_text, "plain", "utf-8")

    # Styled HTML body
    html_body = f"""
    <html><body>
      <div style="background-color: #ffb6c1; padding: 20px; margin-bottom: 10px;">
        <h2>日本語 (Japanese)</h2>
        <p>{japanese_body}</p>
      </div>
      <div style="background-color: #ffffe0; padding: 20px;">
        <h2>English (Original)</h2>
        <p>{english_body}</p>
      </div>
    </body></html>
    """
    part_html = MIMEText(html_body, "html", "utf-8")

    # Attach both parts (HTML takes priority in modern clients)
    msg.attach(part_plain)
    msg.attach(part_html)

    try:
        with smtplib.SMTP(smtp_host, smtp_port) as server:
            server.ehlo()
            server.starttls()          # Upgrade to encrypted TLS connection
            server.login(sender_email, sender_password)
            server.sendmail(sender_email, recipient_email, msg.as_string())
            print("Email sent successfully!")

    except smtplib.SMTPAuthenticationError:
        print("Authentication failed. Check your email/password or App Password.")
    except smtplib.SMTPException as e:
        print(f"SMTP error occurred: {e}")
    except Exception as e:
        print(f"Unexpected error: {e}")
```

### Line-by-Line Walkthrough

| Component | Explanation |
|-----------|-------------|
| `MIMEMultipart("alternative")` | Creates an email container with **two alternative representations** — plain text and HTML. Email clients pick the best one they can render. |
| `MIMEText(plain_text, "plain", "utf-8")` | The plain text fallback. `utf-8` encoding is critical to correctly transmit Japanese Unicode characters. |
| `MIMEText(html_body, "html", "utf-8")` | The rich HTML version. The `charset="utf-8"` ensures Japanese characters aren't garbled. |
| `msg.attach(part_plain)` then `msg.attach(part_html)` | Both parts are attached. Per MIME spec, the **last attachment takes priority**, so HTML renders first in modern clients. |
| `background-color: #ffb6c1` (pink div) | Contains the Japanese translation, styled with a pink background for visual distinction. |
| `background-color: #ffffe0` (yellow div) | Contains the original English email on a yellow background. |
| `smtplib.SMTP(smtp_host, smtp_port)` | Opens a TCP connection to Gmail's SMTP server on port `587` (the standard STARTTLS port). |
| `server.starttls()` | **Upgrades** the plain TCP connection to TLS encryption before any credentials are sent. Essential for security. |
| `server.login(…)` | Authenticates with Gmail. For accounts with 2FA, this requires a **Gmail App Password**, not the account password. |
| `server.sendmail(…)` | Transmits the fully formatted email. |
| `SMTPAuthenticationError` | Caught specifically to give a helpful message about App Passwords. |
| `SMTPException` | Catches all other SMTP-level errors (connection timeout, relay rejection, etc.). |
| `Exception` | Final catch-all for unexpected runtime errors. |

### 🔑 Key Concept: MIME Email Structure

```
MIMEMultipart("alternative")
├── MIMEText("plain text version", "plain")   ← Fallback for basic clients
└── MIMEText("<html>...</html>", "html")      ← Preferred by modern clients
     ├── <div style="pink">  Japanese body  </div>
     └── <div style="yellow"> English body  </div>
```

### Example Email Rendered in Client

```
┌──────────────────────────────────────────────┐
│  Subject: Project Completion Notification    │
│  From: sender@gmail.com                     │
│  To: client@example.com                     │
├──────────────────────────────────────────────┤
│ ┌──────────────────────────────────────────┐ │
│ │  🌸 日本語 (Japanese)         [pink bg] │ │
│ │  プロジェクトが完了しました...            │ │
│ └──────────────────────────────────────────┘ │
│ ┌──────────────────────────────────────────┐ │
│ │  English (Original)          [yellow bg] │ │
│ │  The project has been completed...       │ │
│ └──────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
```

---

## 🔗 How All Three Modules Connect

### Integration Pattern

```python
# orchestrator.py (conceptual glue code)
from write_email import chain as email_chain
from translate import translate_runnable
from email_sender import send_email

# Stage 1: Generate English email
result = email_chain.invoke({"topic": "informing client for project completion."})
english_email = result.content

# Stage 2: Translate to Japanese
japanese_email = translate_runnable.invoke(english_email)

# Stage 3: Send bilingual email
send_email(
    sender_email="you@gmail.com",
    sender_password="your-app-password",
    recipient_email="client@example.com",
    subject="Project Completion Notification / プロジェクト完了通知",
    japanese_body=japanese_email,
    english_body=english_email
)
```

### Data Flow Summary

```
Topic String
   "informing client for project completion."
              │
              ▼  write_email.py (LangChain + NVIDIA DiffusionGemma)
   English Email Text
   "Dear Client, I am pleased to inform you..."
              │
              ▼  translate.py (HuggingFace NLLB seq2seq model)
   Japanese Email Text
   "拝啓、プロジェクトが完了したことをお知らせします..."
              │
              ▼  email_sender.py (smtplib + MIME)
   Bilingual HTML Email → Delivered to recipient's inbox
```

---

## 📦 Key Concepts Summary

| Concept | Module | Description |
|---------|--------|-------------|
| **LangChain Pipe `\|`** | `write_email.py` | Composes Runnables into a chain; each step's output feeds the next |
| **ChatPromptTemplate** | `write_email.py` | Parameterized prompt with `{placeholders}` for dynamic input |
| **NVIDIA AI Endpoints** | `write_email.py` | Hosted inference API for large language models |
| **Seq2Seq Architecture** | `translate.py` | Encoder reads source, decoder generates target language |
| **NLLB Language Codes** | `translate.py` | BCP-47 codes like `eng_Latn`, `jpn_Jpan` control translation direction |
| **forced_bos_token_id** | `translate.py` | Forces first output token to be target language marker |
| **RunnableLambda** | `translate.py` | Wraps plain Python functions for LangChain pipeline compatibility |
| **MIMEMultipart** | `email_sender.py` | Email with multiple content types (plain + HTML) |
| **STARTTLS** | `email_sender.py` | Upgrades SMTP connection to TLS for secure credential transmission |
| **App Password** | `email_sender.py` | Gmail-specific password for SMTP access when 2FA is enabled |

---

## 🚀 Running the Project

### Prerequisites

```bash
pip install langchain langchain-nvidia-ai-endpoints python-dotenv transformers torch
```

### `.env` File

```env
NVIDIA_API_KEY=your_nvidia_api_key_here
```

### Execution Order

```bash
# Run each stage independently, or wire them together in an orchestrator script
python write_email.py     # Test LLM email generation
python translate.py       # Test translation pipeline
python email_sender.py    # Test email delivery
```

> [!TIP]
> For Gmail SMTP, go to **Google Account → Security → App Passwords** and generate a dedicated password for this app. Never use your main account password.

> [!NOTE]
> The NLLB model is loaded from a **local directory**. Download `facebook/nllb-200-distilled-600M` (or larger variant) from HuggingFace and set the correct local path in `translate.py`.
