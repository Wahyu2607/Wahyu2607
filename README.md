"""pixel-profile: converts a portrait photo into ASCII symbol art and renders
animated pixel-art SVGs (banner.svg, status.svg, divider.svg, footer.svg)
for a GitHub profile README.

Usage:
    pip install pillow
    python pixel_profile.py --image profile_clean.png --out assets
"""
import argparse
import json
import os
import random
import sys
from xml.sax.saxutils import escape

try:
    from PIL import Image, ImageEnhance
except ImportError:
    sys.exit("Install Pillow terlebih dahulu: pip install pillow")

OUT = "assets"

# Urutan simbol dari paling gelap (latar) ke paling terang (highlight wajah)
ASCII_SYMBOLS = " .:-=+*#%@"

DEFAULTS = {
    "title": "WAHYU PAMUNGKAS",
    "handle": "Wahyu2607",
    "tagline": "DATA SCIENCE · ML · WEB APPS",
    "card_title": "SYS.USR: WAHYU2607",
    "player": "WAHYU2607",
    "fields": [
        ["NAME", "WAHYU PAMUNGKAS"],
        ["FOCUS", "DATA & ML"],
        ["BASE", "JAKARTA, ID"],
    ],
    "stack": ["PYTHON", "XGBOOST", "SHAP", "STREAMLIT", "SQL", "FIGMA"],
    "status": "ONLINE",
    "quest": "BUILDING PREDICTIVE ML APPS",
    "footer": "THANKS FOR VISITING *",
    "footer_hint": "PRESS START",
    "theme": "forest",
    "image": "profile_clean.png",
}

# ---------------------------------------------------------------- palette
INK = "#050a07"
CARD = "#08120c"
CARD_LINE = "#1f4d31"
CREAM = "#e8ffef"
PINK = "#22c55e"
SAKURA = "#4ade80"
SAKURA_DARK = "#16a34a"
LAVENDER = "#86efac"
CYAN = "#34d399"
MINT = "#8cffc1"
PEACH = "#6ee7b7"
YELLOW = "#bbf7d0"
MUTED = "#5e8f70"
WINDOW = "#4ade80"
BUILDING = "#030705"
SKY = ["#030705", "#040a07", "#060e09", "#07120c", "#09170f", "#0b1c12",
       "#0d2216", "#10281a", "#132f1e", "#163723"]
CLOUD = "#123320"
PANEL = "#0b1a11"
MOON_LIGHT = "#dcfce7"
MOON_DARK = "#86efac"

THEMES = {
    "forest": {},
    "dusk": {
        "INK": "#1b1035", "CARD": "#1c1433", "CARD_LINE": "#3a2b63", "PANEL": "#241a42",
        "CREAM": "#fff3d6", "PINK": "#ff77a8", "SAKURA": "#ffb7c5", "SAKURA_DARK": "#ff8fab",
        "LAVENDER": "#c3a6ff", "CYAN": "#7ae7ff", "MUTED": "#8b7fb8", "WINDOW": "#ffd56b",
        "BUILDING": "#120822", "CLOUD": "#4a2e73", "MOON_LIGHT": "#fff1c1", "MOON_DARK": "#f2d98d",
        "SKY": ["#120a26", "#170d2e", "#1d1037", "#241340", "#2d174a", "#381b53",
                "#45205c", "#542564", "#662a6b", "#7a3070"],
    },
    "ocean": {
        "INK": "#06202e", "CARD": "#0b2233", "CARD_LINE": "#1f4d66", "PANEL": "#0f2e44",
        "CREAM": "#e8fbff", "PINK": "#3fd0ff", "SAKURA": "#9ff3ff", "SAKURA_DARK": "#5fd8f0",
        "LAVENDER": "#8fb8ff", "CYAN": "#7affd8", "MUTED": "#6f97ad", "WINDOW": "#ffe9a3",
        "BUILDING": "#04121c", "CLOUD": "#1d4a66", "MOON_LIGHT": "#e8fbff", "MOON_DARK": "#b9dff0",
        "SKY": ["#03101c", "#051726", "#071e31", "#0a263c", "#0d2f48", "#103a55",
                "#134662", "#16536f", "#1a617c", "#1e7089"],
    },
}


