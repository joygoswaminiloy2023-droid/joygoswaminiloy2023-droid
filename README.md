#!/usr/bin/env python3
"""Generate a self-hosted GitHub activity graph (SVG). No third-party services."""
import datetime as dt
import json
import math
import os
import urllib.request

LOGIN = os.environ.get("GH_LOGIN", "joygoswaminiloy2023-droid")
DAYS = 30
OUT = os.environ.get("OUT", "assets/activity-graph.svg")

QUERY = """
query($login: String!, $from: DateTime!, $to: DateTime!) {
  user(login: $login) {
    contributionsCollection(from: $from, to: $to) {
      contributionCalendar {
        weeks { contributionDays { date contributionCount } }
      }
    }
  }
}
"""


def fetch_counts(token):
    today = dt.datetime.now(dt.timezone.utc).date()
    start = today - dt.timedelta(days=DAYS - 1)
    payload = {
        "query": QUERY,
        "variables": {
            "login": LOGIN,
            "from": f"{start.isoformat()}T00:00:00Z",
            "to": f"{today.isoformat()}T23:59:59Z",
        },
    }
    req = urllib.request.Request(
        "https://api.github.com/graphql",
        data=json.dumps(payload).encode(),
        headers={
            "Authorization": f"bearer {token}",
            "Content-Type": "application/json",
            "User-Agent": "activity-graph-generator",
        },
    )
    with urllib.request.urlopen(req, timeout=30) as r:
        data = json.load(r)
    if "errors" in data or not data.get("data", {}).get("user"):
        raise SystemExit(f"GitHub API error: {data}")

    by_date = {}
    weeks = data["data"]["user"]["contributionsCollection"]["contributionCalendar"]["weeks"]
    for w in weeks:
        for d in w["contributionDays"]:
            by_date[d["date"]] = d["contributionCount"]

    days = [start + dt.timedelta(days=i) for i in range(DAYS)]
    return [(d, by_date.get(d.isoformat(), 0)) for d in days]


def render(points):
    W, H = 900, 320
    L, R, T, B = 55, 30, 70, 50
    n = len(points)
    counts = [c for _, c in points]
    ymax = max(4, max(counts))
    ymax = int(math.ceil(ymax / 4) * 4)
    total = sum(counts)

    def x(i):
        return L + i * (W - L - R) / (n - 1)

    def y(v):
        return T + (H - T - B) * (1 - v / ymax)

    coords = [(x(i), y(c)) for i, c in enumerate(counts)]
    line = f"M{coords[0][0]:.1f},{coords[0][1]:.1f}"
    for (px, py), (cx, cy) in zip(coords, coords[1:]):
        mx = (px + cx) / 2
        line += f" C{mx:.1f},{py:.1f} {mx:.1f},{cy:.1f} {cx:.1f},{cy:.1f}"
    base = H - B
    area = f"{line} L{coords[-1][0]:.1f},{base} L{coords[0][0]:.1f},{base} Z"

    grid = ""
    for k in range(5):
        v = ymax * k / 4
        gy = y(v)
        grid += (
            f'<line x1="{L}" y1="{gy:.1f}" x2="{W - R}" y2="{gy:.1f}" stroke="#30363d" stroke-dasharray="4 4"/>'
            f'<text x="{L - 10}" y="{gy + 4:.1f}" text-anchor="end" fill="#8b949e" font-size="11">{int(v)}</text>'
        )

    labels = ""
    for i, (d, _) in enumerate(points):
        if i % 5 == 0 or i == n - 1:
            labels += (
                f'<text x="{x(i):.1f}" y="{H - B + 22}" text-anchor="middle" '
                f'fill="#8b949e" font-size="11">{d.strftime("%b %d")}</text>'
            )

    dots = "".join(
        f'<circle cx="{cx:.1f}" cy="{cy:.1f}" r="3.5" fill="#0d1117" stroke="#92FE9D" stroke-width="2"/>'
        for cx, cy in coords
    )

    return f"""<svg xmlns="http://www.w3.org/2000/svg" width="{W}" height="{H}" viewBox="0 0 {W} {H}" font-family="'Segoe UI', Ubuntu, Arial, sans-serif">
  <defs>
    <linearGradient id="line" x1="0" y1="0" x2="1" y2="0">
      <stop offset="0%" stop-color="#00C9FF"/><stop offset="100%" stop-color="#92FE9D"/>
    </linearGradient>
    <linearGradient id="fill" x1="0" y1="0" x2="0" y2="1">
      <stop offset="0%" stop-color="#00C9FF" stop-opacity="0.35"/>
      <stop offset="100%" stop-color="#92FE9D" stop-opacity="0"/>
    </linearGradient>
  </defs>
  <rect width="{W}" height="{H}" rx="12" fill="#0d1117"/>
  <text x="{L}" y="34" fill="#00C9FF" font-size="20" font-weight="700">{LOGIN}'s Contribution Graph</text>
  <text x="{L}" y="54" fill="#8b949e" font-size="12">Last {DAYS} days · {total} contributions</text>
  {grid}
  <path d="{area}" fill="url(#fill)"/>
  <path d="{line}" fill="none" stroke="url(#line)" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"/>
  {dots}
  {labels}
</svg>
"""


if __name__ == "__main__":
    token = os.environ["GH_TOKEN"]
    svg = render(fetch_counts(token))
    os.makedirs(os.path.dirname(OUT), exist_ok=True)
    with open(OUT, "w", encoding="utf-8") as f:
        f.write(svg)
    print(f"Wrote {OUT}")
