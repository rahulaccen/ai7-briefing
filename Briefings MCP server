"""Briefings MCP server — exposes the AI7 briefings feed as MCP tools via FastMCP."""

from __future__ import annotations

import datetime
from collections import Counter
from typing import Any

import requests
from fastmcp import FastMCP

FEED_URL = "https://raw.githubusercontent.com/rahulaccen/ai7-briefing/main/briefings.json"
FETCH_TIMEOUT = 10
SEARCH_RESULT_LIMIT = 25
DATE_MIN = "0000-00-00"
DATE_MAX = "9999-99-99"

mcp = FastMCP("briefings-mcp")


def _fetch() -> list[dict[str, Any]]:
    response = requests.get(FEED_URL, timeout=FETCH_TIMEOUT)
    response.raise_for_status()
    briefings = response.json()
    for briefing in briefings:
        briefing["_date"] = briefing["date"][:10]
    return sorted(briefings, key=lambda briefing: briefing["_date"])


def _today() -> str:
    return datetime.date.today().isoformat()


def _days_ago(days: int) -> str:
    return (datetime.date.today() - datetime.timedelta(days=days)).isoformat()


def _in_range(briefings: list[dict[str, Any]], date_from: str, date_to: str) -> list[dict[str, Any]]:
    return [briefing for briefing in briefings if date_from <= briefing["_date"] <= date_to]


def _fmt_item(item: dict[str, Any], date_label: str | None = None) -> str:
    prefix = f"[{date_label}] " if date_label else ""
    lines = [f"{prefix}[{item['company']}] {item['headline']}", f"  {item['summary']}"]
    if item.get("sourceUrl"):
        lines.append(f"  {item['sourceUrl']}")
    return "\n".join(lines)


def _fmt_briefing(briefing: dict[str, Any]) -> str:
    lines = [f"── {briefing['_date']} ──"]
    for item in briefing["items"]:
        lines.append(_fmt_item(item))
        lines.append("")
    return "\n".join(lines)


@mcp.tool
def get_latest() -> str:
    """Return the most recent briefing with all company headlines and summaries."""
    briefings = _fetch()
    return _fmt_briefing(briefings[-1]) if briefings else "Feed is empty."


@mcp.tool
def get_by_date(date: str) -> str:
    """Return the briefing for a specific date (YYYY-MM-DD)."""
    briefings = _fetch()
    match = next((briefing for briefing in briefings if briefing["_date"] == date.strip()), None)
    if not match:
        return f"No briefing for {date}. Recent dates: {[briefing['_date'] for briefing in briefings[-5:]]}"
    return _fmt_briefing(match)


@mcp.tool
def get_range(date_from: str, date_to: str) -> str:
    """Return all briefings between two dates inclusive (YYYY-MM-DD)."""
    subset = _in_range(_fetch(), date_from.strip(), date_to.strip())
    if not subset:
        return "No briefings in that range."
    return "\n\n".join(_fmt_briefing(briefing) for briefing in subset)


@mcp.tool
def get_last_n_days(days: int = 7) -> str:
    """Return briefings from the last N calendar days (e.g. 7 for last week)."""
    date_from, date_to = _days_ago(days), _today()
    subset = _in_range(_fetch(), date_from, date_to)
    if not subset:
        return f"No briefings in the last {days} days ({date_from} → {date_to})."
    header = f"Last {days} days  ({date_from} → {date_to})  —  {len(subset)} briefings found\n\n"
    return header + "\n\n".join(_fmt_briefing(briefing) for briefing in subset)


@mcp.tool
def get_by_company(company: str, date_from: str = DATE_MIN, date_to: str = DATE_MAX) -> str:
    """Return all briefing items for one company, newest first. Optional date_from / date_to filters (YYYY-MM-DD).

    Known companies: NVIDIA, Anthropic, OpenAI, Snowflake, Palantir, Mistral, Databricks.
    """
    wanted = company.strip().lower()
    hits = [
        (briefing["_date"], item)
        for briefing in _in_range(_fetch(), date_from, date_to)
        for item in briefing["items"]
        if item.get("company", "").lower() == wanted
    ]
    if not hits:
        return f"No items for '{company}'."
    lines = [f"{len(hits)} items — {company}\n"]
    for date, item in hits:
        lines += [_fmt_item(item, date_label=date), ""]
    return "\n".join(lines)


@mcp.tool
def search(keyword: str, company: str = "", date_from: str = DATE_MIN, date_to: str = DATE_MAX) -> str:
    """Full-text search across headlines and summaries. Optional filters: company name, date_from, date_to."""
    needle = keyword.strip().lower()
    wanted_company = company.strip().lower()
    hits = []
    for briefing in _in_range(_fetch(), date_from, date_to):
        for item in briefing["items"]:
            if wanted_company and item.get("company", "").lower() != wanted_company:
                continue
            if needle in item.get("headline", "").lower() or needle in item.get("summary", "").lower():
                hits.append((briefing["_date"], item))
    if not hits:
        return f"No results for '{keyword}'."
    lines = [f"{len(hits)} results for '{keyword}'\n"]
    for date, item in hits[:SEARCH_RESULT_LIMIT]:
        lines += [_fmt_item(item, date_label=date), ""]
    if len(hits) > SEARCH_RESULT_LIMIT:
        lines.append(f"... {len(hits) - SEARCH_RESULT_LIMIT} more results not shown.")
    return "\n".join(lines)


@mcp.tool
def compare_companies(date_from: str, date_to: str = "") -> str:
    """Side-by-side headline comparison for all companies over a date or range.

    date_to defaults to date_from when omitted (single day).
    """
    date_from = date_from.strip()
    date_to = date_to.strip() or date_from
    subset = _in_range(_fetch(), date_from, date_to)
    if not subset:
        return f"No briefings between {date_from} and {date_to}."
    by_company: dict[str, list[tuple[str, dict[str, Any]]]] = {}
    for briefing in subset:
        for item in briefing["items"]:
            by_company.setdefault(item.get("company", "?"), []).append((briefing["_date"], item))
    lines = [f"Company comparison  {date_from} → {date_to}\n"]
    for company, entries in sorted(by_company.items()):
        lines += [f"{'=' * 52}", f"  {company}  ({len(entries)} headlines)", f"{'=' * 52}"]
        for date, item in entries:
            lines.append(f"  [{date}] {item['headline']}")
        lines.append("")
    return "\n".join(lines)


@mcp.tool
def get_trending(date_from: str, date_to: str) -> str:
    """Rank companies by number of briefing items in a date range (YYYY-MM-DD)."""
    subset = _in_range(_fetch(), date_from.strip(), date_to.strip())
    if not subset:
        return f"No briefings between {date_from} and {date_to}."
    counts = Counter(item.get("company", "?") for briefing in subset for item in briefing["items"])
    lines = [f"Coverage ranking  {date_from} → {date_to}  ({len(subset)} days)\n"]
    for rank, (company, count) in enumerate(counts.most_common(), 1):
        lines.append(f"  {rank}. {company:<12}  {count:>3} items  {'█' * count}")
    return "\n".join(lines)


if __name__ == "__main__":
    mcp.run(transport="http", host="0.0.0.0", port=8080)  # nosec B104 — playground server, intentional bind
