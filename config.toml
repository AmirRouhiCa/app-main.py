# path: app/main.py
from __future__ import annotations

import argparse
import html
import json
import logging
import os
import sys
import time
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Dict, List, Optional

import feedparser
import requests

# TOML loader
try:
    import tomllib  # Python 3.11+
except Exception:
    import tomli as tomllib  # Python <3.11

@dataclass(frozen=True)
class FeedConfig:
    url: str
    enabled: bool = True
    per_run_limit: int = 5
    timeout: float = 15.0

@dataclass(frozen=True)
class AppConfig:
    feeds: List[FeedConfig]
    poll_interval: int = 300
    message_template: str = (
        "<b>{title}</b>\n"
        "منبع: {source}\n"
        "⏰ {published}\n"
        "{link}"
    )
    disable_web_page_preview: bool = False
    dry_run: bool = False
    per_loop_global_limit: int = 50
    user_agent: str = "RSS2Telegram/1.0 (+https://example.invalid)"
    seen_json_path: str = "data/seen.json"
    telegram_api_base: str = "https://api.telegram.org"
    send_delay_ms: int = 500

def load_config_from_toml(path: str) -> AppConfig:
    with open(path, "rb") as f:
        raw = tomllib.load(f)
    feeds = [
        FeedConfig(
            url=item["url"],
            enabled=item.get("enabled", True),
            per_run_limit=int(item.get("per_run_limit", 5)),
            timeout=float(item.get("timeout", 15.0)),
        )
        for item in raw.get("feeds", [])
    ]
    def _d(name: str): return AppConfig.__dataclass_fields__[name].default  # why: defaults centralized
    return AppConfig(
        feeds=feeds,
        poll_interval=int(raw.get("poll_interval", 300)),
        message_template=str(raw.get("message_template", _d("message_template"))),
        disable_web_page_preview=bool(raw.get("disable_web_page_preview", False)),
        dry_run=bool(raw.get("dry_run", False)),
        per_loop_global_limit=int(raw.get("per_loop_global_limit", 50)),
        user_agent=str(raw.get("user_agent", _d("user_agent"))),
        seen_json_path=str(raw.get("seen_json_path", _d("seen_json_path"))),
        telegram_api_base=str(raw.get("telegram_api_base", _d("telegram_api_base"))),
        send_delay_ms=int(raw.get("send_delay_ms", 500)),
    )

class SeenStoreJSON:
    # چرا: در Actions ماشین موقتی است؛ با commit کردن JSON در repo وضعیت حفظ می‌شود.
    def __init__(self, path: str) -> None:
        self.path = path
        self.data: Dict[str, Dict[str, int]] = {}
        self._load()
    def _load(self) -> None:
        try:
            os.makedirs(os.path.dirname(self.path), exist_ok=True)
            if os.path.exists(self.path):
                import json
                with open(self.path, "r", encoding="utf-8") as f:
                    raw = json.load(f)
                    if isinstance(raw, dict):
                        self.data = {
                            str(feed): {str(k): int(v) for k, v in items.items()}
                            for feed, items in raw.items() if isinstance(items, dict)
                        }
        except Exception:
            self.data = {}
    def has_seen(self, feed_url: str, entry_id: str) -> bool:
        return entry_id in self.data.get(feed_url, {})
    def mark_seen(self, feed_url: str, entry_id: str) -> None:
        self.data.setdefault(feed_url, {})[entry_id] = int(time.time())
    def save(self) -> None:
        tmp = self.path + ".tmp"
        with open(tmp, "w", encoding="utf-8") as f:
            json.dump(self.data, f, ensure_ascii=False, indent=2, sort_keys=True)
        os.replace(tmp, self.path)

class TelegramClient:
    def __init__(self, bot_token: str, base_url: str = "https://api.telegram.org") -> None:
        self.bot_token = bot_token
        self.base_url = base_url.rstrip("/")
        self.session = requests.Session()
    def send_message(self, chat_id: str | int, text: str,
                     disable_web_page_preview: bool = False, parse_mode: str = "HTML") -> dict:
        url = f"{self.base_url}/bot{self.bot_token}/sendMessage"
        payload = {"chat_id": chat_id, "text": text, "parse_mode": parse_mode,
                   "disable_web_page_preview": disable_web_page_preview}
        attempt = 0
        while True:
            attempt += 1
            try:
                resp = self.session.post(url, data=payload, timeout=25)
                if resp.status_code == 429:
                    retry_after = max(1, int(resp.json().get("parameters", {}).get("retry_after", 1)))
                    time.sleep(retry_after + 1)  # why: rate limit
                    continue
                resp.raise_for_status()
                data = resp.json()
                if not data.get("ok", False):
                    raise RuntimeError(f"Telegram API error: {data}")
                return data
            except (requests.RequestException, ValueError) as e:
                if attempt <= 3:
                    time.sleep(min(2 ** attempt, 10))
                    continue
                raise RuntimeError(f"Failed to send message: {e}") from e

@dataclass(frozen=True)
class FeedResult:
    source_title: str
    entries: List[dict]

class RSSFetcher:
    def __init__(self, user_agent: str) -> None:
        self.user_agent = user_agent
    def fetch(self, feed: FeedConfig) -> FeedResult:
        d = feedparser.parse(feed.url, request_headers={"User-Agent": self.user_agent})
        return FeedResult(source_title=d.feed.get("title", feed.url), entries=list(d.entries or []))

