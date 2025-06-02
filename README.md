# --- STEP 1: Install required libraries ---
!pip install oandapyV20 pandas

# --- STEP 2: Import libraries ---
import pandas as pd
import oandapyV20
import oandapyV20.endpoints.instruments as instruments
from datetime import datetime, timedelta
import pytz

# --- STEP 3: Set your OANDA demo credentials here ---
OANDA_API_KEY = "YOUR_OANDA_API_KEY"
OANDA_ACCOUNT_ID = "YOUR_OANDA_ACCOUNT_ID"
OANDA_URL = "https://api-fxpractice.oanda.com"

client = oandapyV20.API(access_token=OANDA_API_KEY)

# --- STEP 4: Define function to fetch candlesticks ---
def fetch_candles(symbol="EUR_USD", count=100, granularity="M15"):
    params = {
        "count": count,
        "granularity": granularity,
        "price": "M"
    }
    r = instruments.InstrumentsCandles(instrument=symbol, params=params)
    client.request(r)
    candles = r.response["candles"]
    df = pd.DataFrame([{
        "time": c["time"],
        "open": float(c["mid"]["o"]),
        "high": float(c["mid"]["h"]),
        "low": float(c["mid"]["l"]),
        "close": float(c["mid"]["c"]),
    } for c in candles if c["complete"]])
    df["time"] = pd.to_datetime(df["time"])
    return df

# --- STEP 5: Define engulfing pattern detection ---
def detect_engulfing(df):
    signals = []
    for i in range(1, len(df)):
        prev = df.iloc[i-1]
        curr = df.iloc[i]

        # Bullish engulfing
        if prev["close"] < prev["open"] and curr["close"] > curr["open"]:
            if curr["close"] > prev["open"] and curr["open"] < prev["close"]:
                signals.append((curr["time"], "BUY"))

        # Bearish engulfing
        elif prev["close"] > prev["open"] and curr["close"] < curr["open"]:
            if curr["close"] < prev["open"] and curr["open"] > prev["close"]:
                signals.append((curr["time"], "SELL"))

    return signals

# --- STEP 6: Run the bot ---
df = fetch_candles(symbol="EUR_USD", count=100)
signals = detect_engulfing(df)

print("📢 Detected Signals:")
if signals:
    for s in signals[-5:]:  # Show last 5 signals
        print(f"{s[0]}: {s[1]}")
else:
    print("No signals found in the last 100 candles.")
