# 📘 LEARNINGS.md — English-to-Japanese Email Translator

> **Project**: English-to-Japanese Email Translator  
> **Type**: College Minor Project  
> **Stack**: Python, LangChain, HuggingFace Transformers, NVIDIA AI API, SMTP

---

## 📁 Project Overview

This project is a three-script pipeline that:
1. **Drafts** a professional English email using an LLM (`write_email.py`)
2. **Translates** the email to Japanese using a local seq2seq model (`translate.py`)
3. **Sends** the translated HTML email via SMTP (`email_sender.py`)

---

## 🔑 Key Technical Learnings

### 1. LangChain Chains & Prompt Templates (`write_email.py`)

- **`ChatPromptTemplate`** lets you separate *system* instructions from *human* messages cleanly. System messages define the AI's role/persona; human messages carry the actual request.
- **`ChatNVIDIA`** from `langchain_nvidia_ai_endpoints` connects to NVIDIA-hosted models (e.g., Google DiffusionGemma 26B) via a simple API key — no local GPU needed for the LLM step.
- **`temperature=0.5`** is a balanced middle ground: low enough to keep emails professional and coherent, high enough to avoid robotic repetition.
- LangChain's **pipe operator (`|`)** chains components elegantly:  
  ```python
  chain = prompt | llm | output_parser
  result = chain.invoke({"topic": "project update"})
  ```
- The `StrOutputParser` extracts the raw string from the LLM's response object — easy to forget without it.

### 2. HuggingFace Transformers API (`translate.py`)

