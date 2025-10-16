
---

# 8 — Intent Engine backed by SQLite (intent_engine.py)

**File:** `src/intent_engine.py`
```python
#!/usr/bin/env python3
"""
SQL-backed Intent Engine - intent_engine.py

- A simple intent store in SQLite and a SQL-first matcher using trigram-like fuzzy match
- Demo uses SQLite (standard library)
"""

import sqlite3
import os
from typing import List

DB = "intents.db"

def init_db():
    if os.path.exists(DB):
        os.remove(DB)
    conn = sqlite3.connect(DB)
    c = conn.cursor()
    c.execute("CREATE TABLE intents(id INTEGER PRIMARY KEY, intent TEXT, templates TEXT)")
    c.execute("CREATE VIRTUAL TABLE IF NOT EXISTS trigram USING fts5(template)")
    conn.commit()
    return conn

def add_intent(conn, intent, templates: List[str]):
    c = conn.cursor()
    c.execute("INSERT INTO intents(intent, templates) VALUES (?, ?)", (intent, "\n".join(templates)))
    # for demo we just store templates; a real system would index ngrams
    conn.commit()

def match(conn, text):
    # naive SQL-first search: find intents where template substring appears
    c = conn.cursor()
    c.execute("SELECT id, intent, templates FROM intents")
    best = []
    for row in c.fetchall():
        tid, intent, templates = row
        score = 0
        for t in templates.splitlines():
            if t.lower() in text.lower():
                score += 1
            elif any(word in text.lower() for word in t.lower().split()):
                score += 0.1
        if score > 0:
            best.append((intent, score))
    best.sort(key=lambda x: -x[1])
    return best

def demo():
    conn = init_db()
    add_intent(conn, "greet", ["hello", "hi", "hey there"])
    add_intent(conn, "set_timer", ["set a timer", "timer for", "start timer"])
    add_intent(conn, "play_music", ["play music", "play song", "start playlist"])
    print(match(conn, "Hey, can you set a timer for 10 minutes?"))
    print(match(conn, "hello!"))

if __name__ == "__main__":
    demo()
