# anime-bot  telebot
from telebot import types
import sqlite3

TOKEN = "8744235232:AAGjTarMs0YIwr1iqAWrqAmCi4J0VjkcZoY"
ADMIN_ID = 8526850458
CHANNEL = "AniConik_1"

bot = telebot.TeleBot(TOKEN)

conn = sqlite3.connect("anime.db", check_same_thread=False)
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS anime(
id INTEGER PRIMARY KEY AUTOINCREMENT,
name TEXT,
code TEXT,
file_id TEXT
)
""")
conn.commit()


def check_sub(user_id):
    try:
        member = bot.get_chat_member("@"+CHANNEL, user_id)
        return member.status != "left"
    except:
        return False


@bot.message_handler(commands=['start'])
def start(message):

    if not check_sub(message.from_user.id):

        markup = types.InlineKeyboardMarkup()
        btn = types.InlineKeyboardButton(
            "📢 Kanalga qo'shilish",
            url="https://t.me/"+CHANNEL
        )

        markup.add(btn)

        bot.send_message(
            message.chat.id,
            "Bot ishlashi uchun kanalga qo'shiling",
            reply_markup=markup
        )
        return

    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)

    markup.add(
        "🔎 Anime qidirish",
        "📌 Kod orqali"
    )

    bot.send_message(
        message.chat.id,
        "🎌 PRO Anime Botga xush kelibsiz",
        reply_markup=markup
    )


@bot.message_handler(func=lambda m: m.text=="🔎 Anime qidirish")
def search(message):

    msg = bot.send_message(
        message.chat.id,
        "Anime nomini yozing"
    )

    bot.register_next_step_handler(msg, search_anime)


def search_anime(message):

    name = message.text

    cursor.execute(
        "SELECT name,code FROM anime WHERE name LIKE ?",
        ("%"+name+"%",)
    )

    result = cursor.fetchall()

    if result:

        for r in result:

            markup = types.InlineKeyboardMarkup()

            btn = types.InlineKeyboardButton(
                "🎬 Tomosha qilish",
                callback_data="watch_"+r[1]
            )

            markup.add(btn)

            bot.send_message(
                message.chat.id,
                f"🎬 {r[0]}\n📌 Kod: {r[1]}",
                reply_markup=markup
            )

    else:
        bot.send_message(message.chat.id,"Anime topilmadi")


@bot.callback_query_handler(func=lambda call: call.data.startswith("watch_"))
def watch(call):

    code = call.data.split("_")[1]

    cursor.execute(
        "SELECT file_id FROM anime WHERE code=?",
        (code,)
    )

    result = cursor.fetchone()

    if result:

        bot.send_video(
            call.message.chat.id,
            result[0]
        )


@bot.message_handler(func=lambda m: m.text=="📌 Kod orqali")
def code(message):

    msg = bot.send_message(
        message.chat.id,
        "Anime kodini yuboring"
    )

    bot.register_next_step_handler(msg, send_video)


def send_video(message):

    code = message.text

    cursor.execute(
        "SELECT file_id FROM anime WHERE code=?",
        (code,)
    )

    result = cursor.fetchone()

    if result:

        bot.send_video(
            message.chat.id,
            result[0]
        )

    else:
        bot.send_message(message.chat.id,"Anime topilmadi")


@bot.message_handler(content_types=['video'])
def add_video(message):

    if message.from_user.id != ADMIN_ID:
        return

    file_id = message.video.file_id

    msg = bot.send_message(message.chat.id,"Anime nomi:")
    bot.register_next_step_handler(msg,get_name,file_id)


def get_name(message,file_id):

    name = message.text

    msg = bot.send_message(message.chat.id,"Anime kodi:")
    bot.register_next_step_handler(msg,save_anime,name,file_id)


def save_anime(message,name,file_id):

    code = message.text

    cursor.execute(
        "INSERT INTO anime(name,code,file_id) VALUES(?,?,?)",
        (name,code,file_id)
    )

    conn.commit()

    bot.send_message(message.chat.id,"Anime qo'shildi")

    bot.send_video(
        "@"+CHANNEL,
        file_id,
        caption=f"🎬 {name}\n📌 Kod: {code}"
    )


print("PRO Anime Bot ishga tushdi")
bot.infinity_polling()