def apply_theme(name):
    globals().update(THEMES[name])
    globals()["CHIP_COLORS"] = [CYAN, YELLOW, MINT, LAVENDER, PEACH, SAKURA]


# ---------------------------------------------------------------- konversi foto ke simbol ASCII
def image_to_ascii_lines(image_path, cols=42, rows=32):
    """Mengubah foto menjadi baris-baris susunan simbol ASCII sesuai tingkat kecerahan."""
    if not os.path.exists(image_path):
        raise FileNotFoundError(
            f"File foto '{image_path}' tidak ditemukan. "
            f"Simpan foto dari @profile sebagai '{image_path}' di folder yang sama."
        )
    img = Image.open(image_path).convert("L")  # grayscale
    # Tingkatkan kontras sedikit agar detail wajah dan kacamata terbaca jelas dalam simbol
    img = ImageEnhance.Contrast(img).enhance(1.45)

    # Crop bagian tengah-atas (fokus wajah & bahu) menjadi rasio persegi
    w, h = img.size
    side = min(w, h)
    left = (w - side) // 2
    top = max(0, int((h - side) * 0.25))
    img = img.crop((left, top, left + side, top + side))

    img = img.resize((cols, rows), Image.Resampling.LANCZOS)
    pixels = list(img.getdata())

    lines = []
    scale = len(ASCII_SYMBOLS) - 1
    for r in range(rows):
        row_chars = []
        for c in range(cols):
            val = pixels[r * cols + c]
            idx = int((val / 255.0) * scale)
            row_chars.append(ASCII_SYMBOLS[idx])
        lines.append("".join(row_chars))
    return lines


def render_ascii_portrait_svg(lines, x, y, box_w, box_h, font_size=6.2):
    """Menyusun baris simbol ASCII menjadi elemen teks SVG beranimasi scanline."""
    line_height = round(box_h / max(1, len(lines)), 2)
    tspans = []
    for i, line in enumerate(lines):
        escaped = escape(line)
        # Variasikan opasitas tiap baris sedikit untuk efek terminal CRT
        opacity = "0.95" if i % 2 == 0 else "0.78"
        tspans.append(
            f'<tspan x="{x + box_w / 2}" dy="{line_height if i > 0 else 0}" '
            f'fill-opacity="{opacity}">{escaped}</tspan>'
        )

    scan_y_end = y + box_h - 6
    return (
        f'<g>'
        f'<text x="{x + box_w / 2}" y="{y + line_height}" '
        f'font-family="monospace" font-size="{font_size}px" font-weight="bold" '
        f'letter-spacing="1.2px" text-anchor="middle" xml:space="preserve" fill="{SAKURA}">'
        f'{"".join(tspans)}'
        f'</text>'
        f'<rect x="{x + 4}" y="{y}" width="{box_w - 8}" height="3" fill="{CREAM}" opacity="0.25">'
        f'<animate attributeName="y" values="{y};{scan_y_end};{y}" dur="4.2s" repeatCount="indefinite"/>'
        f'</rect>'
        f'</g>'
    )


