"""
fetch_egypt_news.py
Fetches RSS feeds from Egyptian news sources, translates non-English content
to English, categorizes stories, and writes up to 20 stories per category
into docs/Egypt_news.json.

Output published via GitHub Pages at:
https://stratagemdrive.github.io/egypt-local/Egypt_news.json
"""

import json
import os
import re
import time
from datetime import datetime, timezone, timedelta
from dateutil import parser as dateparser

import feedparser
from deep_translator import GoogleTranslator

# ---------------------------------------------------------------------------
# Configuration
# ---------------------------------------------------------------------------

FEEDS = [
    # English-language sources (no translation needed)
    {"name": "Egypt Independent",   "url": "https://egyptindependent.com/feed",         "lang": "en"},
    {"name": "Daily News Egypt",    "url": "https://dailynewsegypt.com/feed",            "lang": "en"},
    {"name": "Egyptian Streets",    "url": "https://egyptianstreets.com/feed",           "lang": "en"},
    {"name": "Egypt Today",         "url": "https://www.egypttoday.com/rss",             "lang": "en"},
    {"name": "Mada Masr",           "url": "https://madamasr.com/en/feed",               "lang": "en"},
    {"name": "KingFut",             "url": "https://feeds.feedburner.com/KingFut",       "lang": "en"},
    {"name": "NileSports",          "url": "https://nilesports.com/feed",                "lang": "en"},
    # Arabic-language sources (require translation)
    {"name": "Sada El Balad",       "url": "https://elbalad.news/rss.aspx",              "lang": "ar"},
    {"name": "Elfagr",              "url": "https://www.elfagr.org/rss.aspx",            "lang": "ar"},
    {"name": "Al-Masry Al-Youm",    "url": "https://www.almasryalyoum.com/rss/rssfeed", "lang": "ar"},
    {"name": "Masress",             "url": "https://masress.com/en/rss",                 "lang": "en"},
    {"name": "Egypt Oil & Gas",     "url": "https://egyptoil-gas.com/news/feed",         "lang": "en"},
]

CATEGORIES = ["Diplomacy", "Military", "Energy", "Economy", "Local Events"]
MAX_PER_CATEGORY = 20
MAX_AGE_DAYS = 7
OUTPUT_PATH = os.path.join("docs", "Egypt_news.json")

# ---------------------------------------------------------------------------
# Category keyword mapping
# ---------------------------------------------------------------------------

CATEGORY_KEYWORDS = {
    "Diplomacy": [
        "diplomat", "ambassador", "foreign minister", "ministry of foreign",
        "bilateral", "treaty", "summit", "united nations", "un ", "nato",
        "arab league", "african union", "embassy", "consulate", "sanction",
        "agreement", "accord", "ceasefire", "peace talks", "negotiat",
        "foreign policy", "secretary of state", "eu ", "european union",
    ],
    "Military": [
        "military", "army", "armed forces", "air force", "navy", "war",
        "troops", "soldier", "weapon", "missile", "drone", "airstrike",
        "terrorist", "insurgent", "defense minister", "ministry of defense",
        "border security", "operation", "combat", "attack", "conflict",
        "sinai", "hamas", "hezbollah", "isis", "isil", "daesh",
    ],
    "Energy": [
        "energy", "oil", "gas", "petroleum", "electricity", "power plant",
        "renewabl", "solar", "wind farm", "nuclear", "lng", "pipeline",
        "fuel", "refinery", "suez", "hydrocarbon", "megawatt", "gigawatt",
        "ministry of petroleum", "ministry of energy", "eni ", "bp ",
        "exxon", "natural gas", "carbon", "emission",
    ],
    "Economy": [
        "economy", "economic", "gdp", "inflation", "interest rate",
        "central bank", "imf", "world bank", "budget", "deficit", "debt",
        "trade", "export", "import", "investment", "stock exchange",
        "currency", "pound", "egp", "revenue", "fiscal", "tax",
        "ministry of finance", "privatization", "bond", "loan", "credit",
        "tourism", "remittance", "foreign reserve", "unemployment",
    ],
    "Local Events": [
        "cairo", "alexandria", "giza", "luxor", "aswan", "suez", "sinai",
        "nile", "egypt", "egyptian", "festival", "protest", "election",
        "parliament", "president sisi", "prime minister", "ministry",
        "court", "law", "police", "crime", "flood", "earthquake",
        "infrastructure", "metro", "highway", "hospital", "school",
        "university", "culture", "sport", "football", "africa cup",
    ],
}


def classify(title: str, summary: str) -> str:
    """Return the best-fit category for a story or 'Local Events' as fallback."""
    text = (title + " " + summary).lower()
    scores = {cat: 0 for cat in CATEGORIES}
    for cat, keywords in CATEGORY_KEYWORDS.items():
        for kw in keywords:
            if kw in text:
                scores[cat] += 1
    best = max(scores, key=scores.get)
    # Require at least one keyword match; otherwise fall back to Local Events
    return best if scores[best] > 0 else "Local Events"


# ---------------------------------------------------------------------------
# Translation helper
# ---------------------------------------------------------------------------