- **`AutoModelForSeq2SeqLM`** and **`AutoTokenizer`** are the go-to classes for loading any seq2seq model (like NLLB-200) without knowing the exact architecture in advance.
- The model is loaded once and reused — avoid reloading on every call (it's expensive: ~2–3 GB for NLLB-200).
- **`LangChain RunnableLambda`** wraps a plain Python function into a LangChain-compatible component, making it easy to plug local inference into a chain alongside API-based steps.
- Tokenizer output must be moved to the **same device** as the model (CPU vs. CUDA) — a common source of runtime errors.

### 3. SMTP Protocol (`email_sender.py`)

- **`smtplib.SMTP(host, port)`** opens a plain connection; **`.starttls()`** upgrades it to an encrypted channel. Always call `starttls()` before `login()` — sending credentials over plain TCP is a security risk.
- Port **587** is the standard for STARTTLS (submission port). Port 465 uses SSL from the start (`SMTP_SSL`).
- **`SMTPAuthenticationError`** almost always means one of: wrong password, 2FA enabled without an App Password, or "Less secure app access" disabled — not a code bug.
- Always call `server.quit()` or use a `with` block to close the SMTP connection properly.

### 4. MIME Email Construction (`email_sender.py`)

- **`MIMEMultipart('alternative')`** is the right container when sending both plain-text and HTML versions. Email clients pick the best version they support.
- **`MIMEText(html_content, 'html')`** wraps the HTML body. The HTML is attached *after* the plain-text part — clients render the last matching part they support (HTML > plain text).
- Styled `<div>` blocks inside the HTML body work in most desktop clients, but **avoid external CSS files or `<link>` tags** — they're stripped by many email clients. Inline styles are the safe choice.
- Set `msg['Subject']`, `msg['From']`, and `msg['To']` headers explicitly; some servers reject emails with missing headers.

### 5. PyTorch Inference (`translate.py`)

- **`torch.no_grad()`** disables gradient tracking during inference, reducing memory usage and speeding up forward passes. Always use it when you're not training.
- **`model.generate()`** is the standard seq2seq decoding entry point. Key parameters used:
  - `forced_bos_token_id` — forces the decoder to start with the target language token (e.g., `jpn_Jpan` for Japanese in NLLB-200).
  - `max_new_tokens` — caps output length to prevent runaway generation.
  - Beam search (default in `generate()`) produces more fluent translations than greedy decoding.

---

## 🧠 ML / NLP Concepts Learned

### Seq2Seq Models
- Seq2seq (sequence-to-sequence) models use an **encoder-decoder** architecture. The encoder reads the source sentence into a context representation; the decoder generates the target sentence from it.
- Models like **NLLB-200** (No Language Left Behind) are trained on 200+ languages simultaneously, learning shared multilingual representations.

### Tokenization
- Tokenizers split text into subword units (BPE, SentencePiece) rather than whole words. This handles rare words and morphologically rich languages (like Japanese) gracefully.
- For Japanese, tokenization is especially important because there are no spaces between words — the model relies entirely on subword segmentation.
- `tokenizer.decode(output_ids, skip_special_tokens=True)` removes padding and language tokens from the output before returning the translated string.

### Beam Search
- Instead of greedily picking the single best next token, beam search keeps **k candidate sequences** (beams) at each step and returns the globally best one at the end.
- A beam width of 4–5 is a good default trade-off between quality and speed.
- Higher beam width = better quality but slower and more memory-intensive.

### Language Tokens in Multilingual Models
- NLLB-200 uses **special BCP-47-style tokens** (e.g., `jpn_Jpan`, `eng_Latn`) to signal the target language to the decoder.
- `forced_bos_token_id = tokenizer.lang_code_to_id["jpn_Jpan"]` is the correct way to look up this token ID — hardcoding integer IDs is fragile across model versions.
- Setting the **source language** on the tokenizer (`tokenizer.src_lang = "eng_Latn"`) ensures the encoder interprets the input correctly.

---

## 🚧 Challenges Faced & Solutions

| Challenge | What Went Wrong | Solution |
|---|---|---|
| SMTP login failure | Used Gmail password directly | Generated a Gmail **App Password** (required when 2FA is on) |
| Wrong device error in PyTorch | Tokenizer output on CPU, model on CUDA | Added `.to(model.device)` on tokenizer output tensors |
| Garbled Japanese output | `skip_special_tokens=False` | Set `skip_special_tokens=True` in `tokenizer.decode()` |
| Email HTML not rendering | Used `MIMEText(body, 'plain')` | Switched to `MIMEText(html, 'html')` inside `MIMEMultipart('alternative')` |
| LLM output had extra metadata | Forgot `StrOutputParser` | Appended `| StrOutputParser()` to the chain |
| Model loading was slow | Reloaded model on every call | Moved model/tokenizer loading to module-level (outside functions) |
| `forced_bos_token_id` KeyError | Used wrong language code string | Verified exact key using `tokenizer.lang_code_to_id.keys()` |

---

## ✅ Best Practices Discovered

### General Python
- **Separate concerns**: one script per major responsibility (draft → translate → send) makes debugging and testing much easier.
- Use **`.env` files + `python-dotenv`** to manage API keys and email credentials. Never hardcode secrets.
- Add **`try/except`** blocks around network calls (SMTP, API) — they fail often and with unhelpful default errors.

### LangChain
- Use **`ChatPromptTemplate.from_messages([...])`** over raw string templates for multi-turn or role-based prompts — it's cleaner and more maintainable.
- Keep prompt templates in **separate `.txt` or `.yaml` files** for long prompts to avoid cluttering Python code.
- Always test the chain with `.invoke()` before integrating — it surfaces parsing/output issues early.

### HuggingFace / PyTorch
- Always call **`model.eval()`** before inference to disable dropout and batch normalization training behavior.
- Use **`torch.no_grad()`** as a context manager (not a decorator) for inference loops — it's cleaner.
- Pin model versions with a specific revision hash in `from_pretrained()` for reproducibility in a project submission.

### SMTP / Email
- Test with **Mailtrap** or **Mailhog** (local SMTP test servers) before sending real emails — avoids accidentally spamming contacts.
- Always validate email addresses before calling `sendmail()`.
- Log sent emails (timestamp, recipient, subject) to a local file for traceability.

---

## ❌ Mistakes & What Was Learned

### Mistake 1: Hardcoding credentials
**What happened**: API key and email password were written directly in the script.  
**Risk**: Would be exposed if the file was shared or committed to Git.  
**Lesson**: Always use environment variables. Add `.env` to `.gitignore` immediately.

### Mistake 2: Not calling `model.eval()`
**What happened**: Translations were slightly non-deterministic across runs even with the same input.  
**Why**: Dropout layers behave differently in training vs. eval mode.  
**Lesson**: Always switch to eval mode before inference with `model.eval()`.

### Mistake 3: Reloading the model inside the translation function
**What happened**: Each call to `translate()` took 10–15 seconds to reload the model from disk.  
**Lesson**: Load heavy models once at module import time. Use global or singleton patterns for ML models.

### Mistake 4: Ignoring the `forced_bos_token_id` parameter
**What happened**: Without it, the model sometimes generated output in a random language.  
**Why**: Multilingual models need to be told the target language explicitly.  
**Lesson**: For NLLB-200, always set both `tokenizer.src_lang` and `forced_bos_token_id` for reliable language control.

### Mistake 5: Using `MIMEText` without `MIMEMultipart`
**What happened**: HTML tags appeared as raw text in some email clients.  
**Lesson**: Always use `MIMEMultipart('alternative')` when sending HTML email, and attach plain-text fallback first.

### Mistake 6: Not handling SMTP exceptions granularly
**What happened**: Any SMTP error printed a generic Python traceback — unhelpful for debugging.  
**Lesson**: Catch `smtplib.SMTPAuthenticationError`, `smtplib.SMTPConnectError`, and `smtplib.SMTPException` separately with specific error messages.

---

## 💡 Suggested Improvements

### Short-term (Easy Wins)
- [ ] Add a **CLI interface** (`argparse`) so the email topic/recipient can be passed as arguments instead of hardcoded values.
- [ ] Add **logging** (Python `logging` module) instead of `print()` statements for cleaner output and log files.
- [ ] Validate the generated email with a **word count / quality check** before sending.
- [ ] Add a **dry-run mode** (`--dry-run`) that prints the email without sending it.

### Medium-term (Skill Builders)
- [ ] Add **translation quality scoring** using BLEU or ChrF metrics with a reference translation set.
- [ ] Support **multiple target languages** — parameterize the `forced_bos_token_id` lookup.
- [ ] Build a simple **Gradio or Streamlit UI** so non-technical users can use the tool through a browser.
- [ ] Add **email template selection** — formal, casual, apologetic — via the LLM system prompt.
- [ ] Cache translated results for identical source texts using a simple `shelve` or SQLite store.

### Long-term (Stretch Goals)
- [ ] Replace the local NLLB model with a **fine-tuned model on email-domain Japanese** for better business-register translations.
- [ ] Integrate **DeepL API** as a translation fallback for higher accuracy on critical emails.
- [ ] Add a **feedback loop**: let the recipient reply, detect sentiment, and auto-draft a follow-up.
- [ ] Package the whole pipeline as a **FastAPI service** with endpoints for each stage.

---

## 📚 Resources That Helped

| Topic | Resource |
|---|---|
| LangChain chains | [LangChain Expression Language (LCEL) docs](https://python.langchain.com/docs/expression_language/) |
| NLLB-200 model | [Meta AI NLLB-200 on HuggingFace](https://huggingface.co/facebook/nllb-200-distilled-600M) |
| ChatNVIDIA integration | [NVIDIA AI Endpoints LangChain docs](https://python.langchain.com/docs/integrations/chat/nvidia_ai_endpoints/) |
| SMTP & STARTTLS | [Python smtplib docs](https://docs.python.org/3/library/smtplib.html) |
| MIME email construction | [Python email.mime docs](https://docs.python.org/3/library/email.mime.html) |
| PyTorch inference best practices | [PyTorch no_grad docs](https://pytorch.org/docs/stable/generated/torch.no_grad.html) |
| Beam search explained | [HuggingFace blog: How to generate text](https://huggingface.co/blog/how-to-generate) |

---

*Last updated: 2026-10-11*
