# freeweb.py — free, keyless web helpers

This is the source of `freeweb.py`. It is a single-file Python 3 stdlib
module that wraps the free APIs we use to research books without a Firecrawl key or
any other paid service.

Keep it dependency-free. The skill depends on this script being available on the machine where it runs.

## Why a helper, not ad-hoc curl

- The same six operations (DDG search, Wikipedia summary, Wikipedia extract, Google
  Books, Open Library, URL fetch) get repeated across book runs.
- Each has its own quirks (URL encoding, JSON vs HTML, 429 backoff, HTML stripping).
- A 5-minute tool saves 30 minutes per book.

## Source

```python
#!/usr/bin/env python3
"""Free web search helpers — no API keys required."""
from __future__ import annotations
import html, json, re, sys, urllib.parse, urllib.request
from typing import Any

UA = "Mozilla/5.0 (X11; Linux x86_64) Linus/1.0"
TIMEOUT = 25


def _http(url: str, accept: str = "text/html,*/*") -> bytes:
    req = urllib.request.Request(url, headers={"User-Agent": UA, "Accept": accept})
    with urllib.request.urlopen(req, timeout=TIMEOUT) as r:
        return r.read()


def _http_json(url: str) -> dict[str, Any]:
    raw = _http(url, accept="application/json,*/*")
    try:
        return json.loads(raw.decode("utf-8", errors="replace"))
    except json.JSONDecodeError:
        return {"_raw": raw[:2000].decode("utf-8", errors="replace")}


def ddg_search(query: str, limit: int = 8) -> list[dict[str, str]]:
    """DuckDuckGo HTML discovery. Often bot-walled from datacenter IPs."""
    url = "https://html.duckduckgo.com/html/?" + urllib.parse.urlencode(
        {"q": query, "kl": "us-en"}
    )
    text = _http(url).decode("utf-8", errors="replace")
    out: list[dict[str, str]] = []
    for m in re.finditer(
        r'<a[^>]+class="result__a"[^>]+href="([^"]+)"[^>]*>([^<]+)</a>'
        r'.*?<a[^>]+class="result__snippet"[^>]*>(.*?)</a>',
        text, re.S | re.I,
    ):
        href = html.unescape(m.group(1))
        if "uddg=" in href:
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(href).query)
            href = urllib.parse.unquote(qs.get("uddg", [href])[0])
        out.append({
            "title": html.unescape(re.sub(r"<[^>]+>", "", m.group(2))).strip(),
            "url": href,
            "snippet": html.unescape(re.sub(r"<[^>]+>", "", m.group(3))).strip(),
        })
        if len(out) >= limit:
            break
    return out


def wiki_search(query: str, limit: int = 5) -> list[dict[str, str]]:
    data = _http_json("https://en.wikipedia.org/w/api.php?" + urllib.parse.urlencode({
        "action": "query", "list": "search", "srsearch": query,
        "srlimit": str(limit), "format": "json", "utf8": "1",
    }))
    return [
        {"title": h["title"],
         "url": "https://en.wikipedia.org/wiki/" + urllib.parse.quote(h["title"].replace(" ", "_"))}
        for h in data.get("query", {}).get("search", [])
    ]


def wiki_summary(title: str) -> dict[str, Any]:
    return _http_json(
        "https://en.wikipedia.org/api/rest_v1/page/summary/" + urllib.parse.quote(title)
    )


def wiki_extract(title: str, char_limit: int = 12000) -> str:
    data = _http_json("https://en.wikipedia.org/w/api.php?" + urllib.parse.urlencode({
        "action": "query", "prop": "extracts", "explaintext": "1",
        "titles": title, "format": "json", "utf8": "1",
    }))
    for _, page in (data.get("query", {}).get("pages") or {}).items():
        return (page.get("extract") or "")[:char_limit]
    return ""


def google_books(query: str, limit: int = 5) -> list[dict[str, Any]]:
    data = _http_json("https://www.googleapis.com/books/v1/volumes?" + urllib.parse.urlencode({
        "q": query, "maxResults": str(limit),
    }))
    return [
        {
            "title": vi.get("title"),
            "authors": vi.get("authors", []),
            "publishedDate": vi.get("publishedDate"),
            "description": vi.get("description"),
            "categories": vi.get("categories", []),
            "pageCount": vi.get("pageCount"),
            "previewLink": vi.get("previewLink"),
        }
        for vi in (item.get("volumeInfo", {}) for item in data.get("items", []) or [])
    ]


def open_library(query: str, limit: int = 5) -> list[dict[str, Any]]:
    data = _http_json("https://openlibrary.org/search.json?" + urllib.parse.urlencode({
        "q": query, "limit": str(limit),
    }))
    return [
        {"title": d.get("title"), "authors": d.get("author_name", []),
         "first_publish_year": d.get("first_publish_year"),
         "subject": d.get("subject", [])[:10], "key": d.get("key")}
        for d in data.get("docs", [])
    ]


def fetch_url(url: str, char_limit: int = 15000) -> str:
    raw = _http(url)
    text = raw.decode("utf-8", errors="replace")
    text = re.sub(r"<(script|style|noscript|nav|header|footer|aside|form)[^>]*>.*?</\1>",
                  "", text, flags=re.S | re.I)
    text = re.sub(r"<[^>]+>", " ", text)
    text = html.unescape(text)
    text = re.sub(r"[ \t]+", " ", text)
    text = re.sub(r"\n{2,}", "\n\n", text)
    return text.strip()[:char_limit]


if __name__ == "__main__":
    cmd = sys.argv[1] if len(sys.argv) > 1 else ""
    rest = sys.argv[2:]
    if cmd == "ddg":
        print(json.dumps(ddg_search(" ".join(rest)), indent=2, ensure_ascii=False))
    elif cmd == "wiki":
        mode, q = (rest[0], " ".join(rest[1:])) if rest else ("search", "Taleb")
        if mode == "search":
            print(json.dumps(wiki_search(q), indent=2, ensure_ascii=False))
        elif mode == "summary":
            print(json.dumps(wiki_summary(q), indent=2, ensure_ascii=False))
        elif mode == "extract":
            print(wiki_extract(q))
    elif cmd == "book":
        print(json.dumps(google_books(" ".join(rest)), indent=2, ensure_ascii=False))
    elif cmd == "ol":
        print(json.dumps(open_library(" ".join(rest)), indent=2, ensure_ascii=False))
    elif cmd == "fetch":
        print(fetch_url(rest[0]) if rest else "")
    else:
        print("usage: freeweb.py ddg|wiki|book|ol|fetch <args>", file=sys.stderr)
        sys.exit(2)
```

## Operational notes

- **DDG**: Often bot-walled. If `ddg_search` returns `[]`, check whether the
  response was the captcha challenge page (size ~14k, no `result__a` class).
  Fall back to Wikipedia + known-good summary URLs.
- **Google Books**: Hits HTTP 429 with low effort. Back off 30s and retry; if
  still 429, switch to Open Library.
- **Wikipedia**: Reliable. The REST `summary` endpoint is JSON; the `extracts`
  endpoint with `explaintext=1` is the cleanest way to get a plain-text article.
- **fetch_url**: Crude HTML stripper. Good enough for blog posts, not for
  JavaScript-rendered pages. If a page is empty after stripping, it's probably
  client-side rendered — fall back to the browser tool if available.

## CLI quick reference

```bash
freeweb.py ddg "Antifragile Nassim Taleb"
freeweb.py wiki search "Antifragile"
freeweb.py wiki summary "Antifragile_(book)"
freeweb.py wiki extract "Antifragile_(book)"
freeweb.py book "Antifragile Nassim Taleb"
freeweb.py ol "Antifragile Nassim Taleb"
freeweb.py fetch "https://example.com/some-summary-page"
```