# ---------------------------------------------------------------- 5x7 pixel font
FONT = {
    "A": [" ### ", "#   #", "#   #", "#####", "#   #", "#   #", "#   #"],
    "B": ["#### ", "#   #", "#   #", "#### ", "#   #", "#   #", "#### "],
    "C": [" ### ", "#   #", "#    ", "#    ", "#    ", "#   #", " ### "],
    "D": ["#### ", "#   #", "#   #", "#   #", "#   #", "#   #", "#### "],
    "E": ["#####", "#    ", "#    ", "#### ", "#    ", "#    ", "#####"],
    "F": ["#####", "#    ", "#    ", "#### ", "#    ", "#    ", "#    "],
    "G": [" ### ", "#   #", "#    ", "# ###", "#   #", "#   #", " ####"],
    "H": ["#   #", "#   #", "#   #", "#####", "#   #", "#   #", "#   #"],
    "I": ["#####", "  #  ", "  #  ", "  #  ", "  #  ", "  #  ", "#####"],
    "J": ["  ###", "   # ", "   # ", "   # ", "   # ", "#  # ", " ##  "],
    "K": ["#   #", "#  # ", "# #  ", "##   ", "# #  ", "#  # ", "#   #"],
    "L": ["#    ", "#    ", "#    ", "#    ", "#    ", "#    ", "#####"],
    "M": ["#   #", "## ##", "# # #", "# # #", "#   #", "#   #", "#   #"],
    "N": ["#   #", "##  #", "# # #", "#  ##", "#   #", "#   #", "#   #"],
    "O": [" ### ", "#   #", "#   #", "#   #", "#   #", "#   #", " ### "],
    "P": ["#### ", "#   #", "#   #", "#### ", "#    ", "#    ", "#    "],
    "Q": [" ### ", "#   #", "#   #", "#   #", "# # #", "#  # ", " ## #"],
    "R": ["#### ", "#   #", "#   #", "#### ", "# #  ", "#  # ", "#   #"],
    "S": [" ####", "#    ", "#    ", " ### ", "    #", "    #", "#### "],
    "T": ["#####", "  #  ", "  #  ", "  #  ", "  #  ", "  #  ", "  #  "],
    "U": ["#   #", "#   #", "#   #", "#   #", "#   #", "#   #", " ### "],
    "V": ["#   #", "#   #", "#   #", "#   #", "#   #", " # # ", "  #  "],
    "W": ["#   #", "#   #", "#   #", "# # #", "# # #", "## ##", "#   #"],
    "X": ["#   #", "#   #", " # # ", "  #  ", " # # ", "#   #", "#   #"],
    "Y": ["#   #", "#   #", " # # ", "  #  ", "  #  ", "  #  ", "  #  "],
    "Z": ["#####", "    #", "   # ", "  #  ", " #   ", "#    ", "#####"],
    "0": [" ### ", "#   #", "#  ##", "# # #", "##  #", "#   #", " ### "],
    "1": ["  #  ", " ##  ", "  #  ", "  #  ", "  #  ", "  #  ", " ### "],
    "2": [" ### ", "#   #", "    #", "   # ", "  #  ", " #   ", "#####"],
    "3": ["#####", "   # ", "  #  ", "   # ", "    #", "#   #", " ### "],
    "4": ["   # ", "  ## ", " # # ", "#  # ", "#####", "   # ", "   # "],
    "5": ["#####", "#    ", "#### ", "    #", "    #", "#   #", " ### "],
    "6": ["  ## ", " #   ", "#    ", "#### ", "#   #", "#   #", " ### "],
    "7": ["#####", "    #", "   # ", "  #  ", " #   ", " #   ", " #   "],
    "8": [" ### ", "#   #", "#   #", " ### ", "#   #", "#   #", " ### "],
    "9": [" ### ", "#   #", "#   #", " ####", "    #", "   # ", " ##  "],
    " ": ["     "] * 7,
    ">": ["#    ", " #   ", "  #  ", "   # ", "  #  ", " #   ", "#    "],
    "_": ["     "] * 6 + ["#####"],
    ".": ["     "] * 5 + [" ##  ", " ##  "],
    ":": ["     ", " ##  ", " ##  ", "     ", " ##  ", " ##  ", "     "],
    "/": ["    #", "    #", "   # ", "  #  ", " #   ", "#    ", "#    "],
    "-": ["     ", "     ", "     ", "#####", "     ", "     ", "     "],
    "!": ["  #  ", "  #  ", "  #  ", "  #  ", "  #  ", "     ", "  #  "],
    "?": [" ### ", "#   #", "    #", "   # ", "  #  ", "     ", "  #  "],
    "*": ["     ", " # # ", "#####", "#####", " ### ", "  #  ", "     "],
    "&": [" ##  ", "#  # ", "# #  ", " #   ", "# # #", "#  # ", " ## #"],
    "(": ["   # ", "  #  ", " #   ", " #   ", " #   ", "  #  ", "   # "],
    ")": [" #   ", "  #  ", "   # ", "   # ", "   # ", "  #  ", " #   "],
    "[": [" ### ", " #   ", " #   ", " #   ", " #   ", " #   ", " ### "],
    "]": [" ### ", "   # ", "   # ", "   # ", "   # ", "   # ", " ### "],
    "<": ["    #", "   # ", "  #  ", " #   ", "  #  ", "   # ", "    #"],
    "+": ["     ", "  #  ", "  #  ", "#####", "  #  ", "  #  ", "     "],
    "=": ["     ", "     ", "#####", "     ", "#####", "     ", "     "],
    ",": ["     "] * 5 + ["  #  ", " #   "],
    "'": ["  #  ", "  #  ", "     ", "     ", "     ", "     ", "     "],
    "#": [" # # ", " # # ", "#####", " # # ", "#####", " # # ", " # # "],
    "@": [" ### ", "#   #", "# ###", "# # #", "# ###", "#    ", " ####"],
    "$": ["  #  ", " ####", "# #  ", " ### ", "  # #", "#### ", "  #  "],
    "%": ["##  #", "##  #", "   # ", "  #  ", " #   ", "#  ##", "#  ##"],
    "|": ["  #  "] * 7,
    "~": ["     ", "     ", " #   ", "# # #", "   # ", "     ", "     "],
    "·": ["     ", "     ", "     ", "  #  ", "     ", "     ", "     "],
}


