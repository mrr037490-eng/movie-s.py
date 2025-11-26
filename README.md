import logging
import sqlite3
import requests
from bs4 import BeautifulSoup
from io import BytesIO
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import A4
from datetime import datetime
from googletrans import Translator
from telegram import InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Updater, CommandHandler, CallbackQueryHandler, MessageHandler, Filters

# ---------------- Configuration ----------------
BOT_TOKEN = "8224595879:AAHyovwlk5a81lWO-vNq_WC6mgF2SZugBkQ"  # ضع التوكن الجديد هنا
DB_PATH = "movies.db"

# --------------- Logging -----------------
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# --------------- DB Init -----------------
def init_db():
    conn = sqlite3.connect(DB_PATH)
    cur = conn.cursor()
    cur.execute("""
    CREATE TABLE IF NOT EXISTS movies (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT,
        url TEXT,
        summary TEXT,
        poster TEXT,
        saved_at TEXT,
        hits INTEGER DEFAULT 1
    )
    """)
    conn.commit()
    conn.close()

init_db()

# --------------- Translator ----------------
translator = Translator()

# --------------- Utilities ----------------
HEADERS = {"User-Agent": "Mozilla/5.0"}

def find_cinemana_pages(query, max_results=3):
    q = query.replace(" ", "+") + "+site:cinemana.shabakaty.com"
    google_url = f"https://www.google.com/search?q={q}"
    r = requests.get(google_url, headers=HEADERS, timeout=10)
    soup = BeautifulSoup(r.text, "html.parser")
    urls = []
    for a in soup.find_all("a", href=True):
        href = a["href"]
        if "cinemana.shabakaty.com" in href:
            if "/url?q=" in href:
                clean = href.split("/url?q=")[1].split("&")[0]
            else:
                clean = href
            if clean not in urls:
                urls.append(clean)
            if len(urls) >= max_results:
                break
    return urls

def scrape_cinemana(url):
    r = requests.get(url, headers=HEADERS, timeout=10)
    soup = BeautifulSoup(r.text, "html.parser")
    data = {"url": url}

    h1 = soup.find("h1")
    data["title"] = h1.get_text(strip=True) if h1 else "غير معروف"

    story = ""
    meta = soup.find("meta", {"name":"description"}) or soup.find("meta", {"property":"og:description"})
    if meta and meta.get("content"):
        story = meta.get("content")
    else:
        p_tags = soup.find_all("p")
        for p in p_tags:
            txt = p.get_text(strip=True)
            if len(txt) > 80:
                story = txt
                break
        if not story and p_tags:
            story = p_tags[0].get_text(strip=True)
    data["story"] = story

    og = soup.find("meta", {"property":"og:image"})
    if og and og.get("content"):
        data["poster"] = og.get("content")
    else:
        img = soup.find("img")
        data["poster"] = img.get("src") if img and img.get("src") else None

    actors = []
    for sel in (".cast li", ".actors li", ".credits li"):
        items = soup.select(sel)
        if items:
            for it in items[:20]:
                actors.append(it.get_text(strip=True))
    if not actors:
        for a in soup.find_all(["a","span"]):
            txt = a.get_text(strip=True)
            if txt and len(txt) > 2 and len(txt.split()) <= 3 and txt[0].isupper():
                actors.append(txt)
        actors = list(dict.fromkeys(actors))[:10]
    data["actors"] = actors
    return data

def save_movie_to_db(title, url, summary, poster):
    conn = sqlite3.connect(DB_PATH)
    cur = conn.cursor()
    cur.execute("SELECT id,hits FROM movies WHERE url=?", (url,))
    row = cur.fetchone()
    if row:
        cur.execute("UPDATE movies SET hits=hits+1 WHERE id=?", (row[0],))
        movie_id = row[0]
    else:
        saved_at = datetime.utcnow().isoformat()
        cur.execute("INSERT INTO movies (title,url,summary,poster,saved_at) VALUES (?,?,?,?,?)",
                    (title,url,summary,poster,saved_at))
        movie_id = cur.lastrowid
    conn.commit()
    conn.close()
    return movie_id

def list_saved_movies(limit=20):
    conn = sqlite3.connect(DB_PATH)
    cur = conn.cursor()
    cur.execute("SELECT id,title,url,saved_at,hits FROM movies ORDER BY saved_at DESC LIMIT ?", (limit,))
    rows = cur.fetchall()
    conn.close()
    return rows

