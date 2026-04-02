import MetaTrader5 as mt5
import time
import random
import struct
import sys

# ------------------ CONFIG ------------------
SYMBOL = "EURUSD"
LOT = 0.1
SAFE_MODE = True  # True = no real trades, False = real trades
SLEEP_SECONDS = 2  # Seconds between loops
LOG_FILE = "bot_log.txt"

# ------------------ LOGGING ------------------
def log(msg):
    print(msg)
    try:
        with open(LOG_FILE, "a") as f:
            f.write(f"{time.strftime('%Y-%m-%d %H:%M:%S')} - {msg}\n")
    except Exception:
        pass  # ignore file write errors

# ------------------ BIT VERSION CHECK ------------------
def check_bit_version():
    bit = struct.calcsize("P") * 8
    log(f"Python is {bit}-bit")
    if mt5.version() is not None:
        log(f"MetaTrader5 version: {mt5.version()}")
    else:
        log("⚠️ MT5 not installed or cannot detect version")

# ------------------ CONNECT ------------------
def connect():
    if not mt5.initialize():
        err = mt5.last_error()
        log(f"❌ initialize() failed, error code: {err}")
        return False
    log("✅ Connected to MetaTrader 5")
    return True

# ------------------ ACCOUNT INFO ------------------
def get_account_info():
    info = mt5.account_info()
    if info is None:
        log("❌ Failed to get account info")
        return False
    log(f"💰 Balance: {info.balance}, Equity: {info.equity}, Leverage: {info.leverage}, Login: {info.login}")
    return True

# ------------------ GET DATA ------------------
def get_data():
    tick = mt5.symbol_info_tick(SYMBOL)
    if tick is None:
        log(f"❌ No tick data for {SYMBOL}")
        return None
    return tick

# ------------------ STRATEGY ------------------
def strategy(data):
    if data is None:
        return None
    # Simple random strategy
    return random.choice(["buy", "sell", None])

# ------------------ CHECK OPEN POSITION ------------------
def has_open_position():
    try:
        positions = mt5.positions_get(symbol=SYMBOL)
        return positions is not None and len(positions) > 0
    except Exception as e:
        log(f"❌ Error checking positions: {e}")
        return False

# ------------------ PLACE TRADE ------------------
def place_trade(signal):
    if signal is None:
        return
    if has_open_position():
        log("ℹ️ Position already open, skipping trade")
        return

    log(f"📊 Signal: {signal}")

    if SAFE_MODE:
        log("🟡 SAFE MODE: No real trade executed")
        return

    tick = mt5.symbol_info_tick(SYMBOL)
    if tick is None:
        log(f"❌ Cannot trade, no tick data for {SYMBOL}")
        return

    price = tick.ask if signal == "buy" else tick.bid
    sl = 20 * 0.0001
    tp = 40 * 0.0001

    request = {
        "action": mt5.TRADE_ACTION_DEAL,
        "symbol": SYMBOL,
        "volume": LOT,
        "type": mt5.ORDER_TYPE_BUY if signal == "buy" else mt5.ORDER_TYPE_SELL,
        "price": price,
        "sl": price - sl if signal == "buy" else price + sl,
        "tp": price + tp if signal == "buy" else price - tp,
        "deviation": 10,
        "magic": 123456,
        "comment": "Python bot",
        "type_time": mt5.ORDER_TIME_GTC,
        "type_filling": mt5.ORDER_FILLING_IOC,
    }

    try:
        result = mt5.order_send(request)
        if result.retcode != mt5.TRADE_RETCODE_DONE:
            log(f"❌ Trade failed, retcode: {result.retcode}, result: {result}")
        else:
            log("✅ Trade executed successfully")
    except Exception as e:
        log(f"❌ Exception sending order: {e}")

# ------------------ MAIN LOOP ------------------
def main():
    check_bit_version()
    
    if not connect():
        log("⛔ Cannot continue without MT5 connection")
        return

    if not get_account_info():
        log("⛔ Cannot continue without account info")
        return

    log(f"🟢 Starting bot on {SYMBOL} (SAFE_MODE={SAFE_MODE})")

    try:
        while True:
            data = get_data()
            if data:
                log(f"📈 Bid: {data.bid}, Ask: {data.ask}")

            signal = strategy(data)
            place_trade(signal)

            time.sleep(SLEEP_SECONDS)

    except KeyboardInterrupt:
        log("⛔ Bot stopped by user (KeyboardInterrupt)")
    except Exception as e:
        log(f"❌ Unexpected error: {e}")
    finally:
        mt5.shutdown()
        log("🔌 Disconnected from MetaTrader 5")
        log("🛑 Bot ended")

# ------------------ START ------------------
if __name__ == "__main__":
    main()
