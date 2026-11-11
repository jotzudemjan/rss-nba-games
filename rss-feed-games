import json
import os
import urllib.request
from datetime import datetime, timedelta, timezone
from zoneinfo import ZoneInfo
from xml.sax.saxutils import escape

LOCAL = ZoneInfo("Europe/Berlin")   # schaltet Sommer-/Winterzeit automatisch um
API = "https://site.api.espn.com/apis/site/v2/sports/basketball/nba/scoreboard"
WD = ["Mo", "Di", "Mi", "Do", "Fr", "Sa", "So"]

def week_range():
    """Montag 00:00 bis Sonntag 24:00 der aktuellen Woche (deutsche Zeit)."""
    today = datetime.now(LOCAL).date()
    monday = today - timedelta(days=today.weekday())
    start = datetime(monday.year, monday.month, monday.day, 0, 0, tzinfo=LOCAL)
    end = start + timedelta(days=7)
    return start, end

def fetch_day(date):
    """Alle Spiele eines US-Spieltags (JJJJ-MM-TT)."""
    url = f"{API}?dates={date.isoformat()}"
    with urllib.request.urlopen(url) as r:
        data = json.load(r)
    games = []
    for event in data.get("events", []):
        comp = event["competitions"][0]
        home = next(c["team"]["displayName"] for c in comp["competitors"] if c["homeAway"] == "home")
        away = next(c["team"]["displayName"] for c in comp["competitors"] if c["homeAway"] == "away")
        venue = comp.get("venue", {}).get("fullName", "")
        games.append({
            "utc": datetime.fromisoformat(event["date"].replace("Z", "+00:00")),
            "matchup": f"{away} vs {home}",
            "venue": venue,
        })
    return games

def fetch_week():
    start, end = week_range()
    # US-Spieltage, die deutsche Nachtspiele der Woche enthalten können:
    days = [start.date() - timedelta(days=1)] + \
           [start.date() + timedelta(days=i) for i in range(7)] + \
           [end.date()]
    games = []
    for d in sorted(set(days)):
        games.extend(fetch_day(d))
    # Nur Spiele, deren deutscher Startzeitpunkt in Mo 00:00 – So 24:00 liegt
    games = [g for g in games if start <= g["utc"].astimezone(LOCAL) < end]
    games.sort(key=lambda g: g["utc"])
    return games

def build_rss(games):
    now = datetime.now(timezone.utc).strftime("%a, %d %b %Y %H:%M:%S GMT")
    items = []
    for g in games:
        local = g["utc"].astimezone(LOCAL)
        title = f"{WD[local.weekday()]} {local:%d.%m.} {local:%H:%M} Uhr – {g['matchup']}"
        desc = g["matchup"] + (f" – {g['venue']}" if g["venue"] else "")
        items.append(f"""    <item>
      <title>{escape(title)}</title>
      <pubDate>{g['utc'].strftime('%a, %d %b %Y %H:%M:%S GMT')}</pubDate>
      <description>{escape(desc)}</description>
      <guid isPermaLink="false">{g['utc'].isoformat()}-{escape(g['matchup'])}</guid>
    </item>""")
    return f"""<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0">
  <channel>
    <title>NBA Games – aktuelle Woche (Mo–So)</title>
    <link>https://www.nba.com/schedule</link>
    <description>Ein Eintrag pro NBA-Spiel der aktuellen Woche: Startuhrzeit, Team gegen Team, ohne Ergebnis.</description>
    <lastBuildDate>{now}</lastBuildDate>
{chr(10).join(items)}
  </channel>
</rss>
"""

if __name__ == "__main__":
    games = fetch_week()
    os.makedirs("docs", exist_ok=True)
    with open("docs/feed.xml", "w", encoding="utf-8") as f:
        f.write(build_rss(games))
    print(f"{len(games)} Spiele geschrieben.")
