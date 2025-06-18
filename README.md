import ccxt
import pandas as pd
import ta
import smtplib
import schedule
import time
import datetime
import requests
from email.mime.text import MIMEText
from flask import Flask
from threading import Thread

# Thiết lập email người gửi
EMAIL_SENDER = 'nguyenvantien02011995@gmail.com'
EMAIL_PASSWORD = 'uhpfbxiolgtkmffu'
EMAIL_RECEIVER = 'nguyenvantien02011990@gmail.com'
EMAIL_RECEIVER = 'tdhlan@gmail.com'

# Gửi email cảnh báo
def send_email(subject, content):
    msg = MIMEText(content)
    msg['Subject'] = subject
    msg['From'] = EMAIL_SENDER
    msg['To'] = EMAIL_RECEIVER

    try:
        with smtplib.SMTP_SSL('smtp.gmail.com', 465) as smtp:
            smtp.login(EMAIL_SENDER, EMAIL_PASSWORD)
            smtp.send_message(msg)
        print(f"✅ Đã gửi email: {subject}")
    except Exception as e:
        print(f"❌ Lỗi gửi email: {e}")

# Ghi log vào file
def log_alert(content):
    with open("alerts_log.txt", "a") as f:
        timestamp = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        f.write(f"[{timestamp}] {content}\n")

# Lấy danh sách top 50 coin theo vốn hóa từ CoinGecko
def get_top50_symbols():
    url = "https://api.coingecko.com/api/v3/coins/markets"
    params = {
        'vs_currency': 'usd',
        'order': 'market_cap_desc',
    'per_page': 100,
        'page': 1,
        'sparkline': False
    }
    try:
        response = requests.get(url, params=params)
        data = response.json()
        symbols = [coin['symbol'].upper() for coin in data]
        return symbols
    except:
        return []

# Phân tích tín hiệu mua
def get_long_signal(symbol, exchange):
    try:
        ohlcv = exchange.fetch_ohlcv(symbol, timeframe='15m', limit=150)
        df = pd.DataFrame(ohlcv, columns=['time', 'open', 'high', 'low', 'close', 'volume'])

        df['ma7'] = df['close'].rolling(window=7).mean()
        df['ma50'] = df['close'].rolling(window=50).mean()
        df['ma99'] = df['close'].rolling(window=99).mean()
        df['rsi'] = ta.momentum.RSIIndicator(df['close'], window=14).rsi()
        df['macd'] = ta.trend.MACD(df['close']).macd()
        df['macd_signal'] = ta.trend.MACD(df['close']).macd_signal()
        df['vol_avg20'] = df['volume'].rolling(window=20).mean()

        last = df.iloc[-1]
        volume_boost = last['volume'] > 1.5 * last['vol_avg20']
        macd_cross = last['macd'] > last['macd_signal'] and last['macd'] > 0
        rsi_ok = 55 < last['rsi'] < 64
        ma_signal = last['ma7'] > last['ma50'] > last['ma99']
        price_above_ma7 = last['close'] > last['ma7']

        if volume_boost and macd_cross and rsi_ok and ma_signal and price_above_ma7:
            return f"{symbol} | Price: {last['close']:.4f} | RSI: {last['rsi']:.2f}, MACD: {last['macd']:.4f}, Volume: {last['volume']:.0f}, MA7: {last['ma7']:.2f}, MA50: {last['ma50']:.2f}"
        else:
            return False
    except:
        return False

# Phân tích tín hiệu bán
def get_short_signal(symbol, exchange):
    try:
        ohlcv = exchange.fetch_ohlcv(symbol, timeframe='15m', limit=150)
        df = pd.DataFrame(ohlcv, columns=['time', 'open', 'high', 'low', 'close', 'volume'])

        df['ma7'] = df['close'].rolling(window=7).mean()
        df['ma50'] = df['close'].rolling(window=50).mean()
        df['ma99'] = df['close'].rolling(window=99).mean()
        df['rsi'] = ta.momentum.RSIIndicator(df['close'], window=14).rsi()
        df['macd'] = ta.trend.MACD(df['close']).macd()
        df['macd_signal'] = ta.trend.MACD(df['close']).macd_signal()
        df['vol_avg20'] = df['volume'].rolling(window=20).mean()

        last = df.iloc[-1]
        volume_boost = last['volume'] > 1.5 * last['vol_avg20']
        macd_cross_down = last['macd'] < last['macd_signal'] and last['macd'] < 0
        rsi_down = last['rsi'] < 44
        ma_bearish = last['ma7'] < last['ma50'] < last['ma99']
        price_below_ma7 = last['close'] < last['ma7']

        if volume_boost and macd_cross_down and rsi_down and ma_bearish and price_below_ma7:
            return f"{symbol} | Price: {last['close']:.4f} | RSI: {last['rsi']:.2f}, MACD: {last['macd']:.4f}, Volume: {last['volume']:.0f}, MA7: {last['ma7']:.2f}, MA50: {last['ma50']:.2f}"
        else:
            return False
    except:
        return False

# Web server giữ cho Replit luôn online
app = Flask('')

@app.route('/')
def home():
    return "Bot is running!"

def run_web():
    app.run(host='0.0.0.0', port=8080)

def keep_alive():
    t = Thread(target=run_web)
    t.start()

# Hàm chính để lọc và gửi cảnh báo
def scan_market():
    print("⏳ Đang quét thị trường...")
    exchange = ccxt.mexc()
    markets = exchange.load_markets()
    top50_symbols = get_top50_symbols()
    usdt_pairs = [symbol for symbol in markets if symbol.endswith('/USDT') and symbol.split('/')[0] in top50_symbols]

    long_signals = []
    short_signals = []

    for symbol in usdt_pairs:
        try:
            print(f"🔍 Đang xét: {symbol}")
            long_signal = get_long_signal(symbol, exchange)
            short_signal = get_short_signal(symbol, exchange)
            if long_signal:
                long_signals.append(long_signal)
            elif short_signal:
                short_signals.append(short_signal)
            time.sleep(2)
        except Exception as e:
            print(f"Lỗi khi xử lý {symbol}: {e}")

    if long_signals:
        message = "📈 Coin có tín hiệu MUA vào:\n" + "\n".join(long_signals)
        print(message)
        send_email("🚀 Tín hiệu MUA coin", message)
        log_alert(message)

    if short_signals:
        message = "📉 Coin có tín hiệu BÁN ra:\n" + "\n".join(short_signals)
        print(message)
        send_email("🔻 Tín hiệu BÁN coin", message)
        log_alert(message)

    if not long_signals and not short_signals:
        print("Không có coin đủ điều kiện.")

# Khởi động Flask giữ Replit online
keep_alive()

# Thiết lập chạy mỗi 5 phút
schedule.every(5).minutes.do(scan_market)

print("🚀 Bot đang chạy 24/7...")
while True:
    schedule.run_pending()
    time.sleep(1)
# python-bot-hue