def create_pdf_bytes(movie):
    buffer = BytesIO()
    c = canvas.Canvas(buffer, pagesize=A4)
    w,h = A4
    margin = 50
    y = h - margin
    c.setFont("Helvetica-Bold", 16)
    c.drawString(margin, y, movie.get("title","No Title"))
    y -= 30
    c.setFont("Helvetica", 12)
    c.drawString(margin, y, "Summary:")
    y -= 20
    text = movie.get("story","")
    for line in split_text_lines(text, 90):
        c.drawString(margin, y, line)
        y -= 14
        if y < margin+80:
            c.showPage()
            y = h - margin
    y -= 10
    c.drawString(margin, y, "Actors:")
    y -= 20
    for actor in movie.get("actors",[]):
        c.drawString(margin+10, y, "- " + actor)
        y -= 14
        if y < margin+80:
            c.showPage()
            y = h - margin
    c.save()
    buffer.seek(0)
    return buffer

def split_text_lines(text, max_chars=90):
    words = text.split()
    lines=[]
    cur=""
    for w in words:
        if len(cur)+len(w)+1 <= max_chars:
            cur = (cur+" "+w).strip()
        else:
            lines.append(cur)
            cur = w
    if cur:
        lines.append(cur)
    return lines

# ---------------- Bot Handlers ----------------
def start(update, context):
    update.message.reply_text("مرحباً! أرسل /search <اسم الفيلم> للبحث في Cinemana.")

def search_cmd(update, context):
    if context.args:
        query = " ".join(context.args)
    else:
        update.message.reply_text("استخدم: /search <اسم الفيلم>")
        return

    update.message.reply_text(f"🔍 جارٍ البحث عن: {query}")
    pages = find_cinemana_pages(query, max_results=3)
    if not pages:
        update.message.reply_text("❌ لا توجد نتائج.")
        return

    keyboard = []
    context.user_data['last_results'] = []

    for i, url in enumerate(pages):
        info = scrape_cinemana(url)
        context.user_data['last_results'].append(info)

        buttons = [
            InlineKeyboardButton("تفاصيل", callback_data=f"details|{i}"),
            InlineKeyboardButton("PDF", callback_data=f"pdf|{i}"),
            InlineKeyboardButton("ترجم", callback_data=f"translate|{i}"),
            InlineKeyboardButton("حفظ", callback_data=f"save|{i}"),
            InlineKeyboardButton("▶️ تشغيل", callback_data=f"play|{i}")
        ]
        keyboard.append(buttons)

    update.message.reply_text("اختر إجراء:", reply_markup=InlineKeyboardMarkup(keyboard))

def callback_handler(update, context):
    query = update.callback_query
    query.answer()

    action, idx = query.data.split("|")
    idx = int(idx)
    movie = context.user_data['last_results'][idx]

    if action == "details":
        text = f"🎬 *{movie['title']}*\n\n📄 {movie['story']}\n\n"
        if movie.get('actors'):
            text += "👥 الممثلون:\n" + "\n".join(f"- {a}" for a in movie['actors'])
        if movie.get('poster'):
            context.bot.send_photo(chat_id=query.message.chat_id, photo=movie['poster'], caption=text, parse_mode="Markdown")
        else:
            query.message.reply_text(text, parse_mode="Markdown")

    elif action == "pdf":
        pdf = create_pdf_bytes(movie)
        name = f"{movie['title']}.pdf"
        context.bot.send_document(chat_id=query.message.chat_id, document=pdf, filename=name)

    elif action == "translate":
        text = movie["story"]
        if not text:
            query.message.reply_text("لا يوجد نص للترجمة.")
            return
        lang = translator.detect(text).lang
        target = "ar" if lang != "ar" else "en"
        translated = translator.translate(text, dest=target).text
        query.message.reply_text(f"🔁 الترجمة:\n\n{translated}")

    elif action == "save":
        movie_id = save_movie_to_db(movie["title"], movie["url"], movie["story"], movie["poster"])
        query.message.reply_text(f"✔️ تم حفظ الفيلم (ID: {movie_id})")

    elif action == "play":
        url = movie["url"]
        query.message.reply_text(f"🎬 افتح هذا الرابط لتشغيل الفيلم:\n{url}")

def saved_cmd(update, context):
    movies = list_saved_movies()
    if not movies:
        update.message.reply_text("لا توجد أفلام محفوظة.")
        return

    lines = [f"{m[0]}) {m[1]} — hits: {m[4]}" for m in movies]
    update.message.reply_text("📁 الأفلام المحفوظة:\n\n" + "\n".join(lines))

def unknown(update, context):
    update.message.reply_text("الأمر غير معروف. استخدم /search <اسم الفيلم>")

def main():
    updater = Updater(BOT_TOKEN, use_context=True)
    dp = updater.dispatcher

    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CommandHandler("search", search_cmd))
    dp.add_handler(CommandHandler("saved", saved_cmd))
    dp.add_handler(CallbackQueryHandler(callback_handler))
    dp.add_handler(MessageHandler(Filters.command, unknown))

    updater.start_polling()
    updater.idle()

if __name__ == "__main__":
    main()
