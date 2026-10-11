# Skills Matrix — English-to-Japanese Email Translator

> **Project:** English-to-Japanese Email Translator  
> **Type:** College Minor Project  
> **Stack:** Python · LangChain · HuggingFace Transformers · SMTP  

---

## 1. Technical Skills

### 1.1 Programming & Software Engineering

| Skill | Proficiency | Evidence |
|---|---|---|
| **Python 3** | ★★★★☆ Advanced | All three scripts use idiomatic Python: type-aware variable handling, modular function design, environment variable management |
| **Modular script architecture** | ★★★☆☆ Intermediate | Project split into three single-responsibility scripts (`write_email.py`, `translate.py`, `email_sender.py`) |
| **Environment & secrets management** | ★★★☆☆ Intermediate | NVIDIA API keys loaded from environment; avoids hard-coding credentials |
| **Error handling & robustness** | ★★★☆☆ Intermediate | SMTP and model inference wrapped with appropriate exception handling |

---

### 1.2 AI & Machine Learning

| Skill | Proficiency | Evidence |
|---|---|---|
| **Large Language Model (LLM) integration** | ★★★★☆ Advanced | `write_email.py` calls NVIDIA AI API to access Google DiffusionGemma 26B, a state-of-the-art generative model |
| **LangChain framework** | ★★★☆☆ Intermediate | Uses `ChatNVIDIA` from LangChain to construct prompts and manage model interaction |
| **Prompt engineering** | ★★★☆☆ Intermediate | Crafts structured system + user prompts that constrain the model to generate formal, professional email prose |
| **Seq2Seq machine translation** | ★★★★☆ Advanced | `translate.py` loads an NLLB-style `AutoModelForSeq2SeqLM` locally; manually sets source/target language tokens (`jpn_Jpan`) |
| **HuggingFace Transformers** | ★★★★☆ Advanced | Uses `AutoTokenizer` and `AutoModelForSeq2SeqLM` for tokenization, inference, and decoding |
| **PyTorch inference** | ★★★☆☆ Intermediate | Runs forward pass through the seq2seq model using PyTorch tensors; handles `generate()` parameters |
| **NLP pipeline design** | ★★★★☆ Advanced | Two-stage pipeline: generative LLM → neural machine translation, each stage cleanly isolated |

---

### 1.3 Email & Communication Systems

| Skill | Proficiency | Evidence |
|---|---|---|
| **SMTP protocol** | ★★★☆☆ Intermediate | `email_sender.py` establishes authenticated SMTP sessions to deliver mail |
| **MIME multipart email composition** | ★★★★☆ Advanced | Constructs `multipart/alternative` messages with both `text/plain` and `text/html` parts |
| **Bilingual email formatting** | ★★★☆☆ Intermediate | Japanese translation is set as the primary MIME part; English original included as a reference block |
| **HTML email templating** | ★★★☆☆ Intermediate | HTML part styled for readability; language sections labelled and structured |

---

### 1.4 Libraries & Frameworks

| Library / Tool | Role in Project |
|---|---|
| `langchain-nvidia-ai-endpoints` | LLM access via NVIDIA AI API (ChatNVIDIA wrapper) |
| `transformers` (HuggingFace) | Seq2Seq tokenizer + model for English→Japanese translation |
| `torch` (PyTorch) | Tensor operations and model inference backend |
| `smtplib` + `email` (stdlib) | SMTP session management and MIME message construction |
| `python-dotenv` / `os.environ` | Secrets and configuration management |

---

## 2. Domain Skills

### 2.1 Email Automation

- Designed an **end-to-end automated email pipeline**: topic prompt → generated email body → translated text → formatted & delivered message.
- Applied MIME standards for cross-client compatibility (plain-text fallback + HTML primary).
- Implemented **bilingual message structure** suitable for professional cross-cultural correspondence.

### 2.2 Machine Translation & NLP

- Selected and deployed a **multilingual seq2seq model** (NLLB architecture) capable of high-quality Japanese output.
- Applied correct **language token selection** (`jpn_Jpan` for Japanese in Japanese script) — a non-trivial detail that directly affects translation quality.
- Understood trade-offs between **local inference** (privacy, no API cost) vs. cloud LLM (quality, latency).

### 2.3 Generative AI Integration

- Integrated a **26B-parameter LLM** through an API service, demonstrating awareness of large-model capabilities and constraints.
- Used **LangChain abstractions** to keep the LLM provider swappable without rewriting business logic.
- Formulated prompts that reliably produce **structured, professional output** (subject line + body, correct tone).

---

## 3. Soft Skills

### 3.1 Project Planning & Design

| Skill | Evidence |
|---|---|
| **Separation of concerns** | Three scripts each own one responsibility; clean interfaces between generation, translation, and delivery |
| **Technology selection** | Deliberate choice of cloud LLM for generation quality and local model for translation privacy/cost |
| **Scope management** | Delivered a functional three-component system within a minor-project scope |

### 3.2 Documentation

| Skill | Evidence |
|---|---|
| **Technical writing** | `SKILLS.md` (this document) and inline code comments explain design rationale |
| **Audience awareness** | README-style documentation structured for a technical reviewer or academic evaluator |

### 3.3 Research & Self-Learning

| Skill | Evidence |
|---|---|
| **Library research** | Independently identified and integrated LangChain NVIDIA endpoints, HuggingFace NLLB models, and MIME email APIs |
| **Cross-domain synthesis** | Combined generative AI, NLP/translation, and email infrastructure — three distinct technical domains — into one coherent application |

---

## 4. Architecture Overview

```
User Prompt (topic)
        │
        ▼
┌─────────────────────────┐
│   write_email.py        │  LangChain + NVIDIA API
│   DiffusionGemma 26B    │  → Formal English email
└───────────┬─────────────┘
            │  English email text
            ▼
┌─────────────────────────┐
│   translate.py          │  HuggingFace Transformers + PyTorch
│   NLLB seq2seq model    │  → Japanese translation (jpn_Jpan)
└───────────┬─────────────┘
            │  Japanese + English texts
            ▼
┌─────────────────────────┐
│   email_sender.py       │  smtplib + email (MIME multipart)
│   SMTP delivery         │  → Bilingual HTML/plain-text email
└─────────────────────────┘
```

---

## 5. Proficiency Key

| Symbol | Level | Description |
|---|---|---|
| ★☆☆☆☆ | Awareness | Familiar with the concept; used with guidance |
| ★★☆☆☆ | Beginner | Applied in a limited, controlled context |
| ★★★☆☆ | Intermediate | Used independently; understands internals |
| ★★★★☆ | Advanced | Applied with depth; aware of trade-offs |
| ★★★★★ | Expert | Can teach, extend, and optimize at a professional level |