def text_width(s, px, gap=1):
    return (len(s) * (5 + gap) - gap) * px


def text_path(s, x, y, px, gap=1):
    d = []
    cx = x
    for ch in s.upper():
        for r, row in enumerate(FONT.get(ch, FONT["?"])):
            c = 0
            while c < 5:
                if row[c] == "#":
                    start = c
                    while c < 5 and row[c] == "#":
                        c += 1
                    w = (c - start) * px
                    d.append(f"M{cx + start * px} {y + r * px}h{w}v{px}h{-w}z")
                else:
                    c += 1
        cx += (5 + gap) * px
    return "".join(d)


def sprite_paths(rows, x, y, px):
    out = {}
    for r, row in enumerate(rows):
        c = 0
        while c < len(row):
            key = row[c]
            if key == ".":
                c += 1
                continue
            start = c
            while c < len(row) and row[c] == key:
                c += 1
            w = (c - start) * px
            out.setdefault(key, []).append(f"M{x + start * px} {y + r * px}h{w}v{px}h{-w}z")
    return {k: "".join(v) for k, v in out.items()}


CLOUD = [
    "......######..........",
    "...############.......",
    "..################....",
    ".######################",
    "########################",
]
SPARKLE = ["..#..", "..#..", "##.##", "..#..", "..#.."]
MOON = 9


def moon(cx, cy, px):
    rows = []
    for dy in range(-MOON, MOON + 1):
        row = ""
        for dx in range(-MOON, MOON + 1):
            row += "M" if dx * dx + dy * dy <= MOON * MOON else "."
        rows.append(row)
    for (r, c) in [(5, 6), (5, 7), (6, 6), (10, 12), (11, 12), (11, 13), (13, 5)]:
        rows[r] = rows[r][:c] + "C" + rows[r][c + 1:]
    p = sprite_paths(rows, cx - MOON * px, cy - MOON * px, px)
    return (f'<circle cx="{cx}" cy="{cy}" r="{MOON * px * 2}" fill="{YELLOW}" opacity="0.10"/>'
            f'<path d="{p["M"]}" fill="{MOON_LIGHT}"/><path d="{p["C"]}" fill="{MOON_DARK}"/>')


