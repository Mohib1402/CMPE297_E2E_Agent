# 💳 Personal Expense Assistant – Multimodal Agent (Gemini + SQLite + Gradio)

This project implements a **full-stack multimodal agent** in a single Google Colab notebook:

* **Frontend:** Gradio chat UI with receipt image upload
* **Backend / Agent:** Python functions calling the **Gemini API** via `google-genai`
* **Database:** SQLite (`expenses.db`) for persistent expense storage
* **RAG:** Questions are answered based on **retrieved expense records** from the database

The agent can:

* Read and parse **receipt images** (multimodal)
* Store structured expense records in a **database**
* Answer questions about past spending using **retrieval-augmented generation (RAG)**

---

## 📺 Demo Video

A detailed walkthrough shows:

* Code structure: frontend, backend, DB, and RAG logic
* Running the Colab notebook end-to-end
* Uploading receipts and asking questions about spending

**Video link:** [DEMO](https://drive.google.com/file/d/1Oup11Q4DMp2SxrD8RchCvxee6MH8axh4/view?usp=sharing)

---

## 📓 Colab Notebook

You can run the full app directly in Google Colab:

**Colab link:** [E2E_Agent](https://colab.research.google.com/drive/15u4FD0PGudqO9sxCtVALRJ66VhBeiDCS?usp=sharing)

---

## 🧠 High-Level Behavior

1. **Receipt understanding (multimodal)**

   * User uploads a **receipt image**.
   * `extract_expense_from_receipt(image)` calls Gemini (vision) to extract:

     * `date` (ISO `yyyy-mm-dd`)
     * `merchant`
     * `category` (e.g., groceries, restaurant, coffee, transport, shopping, other)
     * `total`
     * `currency`
     * `notes` (short summary)

2. **Database storage (backend + DB)**

   * The parsed expense is stored in **SQLite** as a row in the `expenses` table via `insert_expense(...)`.
   * The original JSON from Gemini is also stored in a `raw_json` column for traceability.

3. **RAG over expenses**

   * For questions like *“How much have I spent on groceries?”*,
     `answer_question_over_expenses(user_id, question)`:

     * Loads recent expenses from SQLite using `get_recent_expenses(...)`.
     * Embeds them as JSON context in a prompt.
     * Asks Gemini to answer **using only that data**.

4. **Frontend UI (Gradio)**

   * A simple web-style interface where the user can:

     * Upload a receipt and click **Send** to store it.
     * Type natural-language questions about their spending in the text box.

---

## 🧰 Tech Stack

* **Language:** Python (Colab / Jupyter)
* **LLM:** Gemini via `google-genai`

  * `gemini-2.5-flash` for multimodal (image + text)
  * `gemini-2.0-flash-001` for text-only QA
* **Database:** SQLite (`sqlite3`)
* **Frontend:** Gradio
* **Image handling:** Pillow (`PIL.Image`)

---

## 📁 Repository Structure

Suggested layout:

```text
multimodal-expense-assistant/
├── README.md
├── requirements.txt
└── notebook/
    └── personal_expense_assistant.ipynb
```

* `notebook/personal_expense_assistant.ipynb`
  Main Colab notebook containing:

  * Gemini client setup
  * SQLite schema + helper functions
  * Multimodal receipt parsing
  * RAG over expenses
  * Gradio UI

* `requirements.txt` (example):

```text
google-genai
gradio
pillow
```

---

## ⚙️ Setup & Running

### 1. Clone the repo

```bash
git clone https://github.com/YOUR_USERNAME/multimodal-expense-assistant.git
cd multimodal-expense-assistant
```

### 2. Install dependencies (local)

```bash
pip install -r requirements.txt
```

> In Colab, the notebook already includes an `!pip install` cell.

### 3. Get a Gemini API key

1. Go to **Google AI Studio**.
2. Create a **Gemini API key**.
3. Copy the key.

### 4. Configure the key in the notebook

In the setup cell:

```python
import os
from google import genai

GEMINI_API_KEY = "YOUR_GEMINI_API_KEY_HERE"  # ← replace with your key
os.environ["GEMINI_API_KEY"] = GEMINI_API_KEY
client = genai.Client(api_key=GEMINI_API_KEY)
```

> **Important:** Do **not** commit your real API key to GitHub. Use a placeholder string in the repo.

### 5. Run the notebook

1. Open `notebook/personal_expense_assistant.ipynb` in Colab or Jupyter.

2. Run all cells from top to bottom:

   * Install dependencies
   * Set up the Gemini client
   * Create and initialize `expenses.db`
   * Define helper functions
   * Launch the Gradio app

3. At the bottom, a Gradio link will appear (for example, `https://xxxx.gradio.live`).

---

## 🔑 Key Components

### SQLite DB & Access Layer

* Table `expenses` with columns:

  * `id, user_id, date, merchant, category, total, currency, notes, raw_json, created_at`

* Helper functions:

  * `insert_expense(user_id: str, expense: dict) -> int`

    * Inserts a parsed expense into the database.
  * `get_recent_expenses(user_id: str, limit: int = 50) -> list[dict]`

    * Returns recent expenses for that user as dictionaries.

### Multimodal Receipt Parsing

* `extract_expense_from_receipt(image: Image.Image) -> dict`

  * Converts the uploaded receipt image to PNG bytes.
  * Calls `gemini-2.5-flash` with:

    * A strict prompt that requests **JSON only**.
    * Both text and image as `google.genai.types.Part`.
  * Uses a regex to extract the JSON object from Gemini’s response.
  * Returns a dictionary with keys:

    * `date`, `merchant`, `category`, `total`, `currency`, `notes`
  * If parsing fails, falls back to storing the raw model text in `notes`.

### RAG Over Expenses

* `answer_question_over_expenses(user_id: str, question: str, max_rows: int = 50) -> str`

  * Loads recent expenses using `get_recent_expenses`.
  * Serializes them as JSON and embeds them in the prompt context.
  * Calls `gemini-2.0-flash-001` to answer the user’s question **based only on those expenses**.
  * If there are no expenses yet, replies that the user should upload receipts first.

### Gradio Frontend

* `chat_fn(message, history, image)`

  * If `image` is provided:

    * Calls `extract_expense_from_receipt`.
    * Stores the parsed expense via `insert_expense`.
    * Appends a summary (date, merchant, category, total) to the chat history.
  * If only `message` is provided:

    * Calls `answer_question_over_expenses`.
    * Appends user and assistant messages as a tuple `(user, assistant)`.

* Gradio layout:

  * `Chatbot` for conversation.
  * `Textbox` for text messages.
  * `Image` for receipt upload.
  * `Button` wired to `chat_fn`.

---

## 🧪 How to Use the App

1. **Upload a receipt**

   * Click **Upload receipt (optional)** and choose an image file.
   * Optionally type a short message like “Store this receipt”.
   * Click **Send**.
   * The assistant will:

     * Extract expense fields from the receipt using Gemini.
     * Store the expense row in `expenses.db`.
     * Show a summary with date, merchant, category, and total.

2. **Ask questions about your spending**

   * Example questions:

     * “How much have I spent in total?”
     * “How much have I spent on coffee?”
     * “Summarize my spending by category.”
   * The backend:

     * Retrieves expenses for the demo user.
     * Sends them as JSON context to Gemini.
     * Returns a natural-language answer grounded in the stored rows.

3. **Inspect stored expenses (from the notebook)**

   Run this helper to see what’s in the database:

   ```python
   for row in get_recent_expenses("demo-user-001", limit=10):
       print(row)
   ```

---

## 🚧 Limitations & Possible Extensions

**Limitations (current version):**

* Single hard-coded user ID (`demo-user-001`); no authentication or multi-user support.
* SQLite database is local to the runtime (e.g., Colab session), not a shared cloud DB.
* Receipt extraction quality depends on image quality and the model’s OCR performance.

**Possible extensions:**

* Add basic user authentication and per-user databases.
* Add charts (e.g., spending over time or by category).
* Export expense data as CSV.
* Move the Gradio app into a standalone `app.py` for local or cloud deployment.

---

## ✅ Assignment Alignment

This project satisfies the assignment requirements:

* **Multimodal agent:** Processes both text and receipt images using Gemini.
* **Frontend:** Gradio-based chat UI with image upload.
* **Backend:** Python logic that orchestrates Gemini calls, DB operations, and RAG.
* **Database backend integration:** SQLite for persistent expense storage.
* **RAG capabilities:** Answers are explicitly grounded in retrieved expense records from the database, not hallucinated.