_translator_cache: dict[str, GoogleTranslator] = {}


def get_translator(src_lang: str) -> GoogleTranslator:
    if src_lang not in _translator_cache:
        _translator_cache[src_lang] = GoogleTranslator(source=src_lang, target="en")
    return _translator_cache[src_lang]


def safe_translate(text: str, src_lang: str) -> str:
    """Translate text to English, returning original on failure."""
    if not text or src_lang == "en":
        return text
    try:
        translator = get_translator(src_lang)
        # GoogleTranslator has a ~5000-char limit per call
        if len(text) > 4500:
            text = text[:4500]
        return translator.translate(text) or text
    except Exception:
        return text


# ---------------------------------------------------------------------------
# Date helpers
# ---------------------------------------------------------------------------

def parse_date(entry) -> datetime | None:
    """Parse a feedparser entry's published date into a UTC-aware datetime."""
    for attr in ("published", "updated", "created"):
        raw = getattr(entry, attr, None)
        if raw:
            try:
                dt = dateparser.parse(raw)
                if dt and dt.tzinfo is None:
                    dt = dt.replace(tzinfo=timezone.utc)
                return dt.astimezone(timezone.utc)
            except Exception:
                continue
    return None


def is_recent(dt: datetime | None) -> bool:
    if dt is None:
        return False
    cutoff = datetime.now(timezone.utc) - timedelta(days=MAX_AGE_DAYS)
    return dt >= cutoff


# ---------------------------------------------------------------------------
# Feed fetching
# ---------------------------------------------------------------------------

def fetch_all_stories() -> list[dict]:
    """Pull stories from all configured feeds, translate if needed."""
    stories = []
    for feed_cfg in FEEDS:
        print(f"  Fetching: {feed_cfg['name']} …", flush=True)
        try:
            parsed = feedparser.parse(feed_cfg["url"])
        except Exception as exc:
            print(f"    ERROR parsing feed: {exc}")
            continue

        for entry in parsed.entries:
            pub_dt = parse_date(entry)
            if not is_recent(pub_dt):
                continue

            raw_title = getattr(entry, "title", "") or ""
            raw_summary = getattr(entry, "summary", "") or ""
            # Strip HTML tags from summary
            raw_summary = re.sub(r"<[^>]+>", " ", raw_summary).strip()

            lang = feed_cfg["lang"]
            title = safe_translate(raw_title, lang)
            summary = safe_translate(raw_summary[:500], lang)  # translate excerpt only

            url = getattr(entry, "link", "") or ""
            category = classify(title, summary)

            stories.append({
                "title": title,
                "source": feed_cfg["name"],
                "url": url,
                "published_date": pub_dt.isoformat() if pub_dt else "",
                "category": category,
            })

        # Be polite to translation API
        if lang != "en":
            time.sleep(1)

    return stories


# ---------------------------------------------------------------------------
# Merge logic
# ---------------------------------------------------------------------------

def load_existing() -> list[dict]:
    if os.path.exists(OUTPUT_PATH):
        try:
            with open(OUTPUT_PATH, "r", encoding="utf-8") as f:
                return json.load(f)
        except Exception:
            return []
    return []


def merge_stories(existing: list[dict], fresh: list[dict]) -> list[dict]:
    """
    For each category:
      - Start with existing stories that are still within 7 days
      - Add new stories (deduped by URL)
      - Trim to MAX_PER_CATEGORY, replacing oldest first
    """
    now = datetime.now(timezone.utc)
    cutoff = now - timedelta(days=MAX_AGE_DAYS)

    def as_dt(story: dict) -> datetime:
        try:
            return dateparser.parse(story["published_date"]).astimezone(timezone.utc)
        except Exception:
            return datetime.min.replace(tzinfo=timezone.utc)

    result = []
    for cat in CATEGORIES:
        # Filter existing to valid + recent
        pool = [
            s for s in existing
            if s.get("category") == cat and as_dt(s) >= cutoff
        ]
        existing_urls = {s["url"] for s in pool}

        # Add fresh stories not already present
        new_for_cat = [
            s for s in fresh
            if s.get("category") == cat and s["url"] not in existing_urls
        ]
        pool.extend(new_for_cat)

        # Sort newest-first, keep top MAX_PER_CATEGORY
        pool.sort(key=as_dt, reverse=True)
        result.extend(pool[:MAX_PER_CATEGORY])

    return result


# ---------------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------------

def main():
    os.makedirs("docs", exist_ok=True)

    print("Fetching fresh stories …")
    fresh = fetch_all_stories()
    print(f"  → {len(fresh)} recent stories fetched across all feeds.")

    print("Loading existing JSON …")
    existing = load_existing()

    print("Merging …")
    merged = merge_stories(existing, fresh)

    # Summary
    for cat in CATEGORIES:
        count = sum(1 for s in merged if s["category"] == cat)
        print(f"  {cat}: {count} stories")

    with open(OUTPUT_PATH, "w", encoding="utf-8") as f:
        json.dump(merged, f, ensure_ascii=False, indent=2)

    print(f"Done. Wrote {len(merged)} stories to {OUTPUT_PATH}.")


if __name__ == "__main__":
    main()
