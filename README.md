from telegram import Update, ReplyKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes
import json
import os
import random

TOKEN = '7664118380:AAFXakZ4a8bDjqpB0P0HQRupFZLUwFHFmyQ'
SONGS_FILE = "songs.json"

# بارگذاری آهنگ‌ها از فایل
if os.path.exists(SONGS_FILE):
    with open(SONGS_FILE, "r", encoding="utf-8") as f:
        fa_mood_music = json.load(f)
else:
    fa_mood_music = {
        "شاد": [],
        "غمگین": [],
        "عاشق": [],
        "بی حوصله": [],
        "نمیدونم": []
    }

# برای جلوگیری از تکرار آهنگ
user_sent_songs = {}

en_mood_music = {
    "happy": fa_mood_music["شاد"],
    "sad": fa_mood_music["غمگین"],
    "love": fa_mood_music["عاشق"],
    "bored": fa_mood_music["بی حوصله"],
    "idk": fa_mood_music["نمیدونم"]
}

# ذخیره در فایل
def save_songs_to_file():
    with open(SONGS_FILE, "w", encoding="utf-8") as f:
        json.dump(fa_mood_music, f, ensure_ascii=False, indent=2)

# استارت
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [[
        "شاد", "غمگین", "عاشق"
    ], [
        "بی حوصله", "نمیدونم"
    ], [
        "happy", "sad", "love"
    ], [
        "bored", "idk"
    ]]
    reply_markup = ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
    await update.message.reply_text(
        "سلام! حس‌تو انتخاب کن تا موزیک مخصوص حالتو بفرستم:",
        reply_markup=reply_markup
    )

# هندل حال و هوا
async def handle_mood(update: Update, context: ContextTypes.DEFAULT_TYPE):
    mood = update.message.text.strip()
    user_id = update.message.from_user.id

    if user_id not in user_sent_songs:
        user_sent_songs[user_id] = set()

    mood_data = None
    mood_key = ""

    if mood in fa_mood_music:
        mood_data = fa_mood_music[mood]
        mood_key = mood
    elif mood.lower() in en_mood_music:
        mood_data = en_mood_music[mood.lower()]
        mood_key = mood.lower()

    if mood_data:
        available_songs = [song for song in mood_data if f"{mood_key}-{song['title']}" not in user_sent_songs[user_id]]

        if not available_songs:
            await update.message.reply_text("همه آهنگای این حس‌و‌حال رو دیدی! بعداً دوباره امتحان کن.")
            return

        selected_songs = random.sample(available_songs, min(3, len(available_songs)))

        for song in selected_songs:
            await update.message.reply_photo(
                photo=song["cover_url"],
                caption=f"{song['artist']} - {song['title']}"
            )
            user_sent_songs[user_id].add(f"{mood_key}-{song['title']}")
    else:
        await update.message.reply_text("حالتتو درست وارد کن: مثلاً شاد یا happy.")

# اضافه کردن آهنگ
async def add_song(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text.strip()
    try:
        _, data = text.split("اضافه", 1)
        mood, title, artist, url = [x.strip() for x in data.split("|")]

        if mood not in fa_mood_music:
            await update.message.reply_text("حس‌و‌حال وارد شده درست نیست.")
            return

        new_song = {
            "title": title,
            "artist": artist,
            "cover_url": url
        }

        fa_mood_music[mood].append(new_song)
        save_songs_to_file()

        await update.message.reply_text(f"آهنگ «{title}» با موفقیت به لیست {mood} اضافه شد!")
    except:
        await update.message.reply_text("فرمت درست نیست. مثال:\nاضافه شاد | آهنگ من | رپر من | https://cover.url")

# اجرای برنامه
app = ApplicationBuilder().token(TOKEN).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(MessageHandler(filters.TEXT & filters.Regex("^اضافه"), add_song))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_mood))
app.run_polling()
