import os
import ccxt
import time
import requests

# Telegram настройки из переменных окружения
BOT_TOKEN = os.environ.get("BOT_TOKEN")
CHAT_ID = os.environ.get("CHAT_ID")

# Отправка сообщения в Telegram
def send_telegram_message(text):
    url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage"
    data = {"chat_id": CHAT_ID, "text": text}
    try:
        response = requests.post(url, data=data)
        if response.status_code != 200:
            print("Telegram Error:", response.text)
    except Exception as e:
        print("Ошибка отправки в Telegram:", e)

# Подсчёт подряд идущих свечей
def count_consecutive_candles(candles, bullish=True):
    count = 0
    for candle in reversed(candles):
        open_, close = candle[1], candle[4]
        if bullish and close > open_:
            count += 1
        elif not bullish and close < open_:
            count += 1
        else:
            break
    return count

# Проверка одной биржи
def check_exchange(exchange_id):
    print(f"\n🔍 Проверка {exchange_id}...")
    try:
        exchange_class = getattr(ccxt, exchange_id)
        exchange = exchange_class({"options": {"defaultType": "future"}})
        markets = exchange.load_markets()
        symbols = [
            symbol for symbol in markets
            if "/USDT" in symbol and markets[symbol].get("future", False)
        ]
    except Exception as e:
        print(f"Не удалось загрузить {exchange_id}: {e}")
        return

    for symbol in symbols:
        try:
            candles = exchange.fetch_ohlcv(symbol, timeframe="15m")
            if not candles or len(candles) < 12:
                continue

            up_count = count_consecutive_candles(candles, bullish=True)
            down_count = count_consecutive_candles(candles, bullish=False)

            if up_count in [6, 9, 12]:
                message = f"🟢 {symbol} на {exchange_id}: {up_count} зелёных свечей подряд!"
                print(message)
                send_telegram_message(message)
            elif down_count in [6, 9, 12]:
                message = f"🔴 {symbol} на {exchange_id}: {down_count} красных свечей подряд!"
                print(message)
                send_telegram_message(message)

        except Exception as e:
            print(f"Ошибка при обработке {symbol}: {e}")

# Главный бесконечный цикл
while True:
    for exchange_name in ["bybit", "mexc"]:  # Binance убран!
        check_exchange(exchange_name)
        time.sleep(1)
    print("Ожидание 60 секунд перед следующим циклом...\n")
    time.sleep(60)
    
