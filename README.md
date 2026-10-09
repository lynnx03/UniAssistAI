# UniAssist — Student Academic Assistant (Text-to-SQL Chatbot)

An academic assistant for KMITL students and academic advisors, built with **Flask + LLM**.
Users ask questions in Thai, the LLM writes SQL that runs against the curriculum database (read-only),
and the results are turned into a natural-language answer. It also includes a general chat mode, a GPA calculator, a scholarship list, and an advisor dashboard.

## Key Features

- **Curriculum Q&A (Text-to-SQL)** — the LLM decides whether a question needs data retrieval (SQL) or is casual conversation (CHAT); the UI shows the generated SQL, the raw result table, and the latency of each step
- **Chat mode** — answers like a chatbot, grounded in facts from the DB (reduces hallucination), and used as a fallback when SQL fails
- **GPA calculator** — paste a transcript or edit courses in the table to calculate term/cumulative GPA and academic status (retirement risk / probation / honors), using grade values and criteria from the DB rather than hardcoded values
- **Scholarship list**
- **Advisor dashboard** — view advisees, their term GPAs, and a risk summary
- **Google OAuth** — sign in with a Google account restricted by domain, with roles (student / advisor / dev)

## Demo

**Chat page — curriculum Q&A**

![UniAssist chat page](img/image.png)

**GPA calculation from a transcript**

![GPA calculator page](img/image1.png)

**Scholarships**

![Scholarships page](img/image2.png)

## Security

Because students type the questions themselves, SQL execution is restricted at several layers:

- The DB connection is truly **read-only** (`file:...?mode=ro`) — the LLM cannot run `DELETE`/`UPDATE`
- SQL is validated to be a single `SELECT`/`WITH` statement; write statements and multiple statements are blocked
- A `LIMIT` is added automatically if missing — prevents dumping entire tables
- The Google Client Secret is only read server-side; the user is stored in a signed session cookie

## Project Structure

| File | Purpose |
|---|---|
| `app.py` | Flask web app — routes `/`, `/ask`, `/calc-gpa`, `/scholarships`, `/dashboard-data`, `/dev-info` |
| `text_to_sql_chatbot.py` | Text-to-SQL core: builds prompts, calls the LLM, validates/runs SQL, composes answers (also runnable as a CLI) |
| `assist_logic.py` | Shared logic: transcript parsing, GPA calculation, status classification based on DB criteria |
| `google_auth.py` | Google OAuth 2.0 + role decorators (`login_required`, `advisor_required`, `dev_required`) |
| `build_qwen_db.py` | Builds a DB from the JSON that Qwen extracted from the curriculum tables |
| `build_chatbot_db.py` | Assembles the chatbot DB (study plans from Ground Truth + the rest from Qwen) |
| `build_extra_tables.py` | Adds extra tables: `grade_scale`, `rules`, `scholarships`, `students`, `student_term_gpa` (idempotent) |
| `templates/` | `login.html`, `index.html` |
| `data/` | Source data (ground truth, extraction results, curriculum tables) |
| `chatbot_teach_table.db` | Main database used by the app |

## Installation

Requires Python 3.10+

```bash
pip install -r requirements.txt
```

## Configure `.env`

Create a `.env` file in the same folder as `app.py`:

```ini
# LLM (OpenAI-compatible — works with Qwen / DeepSeek / Typhoon / Gemini via OpenRouter, etc.)
LLM_BASE_URL=https://openrouter.ai/api/v1
LLM_API_KEY=sk-...
LLM_MODEL=qwen/qwen3-235b-a22b
LLM_PROVIDER=            # optional (auto)

# Database
DB_PATH=chatbot_teach_table.db

# Google OAuth (see GOOGLE_AUTH_SETUP.md for detailed steps)
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI=http://localhost:5000/auth/callback
FLASK_SECRET_KEY=<long random string>

# Access control (comma-separated)
ALLOWED_DOMAINS=kmitl.ac.th
DEV_EMAILS=you@kmitl.ac.th
ADVISOR_EMAILS=advisor@kmitl.ac.th
STUDENT_EMAILS=
```

> For detailed Google OAuth setup (creating credentials, redirect URI), see [GOOGLE_AUTH_SETUP.md](GOOGLE_AUTH_SETUP.md)

## Running

```bash
python app.py
```

Open http://localhost:5000 in your browser (you must sign in with Google before using the app).

### Run the chatbot core as a CLI (for testing/evaluation)

```bash
python text_to_sql_chatbot.py                                  # interactive mode
python text_to_sql_chatbot.py -q "หลักสูตร IT จบกี่หน่วยกิต"      # "How many credits to graduate from the IT program?"
python text_to_sql_chatbot.py --sql "SELECT ..."               # run SQL directly
```

## Rebuilding the Database (if you want to build it yourself)

```bash
python build_qwen_db.py ./data/extracted_teach_table ./qwen_teach_table.db
python build_chatbot_db.py       # assembles chatbot_teach_table.db
python build_extra_tables.py     # adds grade_scale / rules / scholarships / students
```

## Notes

- The `students` / `student_term_gpa` tables currently contain mock data, prepared to be connected to the registrar API later (see the real data-fetching flow in [README-2.md](README-2.md))
- Some LLM providers are inconsistent (occasionally returning `NO_ANSWER` depending on the time of day) — the system already handles this with retry-on-NO_ANSWER
