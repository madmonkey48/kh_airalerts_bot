import os
import requests
import time
from datetime import datetime
from flask import Flask
from threading import Thread

# ---------- Keep Alive ----------
app = Flask('')

@app.route('/')
def home():
    return "Bot is running"

def run():
    app.run(host='0.0.0.0', port=8080)

def keep_alive():
    t = Thread(target=run)
    t.start()

keep_alive()

# ---------- Переменные окружения ----------
TOKEN = os.getenv("BOT_TOKEN")
CHAT_ID = os.getenv("CHAT_ID")
API_KEY_ALERTS = os.getenv("ALERT_API_KEY")

last_status = None
daily_alerts = []
last_daily_report = datetime.now().date()
last_alert_start = None

# ---------- Получение статуса тревоги ----------
def get_alert_status():
    try:
        url = "https://api.alerts.in.ua/v1/alerts/active.json"
        headers = {"Authorization": f"Bearer {API_KEY_ALERTS}"}
        response = requests.get(url, headers=headers)
        data = response.json()

        for region in data:
            if region["regionName"] == "Харківська область":
                return region["activeAlerts"]
        return []
    except Exception as e:
        print("Ошибка при получении статуса тревоги:", e)
        return []

# ---------- Формирование текста сообщения ----------
def format_alert_message(alerts, active):
    now = datetime.now()
    now_str = now.strftime("%H:%M")

    if active:
        global last_alert_start
        last_alert_start = now
        if not alerts:
            return f"🚨 *Повітряна тривога!*\n📍 Область: Харківська\n🕒 {now_str}"

        types_text = ""
        for alert in alerts:
            t = alert.get("type")
            if t == "air_raid":
                types_text += "🚨 *Повітряна тривога*\n"
            elif t == "artillery":
                types_text += "💣 *Артилерійська загроза*\n"
            elif t == "rocket":
                types_text += "🔥 *Ракетна загроза*\n"
            elif t == "street_fighting":
                types_text += "🛡️ *Вуличні бої*\n"
            elif t == "chemical":
                types_text += "☣️ *Хімічна загроза*\n"
            elif t == "nuclear":
                types_text += "☢️ *Ядерна загроза*\n"
            else:
                types_text += f"⚠️ *Інша загроза*: {t}\n"

        return f"📍 *Харківська область*\n🕒 {now_str}\n\n{types_text}"
    else:
        duration_text = ""
        if last_alert_start:
            duration = now - last_alert_start
            minutes = int(duration.total_seconds() // 60)
            duration_text = f"⏱ Тривала: {minutes} хвилин\n"
        return f"✅ *Відбій повітряної тривоги*\n📍 Область: Харківська\n🕒 {now_str}\n{duration_text}"

# ---------- Отправка сообщений ----------
def send_message(text):
    url = f"https://api.telegram.org/bot{TOKEN}/sendMessage"
    payload = {"chat_id": CHAT_ID, "text": text, "parse_mode": "Markdown"}
    try:
        requests.post(url, data=payload)
    except Exception as e:
        print("Ошибка при отправке сообщения:", e)

# ---------- Основной цикл ----------
while True:
    try:
        alerts = get_alert_status()
        current_status = bool(alerts)

        if last_status is None:
            last_status = current_status

        if current_status != last_status:
            msg = format_alert_message(alerts, current_status)
            send_message(msg)
            if current_status:
                daily_alerts.append(datetime.now())
            last_status = current_status

        # Ежедневный отчет
        today = datetime.now().date()
        if today != last_daily_report:
            count = len(daily_alerts)
            send_message(f"📊 *Статистика повітряних тривог за день:* {count} тривог")
            daily_alerts = []
            last_daily_report = today

    except Exception as e:
        print("Ошибка в основном цикле:", e)

    time.sleep(60)