def build_entry_id(entry: dict) -> str:
    cand = entry.get("id") or entry.get("guid") or entry.get("link") \
        or f"{entry.get('title','')}_{entry.get('published','')}" \
        or f"{entry.get('title','')}_{entry.get('updated','')}"
    return str(cand).strip()

def pick_published(entry: dict) -> Optional[float]:
    for key in ("published_parsed", "updated_parsed"):
        val = entry.get(key)
        if val:
            try:
                return time.mktime(val)
            except Exception:
                pass
    return None

def clean_text(s: str, max_len: int = 500) -> str:
    s = " ".join(html.unescape(str(s or "")).split())
    return s[:max_len].rstrip("…") + ("…" if len(s) > max_len else "")

def format_message(template: str, title: str, source: str, link: str, published_ts: Optional[float]) -> str:
    safe_title = html.escape(clean_text(title, 400))
    safe_source = html.escape(clean_text(source, 120))
    safe_link = html.escape(link or "")
    ts = published_ts or time.time()
    when = datetime.fromtimestamp(ts, tz=timezone.utc).astimezone().strftime("%Y-%m-%d %H:%M")
    return template.format(title=safe_title, source=safe_source, link=safe_link, published=when)

class App:
    def __init__(self, cfg: AppConfig, tg: TelegramClient, store: SeenStoreJSON, chat_id: str | int) -> None:
        self.cfg, self.tg, self.store, self.chat_id = cfg, tg, store, chat_id
        self.fetcher = RSSFetcher(cfg.user_agent)
    def run_once(self) -> int:
        total_sent = 0
        for feed in self.cfg.feeds:
            if not feed.enabled: continue
            try:
                fr = self.fetcher.fetch(feed)
            except Exception as e:
                logging.warning("Failed to fetch %s: %s", feed.url, e); continue
            new_count = 0
            for entry in fr.entries:
                if total_sent >= self.cfg.per_loop_global_limit:
                    logging.info("Per-loop cap reached"); break
                entry_id = build_entry_id(entry)
                if not entry_id or self.store.has_seen(feed.url, entry_id): continue
                title = entry.get("title") or "(بدون عنوان)"
                link = entry.get("link") or ""
                published_ts = pick_published(entry) or time.time()
                msg = format_message(self.cfg.message_template, title, fr.source_title, link, published_ts)
                logging.info("New: %s | %s", fr.source_title, title)
                if not self.cfg.dry_run:
                    self.tg.send_message(self.chat_id, msg, self.cfg.disable_web_page_preview)
                    time.sleep(self.cfg.send_delay_ms / 1000.0)
                self.store.mark_seen(feed.url, entry_id)
                total_sent += 1; new_count += 1
                if new_count >= feed.per_run_limit: break
        self.store.save()
        return total_sent

def parse_args(argv: List[str]) -> argparse.Namespace:
    p = argparse.ArgumentParser(description="Fetch RSS feeds and post to Telegram (GitHub Actions-ready).")
    p.add_argument("--config","-c", default="config.toml")
    p.add_argument("--chat-id", default=os.getenv("TELEGRAM_CHAT_ID",""))
    p.add_argument("--log-level", default=os.getenv("LOG_LEVEL","INFO"))
    return p.parse_args(argv)

def main(argv: List[str]) -> int:
    args = parse_args(argv)
    logging.basicConfig(level=getattr(logging, args.log_level.upper(), logging.INFO),
                        format="%(asctime)s %(levelname)s %(message)s")
    token = os.getenv("TELEGRAM_BOT_TOKEN")
    if not token:
        logging.error("TELEGRAM_BOT_TOKEN is required"); return 2
    if not args.chat_id:
        logging.error("Provide --chat-id or set TELEGRAM_CHAT_ID"); return 2
    cfg = load_config_from_toml(args.config)
    store = SeenStoreJSON(cfg.seen_json_path)
    tg = TelegramClient(bot_token=token, base_url=cfg.telegram_api_base)
    app = App(cfg, tg, store, args.chat_id)
    sent = app.run_once()
    logging.info("Done. Sent=%d", sent)
    return 0

if __name__ == "__main__":
    raise SystemExit(main(sys.argv[1:]))

# --------------------------------------------
# path: config.toml
poll_interval = 300
per_loop_global_limit = 50
disable_web_page_preview = false
dry_run = false
user_agent = "RSS2Telegram/1.0 (+https://yourdomain.example)"
seen_json_path = "data/seen.json"
send_delay_ms = 350

message_template = """
<b>{title}</b>
منبع: {source}
⏰ {published}
{link}
"""

[[feeds]]
url = "https://www.mehrnews.com/rss"
enabled = true
per_run_limit = 5

[[feeds]]
url = "https://www.isna.ir/rss"
enabled = true
per_run_limit = 5

# --------------------------------------------
# path: requirements.txt
feedparser==6.0.11
requests>=2.31.0
tomli>=2.0.1; python_version < "3.11"

# --------------------------------------------
# path: .github/workflows/rss2tg.yml
name: RSS to Telegram

on:
  workflow_dispatch: {}
  schedule:
    - cron: "0 6,14,21 * * *"  # سه بار در روز (UTC)

permissions:
  contents: write  # برای commit کردن data/seen.json

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install deps
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run script
        env:
          TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}
          TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
          LOG_LEVEL: INFO
        run: |
          python app/main.py --config config.toml

      - name: Commit seen state if changed
        run: |
          if [ -n "$(git status --porcelain data/seen.json 2>/dev/null)" ]; then
            git config user.name "github-actions[bot]"
            git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
            git add data/seen.json
            git commit -m "chore: update seen state [skip ci]"
            git push
          else
            echo "No state change."
          fi