def stars(rng, n, w, h, top=0, avoid=()):
    s = []
    for _ in range(n):
        while True:
            x = rng.randrange(0, w // 4) * 4
            y = top + rng.randrange(0, h // 4) * 4
            if not any(x0 - 8 <= x <= x1 + 8 and y0 - 8 <= y <= y1 + 8 for x0, y0, x1, y1 in avoid):
                break
        size = rng.choice([4, 4, 4, 8])
        dur = round(rng.uniform(1.4, 4.0), 1)
        begin = round(rng.uniform(0, 3), 1)
        color = rng.choice([CREAM, CREAM, "#ffffff", LAVENDER, CYAN])
        s.append(f'<rect x="{x}" y="{y}" width="{size}" height="{size}" fill="{color}">'
                 f'<animate attributeName="opacity" values="1;0.25;1" dur="{dur}s" begin="-{begin}s" '
                 f'calcMode="discrete" repeatCount="indefinite"/></rect>')
    return "".join(s)


def sparkle(x, y, px, color, dur, begin):
    p = sprite_paths(SPARKLE, x, y, px)["#"]
    return (f'<path d="{p}" fill="{color}"><animate attributeName="opacity" values="1;0;0;1" '
            f'keyTimes="0;0.4;0.8;1" dur="{dur}s" begin="-{begin}s" calcMode="discrete" repeatCount="indefinite"/></path>')


def cloud(x, y, px, dur, begin, travel):
    p = sprite_paths(CLOUD, 0, 0, px)["#"]
    return (f'<g transform="translate({x} {y})"><g opacity="0.55">'
            f'<animateTransform attributeName="transform" type="translate" from="0 0" to="{travel} 0" dur="{dur}s" '
            f'begin="-{begin}s" repeatCount="indefinite"/><path d="{p}" fill="{CLOUD}"/></g></g>')


def sky(w, h, rx):
    band = h / len(SKY)
    rects = "".join(f'<rect y="{round(i * band)}" width="{w}" height="{round(band) + 1}" fill="{c}"/>'
                    for i, c in enumerate(SKY))
    return f'<clipPath id="frame"><rect width="{w}" height="{h}" rx="{rx}"/></clipPath>', rects


def skyline(rng, w, base, px=4):
    s = []
    x = -10
    while x < w:
        bw = rng.randrange(10, 24) * px
        bh = rng.randrange(8, 20) * px
        top = base - bh
        s.append(f'<rect x="{x}" y="{top}" width="{bw}" height="{bh + 40}" fill="{BUILDING}"/>')
        for wy in range(top + 2 * px, base - px, 3 * px):
            for wx in range(x + 2 * px, x + bw - 2 * px, 3 * px):
                if rng.random() < 0.28:
                    dur = round(rng.uniform(3, 8), 1)
                    s.append(f'<rect x="{wx}" y="{wy}" width="{px}" height="{px + 2}" fill="{WINDOW}">'
                             f'<animate attributeName="opacity" values="1;0.15;1" dur="{dur}s" calcMode="discrete" '
                             f'begin="-{rng.uniform(0, dur):.1f}s" repeatCount="indefinite"/></rect>')
        x += bw + rng.randrange(0, 3) * px
    return "".join(s)


def svg(w, h, body, label):
    label = escape(label, {'"': "&quot;"})
    return (f'<svg xmlns="http://www.w3.org/2000/svg" width="{w}" height="{h}" viewBox="0 0 {w} {h}" '
            f'shape-rendering="crispEdges" role="img" aria-label="{label}">{body}</svg>\n')


def fit_px(text, max_width, largest, smallest=1):
    for px in range(largest, smallest - 1, -1):
        if text_width(text, px) <= max_width:
            return px
    return smallest


def clean(text):
    return "".join(ch if ch in FONT else "?" for ch in str(text).upper())


def write(name, content):
    with open(os.path.join(OUT, name), "w", encoding="utf-8", newline="\n") as f:
        f.write(content)
    print(f"{name}: {len(content) / 1024:.1f} KB")


# ---------------------------------------------------------------- banner
def banner(cfg, ascii_lines):
    rng = random.Random(7)
    W, H = 1200, 340
    clip, bands = sky(W, H, 24)
    title = clean(cfg["title"])
    tx = 64
    ty = 86
    room = 760
    tpx = fit_px(title, room, 9, 4)
    sub1 = clean("> " + cfg["handle"])
    sub2 = clean(cfg["tagline"])
    spx = min(fit_px(sub1, room - 24, 3), fit_px(sub2, room, 3))
    cur_x = tx + text_width(sub1, spx) + 3 * spx

    # Kotak potret simbol ASCII di sebelah kanan banner
    bx, by, bw, bh = 885, 36, 250, 250
    portrait_svg = (
        f'<rect x="{bx}" y="{by}" width="{bw}" height="{bh}" rx="14" fill="{INK}" stroke="{PINK}" stroke-width="2"/>'
        + render_ascii_portrait_svg(ascii_lines, bx + 6, by + 6, bw - 12, bh - 12, font_size=5.4)
    )

    body = [
        f"<defs>{clip}</defs>",
        '<g clip-path="url(#frame)">',
        bands,
        stars(rng, 40, W, 230, avoid=[(tx - 10, ty - 10, tx + room, ty + 180), (bx - 10, by - 10, bx + bw + 10, by + bh + 10)]),
        sparkle(42, 48, 4, CREAM, 2.6, 0.3),
        sparkle(830, 62, 4, CYAN, 3.2, 1.1),
        cloud(-120, 130, 8, 70, 10, 1500),
        cloud(-120, 40, 6, 95, 60, 1500),
        skyline(rng, W, H - 4),
        f'<rect y="{H - 6}" width="{W}" height="6" fill="{PINK}" opacity="0.35"/>',
        portrait_svg,
        f'<path d="{text_path(title, tx + max(2, tpx * 6 // 10), ty + max(2, tpx * 6 // 10), tpx)}" fill="{PINK}"/>',
        f'<path d="{text_path(title, tx, ty, tpx)}" fill="{CREAM}"/>',
        f'<path d="{text_path(sub1, tx, ty + 96, spx)}" fill="{CYAN}"/>',
        f'<rect x="{cur_x}" y="{ty + 96}" width="{5 * spx}" height="{7 * spx}" fill="{CYAN}">'
        f'<animate attributeName="opacity" values="1;0" dur="1s" calcMode="discrete" repeatCount="indefinite"/></rect>',
        f'<path d="{text_path(sub2, tx, ty + 134, spx)}" fill="{LAVENDER}"/>',
        "</g>",
        f'<rect x="1.5" y="1.5" width="{W - 3}" height="{H - 3}" rx="23" fill="none" stroke="{CARD_LINE}" stroke-width="3"/>',
    ]
    write("banner.svg", svg(W, H, "".join(body), cfg["title"]))


# ---------------------------------------------------------------- status card
def chip(x, y, label, color, px=2):
    tw = text_width(label, px)
    w = tw + 28
    return (f'<rect x="{x}" y="{y}" width="{w}" height="32" rx="9" fill="{color}"/>'
            f'<path d="{text_path(label, x + 14, y + 9, px)}" fill="{INK}"/>'), w


CHIP_COLORS = [CYAN, YELLOW, MINT, LAVENDER, PEACH, SAKURA]


def status_card(cfg, ascii_lines):
    W, H = 1200, 500
    cx, cy, cw, ch = 24, 16, W - 48, H - 44
    rows_x, val_x = 400, 560
    body = [
        '<defs><filter id="sh" x="-5%" y="-5%" width="110%" height="120%">'
        f'<feDropShadow dx="0" dy="8" stdDeviation="10" flood-color="#000000" flood-opacity="0.35"/></filter></defs>',
        f'<rect x="{cx}" y="{cy}" width="{cw}" height="{ch}" rx="24" fill="{CARD}" stroke="{CARD_LINE}" stroke-width="2" filter="url(#sh)"/>',
        f'<rect x="{cx + 24}" y="{cy + 22}" width="14" height="14" rx="4" fill="{PINK}"/>',
        f'<rect x="{cx + 46}" y="{cy + 22}" width="14" height="14" rx="4" fill="{YELLOW}"/>',
        f'<rect x="{cx + 68}" y="{cy + 22}" width="14" height="14" rx="4" fill="{MINT}"/>',
        f'<path d="{text_path(clean(cfg["card_title"]), (W - text_width(clean(cfg["card_title"]), 2)) // 2, cy + 22, 2)}" fill="{MUTED}"/>',
        f'<rect x="{cx}" y="{cy + 56}" width="{cw}" height="2" fill="{CARD_LINE}"/>',
        # Panel kiri berisi susunan simbol ASCII dari foto @profile
        f'<rect x="{cx + 40}" y="{cy + 82}" width="300" height="320" rx="18" fill="{INK}" stroke="{PINK}" stroke-width="2"/>',
        render_ascii_portrait_svg(ascii_lines, cx + 48, cy + 90, 284, 244, font_size=5.6),
    ]
    plate, pw = chip(0, 0, clean(cfg["player"])[:16], PINK)
    body.append(f'<g transform="translate({cx + 40 + (300 - pw) // 2} {cy + 350})">{plate}</g>')

    y0, step = cy + 98, 50
    right = cx + cw - 30
    fields = [(clean(k)[:8], clean(v)) for k, v in cfg["fields"][:3]]
    labels = [k for k, _ in fields] + ["STACK", "STATUS", "QUEST", "NEXT"]
    for i, lab in enumerate(labels):
        body.append(f'<path d="{text_path(lab, rows_x, y0 + i * step + 4, 2)}" fill="{MUTED}"/>')

    def value(text, row, color=CREAM, x=val_x):
        px = fit_px(text, right - x, 3, 2)
        body.append(f'<path d="{text_path(text, x, y0 + row * step + (3 - px) * 3, px)}" fill="{color}"/>')

    for i, (_, v) in enumerate(fields):
        value(v, i)
    row = len(fields)
    x = val_x
    for i, lab in enumerate(clean(s) for s in cfg["stack"]):
        c, w = chip(x, y0 + row * step - 6, lab, CHIP_COLORS[i % len(CHIP_COLORS)])
        if x + w > right:
            break
        body.append(c)
        x += w + 10
    sy = y0 + (row + 1) * step
    body.append(f'<rect x="{val_x}" y="{sy + 4}" width="12" height="12" fill="{MINT}">'
                f'<animate attributeName="opacity" values="1;0.2" dur="1.2s" calcMode="discrete" repeatCount="indefinite"/></rect>')
    value(clean(cfg["status"]), row + 1, MINT, val_x + 26)
    value(clean(cfg["quest"]), row + 2)
    by = y0 + (row + 3) * step - 2
    segs, sw = 24, 16
    body.append(f'<rect x="{val_x - 4}" y="{by - 4}" width="{segs * (sw + 4) + 4}" height="30" rx="6" fill="none" stroke="{CARD_LINE}" stroke-width="2"/>')
    body.append(f'<clipPath id="bar"><rect x="{val_x}" y="{by}" height="22" width="0">'
                f'<animate attributeName="width" values="{";".join(str(i * (sw + 4)) for i in range(segs + 1))};{segs * (sw + 4)}" '
                f'dur="5s" calcMode="discrete" repeatCount="indefinite"/></rect></clipPath>')
    seg_path = "".join(f"M{val_x + i * (sw + 4)} {by}h{sw}v22h{-sw}z" for i in range(segs))
    body.append(f'<path d="{seg_path}" fill="{PINK}" clip-path="url(#bar)"/>')
    body.append(f'<path d="{text_path("LOADING", val_x + segs * (sw + 4) + 20, by + 4, 2)}" fill="{MUTED}">'
                f'<animate attributeName="opacity" values="1;0.3" dur="0.8s" calcMode="discrete" repeatCount="indefinite"/></path>')
    summary = ", ".join(f"{k}: {v}" for k, v in cfg["fields"][:3])
    write("status.svg", svg(W, H, "".join(body), f"Player status. {summary}. Stack: {', '.join(cfg['stack'])}"))


# ---------------------------------------------------------------- divider & footer
def divider():
    W, H = 1200, 20
    dots = "".join(f"M{x} 8h8v4h-8z" for x in range(0, W, 16))
    comet = "M0 8h8v4h-8zM12 8h8v4h-8zM24 6h8v8h-8zM36 4h12v12h-12z"
    body = (f'<path d="{dots}" fill="{CARD_LINE}"/>'
            f'<g><animateTransform attributeName="transform" type="translate" from="-60 0" to="{W + 60} 0" dur="5s" repeatCount="indefinite"/>'
            f'<path d="{comet}" fill="{PINK}"/><rect x="40" y="8" width="4" height="4" fill="{CREAM}"/></g>')
    write("divider.svg", svg(W, H, body, ""))


def footer(cfg):
    rng = random.Random(11)
    W, H = 1200, 200
    clip, bands = sky(W, H, 24)
    msg = clean(cfg["footer"])
    px = fit_px(msg, W - 120, 5, 2)
    mx = (W - text_width(msg, px)) // 2
    start = clean(cfg["footer_hint"])
    sx = (W - text_width(start, fit_px(start, W - 120, 3))) // 2
    body = [
        f"<defs>{clip}</defs>",
        '<g clip-path="url(#frame)">',
        bands,
        stars(rng, 30, W, 140, avoid=[(mx, 46, mx + text_width(msg, px) + 4, 144)]),
        skyline(rng, W, H - 4),
        f'<path d="{text_path(msg, mx + 4, 52 + 4, px)}" fill="{PINK}"/>',
        f'<path d="{text_path(msg, mx, 52, px)}" fill="{CREAM}"/>',
        f'<path d="{text_path(start, sx, 110, fit_px(start, W - 120, 3))}" fill="{CYAN}">'
        f'<animate attributeName="opacity" values="1;0" dur="1.1s" calcMode="discrete" repeatCount="indefinite"/></path>',
        "</g>",
        f'<rect x="1.5" y="1.5" width="{W - 3}" height="{H - 3}" rx="23" fill="none" stroke="{CARD_LINE}" stroke-width="3"/>',
    ]
    write("footer.svg", svg(W, H, "".join(body), cfg["footer"].replace("*", "").strip()))


def load_config(argv=None):
    p = argparse.ArgumentParser(description="Convert photo to ASCII symbols and generate animated profile SVGs.")
    p.add_argument("--config", help="JSON config file")
    p.add_argument("--out", default="assets", help="output directory (default: assets)")
    p.add_argument("--image", help="path ke foto profil (default: profile_clean.png)")
    p.add_argument("--title", help="big banner text")
    p.add_argument("--handle", help="GitHub username")
    p.add_argument("--tagline", help="second banner line")
    p.add_argument("--stack", help="comma-separated stack chips")
    p.add_argument("--theme", choices=sorted(THEMES), help="colour theme (default: forest)")
    args = p.parse_args(argv)

    cfg = dict(DEFAULTS)
    if args.config:
        with open(args.config, encoding="utf-8") as f:
            cfg.update(json.load(f))
    for key in ("image", "title", "handle", "tagline", "theme"):
        if getattr(args, key):
            cfg[key] = getattr(args, key)
    if args.stack:
        cfg["stack"] = [s.strip() for s in args.stack.split(",") if s.strip()]
    return cfg, args.out


def main(argv=None):
    global OUT
    cfg, OUT = load_config(argv)
    apply_theme(cfg["theme"])
    os.makedirs(OUT, exist_ok=True)

    # 1. Konversi foto menjadi susunan simbol ASCII
    ascii_lines = image_to_ascii_lines(cfg["image"], cols=42, rows=32)

    # 2. Simpan versi teks mentahnya agar bisa langsung disalin ke README.md
    txt_path = os.path.join(OUT, "ascii_portrait.txt")
    with open(txt_path, "w", encoding="utf-8", newline="\n") as f:
        f.write("\n".join(ascii_lines) + "\n")
    print(f"ascii_portrait.txt: {len(ascii_lines)} baris simbol")

    # 3. Render ke SVG animasi
    banner(cfg, ascii_lines)
    status_card(cfg, ascii_lines)
    divider()
    footer(cfg)


if __name__ == "__main__":
    sys.exit(main())
