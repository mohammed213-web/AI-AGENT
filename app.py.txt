# app.py
import os
import uuid
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional
import sqlite3
import time

# === Database setup ===
DB = "chat.db"

def init_db():
    conn = sqlite3.connect(DB)
    c = conn.cursor()
    c.execute("""CREATE TABLE IF NOT EXISTS sessions (
                    session_id TEXT PRIMARY KEY,
                    created_at INTEGER,
                    email TEXT,
                    profile TEXT,
                    questions_count INTEGER DEFAULT 0
                )""")
    c.execute("""CREATE TABLE IF NOT EXISTS messages (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    session_id TEXT,
                    role TEXT,
                    text TEXT,
                    ts INTEGER
                )""")
    conn.commit()
    conn.close()

init_db()

# === FastAPI app ===
app = FastAPI()
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # allows Wix frontend to call API
    allow_methods=["*"],
    allow_headers=["*"],
    allow_credentials=True
)

# === Models ===
class ChatRequest(BaseModel):
    session_id: Optional[str] = None
    message: str
    mode: Optional[str] = "chat"

class ChatResponse(BaseModel):
    session_id: str
    reply: str
    questions_left: int

# === DB helpers ===
def get_session(sess_id):
    conn = sqlite3.connect(DB)
    c = conn.cursor()
    c.execute("SELECT session_id, questions_count FROM sessions WHERE session_id=?", (sess_id,))
    row = c.fetchone()
    conn.close()
    return row

def create_session():
    sid = str(uuid.uuid4())
    conn = sqlite3.connect(DB)
    c = conn.cursor()
    c.execute("INSERT INTO sessions (session_id, created_at, questions_count) VALUES (?, ?, ?)",
              (sid, int(time.time()), 0))
    conn.commit()
    conn.close()
    return sid

def inc_question_count(sess_id):
    conn = sqlite3.connect(DB)
    c = conn.cursor()
    c.execute("UPDATE sessions SET questions_count = questions_count + 1 WHERE session_id = ?", (sess_id,))
    conn.commit()
    c.execute("SELECT questions_count FROM sessions WHERE session_id = ?", (sess_id,))
    q = c.fetchone()[0]
    conn.close()
    return q

def save_message(sess_id, role, text):
    conn = sqlite3.connect(DB)
    c = conn.cursor()
    c.execute("INSERT INTO messages (session_id, role, text, ts) VALUES (?, ?, ?, ?)",
              (sess_id, role, text, int(time.time())))
    conn.commit()
    conn.close()

# === Placeholder AI call ===
def LLM_CALL(prompt, history=None):
    # TODO: Replace this with Groq/OpenAI API call
    return f"AI: {prompt}"

# === API endpoints ===
@app.post("/chat", response_model=ChatResponse)
async def chat(req: ChatRequest):
    sid = req.session_id
    if not sid or not get_session(sid):
        sid = create_session()

    if req.mode == "chat":
        qcount = inc_question_count(sid)
        if qcount > 3:
            save_message(sid, "user", req.message)
            upgrade_text = "You have reached your free limit of 3 questions. Kindly upgrade to continue this chat: <YOUR_PAYMENT_URL>"
            save_message(sid, "assistant", upgrade_text)
            return ChatResponse(session_id=sid, reply=upgrade_text, questions_left=0)

    save_message(sid, "user", req.message)
    reply = LLM_CALL(req.message)
    save_message(sid, "assistant", reply)

    conn = sqlite3.connect(DB)
    c = conn.cursor()
    c.execute("SELECT questions_count FROM sessions WHERE session_id = ?", (sid,))
    qleft = max(0, 3 - c.fetchone()[0])
    conn.close()

    return ChatResponse(session_id=sid, reply=reply, questions_left=qleft)

@app.post("/generate_report")
async def generate_report(req: Request):
    data = await req.json()
    session_id = data.get("session_id")
    email = data.get("email")
    profile = data.get("profile")
    # TODO: implement scraping + AI report generation
    return {"status":"queued", "message":"Report generation started. You will receive an email when ready."}