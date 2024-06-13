import time
import numpy as np
import pandas as pd
import talib
from iqoptionapi.stable_api import IQ_Option

# إعدادات الدخول
email = "your_email@example.com"
password = "your_password"

# الاتصال بـ IQ Option
api = IQ_Option(email, password)
api.connect()

# التحقق من الاتصال
if api.check_connect():
    print("Connected successfully!")
else:
    print("Connection failed!")
    exit()

# إعدادات التداول
asset = "EURUSD"
amount = 1  # حجم الصفقة
duration = 1  # مدة الصفقة بالدقائق
rsi_period = 14
sma_period = 50

def get_historical_data(asset, interval, num_candles):
    end_from_time = time.time()
    candles = api.get_candles(asset, interval, num_candles, end_from_time)
    df = pd.DataFrame(candles)
    df['timestamp'] = pd.to_datetime(df['from'], unit='s')
    df.set_index('timestamp', inplace=True)
    return df

def generate_signals(df):
    df['rsi'] = talib.RSI(df['close'], timeperiod=rsi_period)
    df['sma'] = talib.SMA(df['close'], timeperiod=sma_period)
    df['buy_signal'] = np.where((df['rsi'] < 30) & (df['close'] > df['sma']), 1, 0)
    df['sell_signal'] = np.where((df['rsi'] > 70) & (df['close'] < df['sma']), 1, 0)
    return df

def place_trade(action):
    direction = "call" if action == 1 else "put"
    check, id = api.buy(amount, asset, direction, duration)
    if check:
        print(f"Trade placed successfully: {direction}")
    else:
        print("Failed to place trade")

while True:
    df = get_historical_data(asset, 60, 100)  # 60 seconds interval, 100 candles
    df = generate_signals(df)

    last_row = df.iloc[-1]
    if last_row['buy_signal'] == 1:
        place_trade(1)
    elif last_row['sell_signal'] == 1:
        place_trade(0)

    time.sleep(60)  # الانتظار لمدة دقيقة قبل التحقق من الإشارات مرة أخرى
