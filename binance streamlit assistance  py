#!/usr/bin/env python3
"""
GILANI'S — AI Binance Live Trading Terminal
===========================================
Real-time Mainnet trading dashboard with automated SL/TP.
"""

from __future__ import annotations
from datetime import datetime
import pandas as pd
import streamlit as st

try:
    import ccxt
except ImportError:
    ccxt = None

try:
    from google import genai
except ImportError:
    genai = None

# ----------------------------- Page setup -----------------------------------
st.set_page_config(
    page_title="GILANI'S | AI Live Trading",
    page_icon="📈",
    layout="wide",
    initial_sidebar_state="expanded",
)

st.markdown(
    """
    <style>
      .stApp { background: #0b0e14; color: #e2e8f0; }
      div[data-testid="stMetricValue"] { font-family: ui-monospace, monospace; }
      .live-dot { color: #22c55e; animation: blink 2s infinite; }
      @keyframes blink { 0% { opacity: 1; } 50% { opacity: 0.3; } 100% { opacity: 1; } }
      .stButton>button { width: 100%; }
    </style>
    """,
    unsafe_allow_html=True,
)

st.title("📈 GILANI'S — Live Real-Money Trading")

# Initialize session state
if "messages" not in st.session_state:
    st.session_state.messages = [{"role": "assistant", "content": "Connected. Waiting for instructions."}]
if "last_order_result" not in st.session_state:
    st.session_state.last_order_result = None
if "api_key" not in st.session_state: st.session_state.api_key = ""
if "api_secret" not in st.session_state: st.session_state.api_secret = ""
if "gemini_key" not in st.session_state: st.session_state.gemini_key = ""
if "testnet" not in st.session_state: st.session_state.testnet = False # Default to Mainnet for real money

# ----------------------------- Sidebar --------------------------------------
with st.sidebar:
    st.header("🔐 Binance Mainnet Connection")
    
    # Security acknowledgment for real money
    st.session_state.testnet = st.toggle("Use Testnet (Play Money)", value=st.session_state.testnet)
    
    if not st.session_state.testnet:
        st.error("⚠️ **REAL MONEY MODE ACTIVE.**")
        live_ack = st.checkbox("I understand I am trading real funds and accept the risk.")
    else:
        live_ack = True # Auto-acknowledged in testnet

    input_api_key = st.text_input("Binance API key", value=st.session_state.api_key, type="password")
    input_api_secret = st.text_input("Binance API secret", value=st.session_state.api_secret, type="password")
    input_gemini_key = st.text_input("Google Gemini API key", value=st.session_state.gemini_key, type="password")
    
    col1, col2 = st.columns(2)
    with col1:
        if st.button("🔗 Connect", use_container_width=True, type="primary"):
            if not st.session_state.testnet and not live_ack:
                st.error("You must check the risk acknowledgement box to connect to Mainnet.")
            else:
                st.session_state.api_key = input_api_key
                st.session_state.api_secret = input_api_secret
                st.session_state.gemini_key = input_gemini_key
                st.rerun()
    with col2:
        if st.button("🚪 Disconnect", use_container_width=True):
            for key in ["api_key", "api_secret", "gemini_key"]:
                st.session_state[key] = ""
            st.rerun()

    if st.session_state.api_key and st.session_state.api_secret:
        st.success("✅ Keys saved. Live connection active.")
        st.divider()
        st.subheader("⚙️ Live Polling Rate")
        refresh_rate = st.slider("Refresh interval (seconds)", min_value=3, max_value=30, value=5, step=1)
    else:
        refresh_rate = 5

    symbol = st.selectbox(
        "Trading pair",
        ["BTC/USDT:USDT", "ETH/USDT:USDT", "SOL/USDT:USDT", "BNB/USDT:USDT", "XRP/USDT:USDT", "DOGE/USDT:USDT"],
        index=0,
    )

# ----------------------------- Client Setup ---------------------------------
def make_exchange(key: str, secret: str, sandbox: bool):
    if ccxt is None: raise RuntimeError("CCXT not installed.")
    if not key or not secret: return None
    
    # Initialize Binance Futures client
    ex = ccxt.binance({
        "apiKey": key, 
        "secret": secret, 
        "enableRateLimit": True, 
        "timeout": 15000,
        "options": {
            "defaultType": "future",
            "adjustForTimeDifference": True,
            "warnOnFetchOpenOrdersWithoutSymbol": False
        },
    })
    # If sandbox is True, it routes to Testnet. If False, it routes to real Mainnet.
    if sandbox:
        ex.set_sandbox_mode(True)
        
    return ex

@st.cache_resource(show_spinner=False, ttl=60)
def _cached_exchange(key: str, secret: str, sandbox: bool):
    return make_exchange(key, secret, sandbox)

exchange = None
if st.session_state.api_key and st.session_state.api_secret:
    try:
        exchange = _cached_exchange(st.session_state.api_key, st.session_state.api_secret, st.session_state.testnet)
    except Exception as e:
        st.error(f"Connection failed: {e}")

ai_client = None
if st.session_state.gemini_key and genai is not None:
    try:
        ai_client = genai.Client(api_key=st.session_state.gemini_key)
    except Exception as e:
        st.sidebar.error(f"Gemini setup failed: {e}")

if exchange is None:
    st.info("👈 **Please log in.** Enter your real Binance API credentials in the sidebar to connect to the live network.")
    st.stop()

# ----------------------------- LIVE DASHBOARD (Auto-Refreshing) ------------------------------
@st.fragment(run_every=refresh_rate)
def live_dashboard():
    try:
        # 1. Fetch live market ticker
        ticker = exchange.fetch_ticker(symbol)
        last_price = float(ticker.get("last") or ticker.get("close") or 0.0)
        
        # 2. Fetch live account balance
        balance = exchange.fetch_balance()
        usdt = balance.get("USDT") or {}
        balance_total = float(usdt.get("total") or 0.0)
        balance_free = float(usdt.get("free") or 0.0)

        # 3. Fetch live open positions
        raw_positions = exchange.fetch_positions([symbol])
        positions = []
        total_pnl = 0.0
        for p in raw_positions:
            contracts = float(p.get("contracts") or 0.0)
            if abs(contracts) > 0:
                pnl = float(p.get("unrealizedPnl") or 0.0)
                total_pnl += pnl
                positions.append({
                    "Symbol": p.get("symbol") or symbol,
                    "Side": str(p.get("side") or "unknown").upper(),
                    "Size": abs(contracts),
                    "Entry": float(p.get("entryPrice") or 0.0),
                    "Mark": float(p.get("markPrice") or 0.0),
                    "PnL (USDT)": pnl,
                    "ROE (%)": float(p.get("percentage") or 0.0),
                })
        
        live_ok = True
        error_msg = None
        
        # Cache in session state for the Order Form to use
        st.session_state.live_price = last_price
        st.session_state.live_free = balance_free
        st.session_state.live_ok = True
        st.session_state.open_positions = positions

    except Exception as e:
        live_ok = False
        error_msg = str(e)
        st.session_state.live_ok = False

    # --- Render UI ---
    st.subheader("📡 Live Real-Time Account Data")
    env_label = "FUTURES MAINNET (REAL MONEY)" if not st.session_state.testnet else "FUTURES TESTNET"
    now = datetime.now().strftime("%H:%M:%S")
    
    if error_msg:
        st.error(f"⚠️ Network disconnected: {error_msg}. Retrying in background...")
    else:
        st.caption(f"Environment: {env_label} | Pair: {symbol} | <span class='live-dot'>●</span> Synced at {now}", unsafe_allow_html=True)
        
        m1, m2, m3, m4 = st.columns(4)
        m1.metric("Total Balance", f"{balance_total:,.2f} USDT")
        m2.metric("Available Balance", f"{balance_free:,.2f} USDT")
        m3.metric("Current Price", f"{last_price:,.4f} USDT")
        
        if positions:
            m4.metric("Open PnL", f"{total_pnl:+,.2f} USDT", delta=f"{total_pnl:+,.2f}")
        else:
            m4.metric("Open PnL", "0.00 USDT")

        st.subheader("Open Positions")
        if positions:
            st.dataframe(pd.DataFrame(positions), use_container_width=True, hide_index=True)
            
            # Add a quick close button for real trading
            if st.button("🚨 Close All Positions (Market)", type="secondary"):
                try:
                    for p in positions:
                        side_to_close = "sell" if p["Side"] == "LONG" else "buy"
                        exchange.create_order(symbol, "market", side_to_close, p["Size"], None, {"reduceOnly": True})
                    st.success("Close orders submitted.")
                    st.rerun(scope="fragment")
                except Exception as e:
                    st.error(f"Failed to close positions: {e}")
        else:
            st.success("No open positions.")

# Execute the live dashboard
live_dashboard()
st.divider()

# ----------------------------- Order Controls ------------------------------
left, right = st.columns([1, 1], gap="large")

with left:
    st.subheader("⚡ Live Order Execution")
    st.caption("Executes real market orders on Binance.")
    
    current_price = st.session_state.get("live_price", 0.0)
    current_free = st.session_state.get("live_free", 0.0)
    is_live = st.session_state.get("live_ok", False)

    leverage = st.slider("Leverage", 1, 20, 5) # Increased max leverage for real trading
    margin_usdt = st.number_input("Margin (USDT)", 5.0, 10000.0, 20.0, 5.0)
    sl_pct = st.number_input("Stop-Loss (%)", 0.2, 10.0, 1.0, 0.1)
    tp_pct = st.number_input("Take-Profit (%)", 0.2, 50.0, 2.0, 0.1)

    notional = margin_usdt * leverage
    quantity_preview = (notional / current_price) if current_price else None
    
    c1, c2 = st.columns(2)
    c1.metric("Notional Size", f"{notional:,.2f} USDT")
    c2.metric("Est. Quantity", "—" if quantity_preview is None else f"{quantity_preview:.6f}")

    confirm_order = st.checkbox(f"I confirm I want to execute a real trade for {symbol}.")
    if not st.session_state.testnet:
        confirm_order = confirm_order and live_ack

    def execute_order(side: str):
        if not is_live or not current_price:
            st.error("Order blocked: Live network data unavailable.")
            return
        if not confirm_order:
            st.error("Order blocked: Please check the confirmation box.")
            return
        if margin_usdt > current_free:
            st.error(f"Order blocked: Insufficient available balance. You have {current_free:.2f} USDT free.")
            return

        try:
            # 1. Get the absolute latest price right before execution
            fresh_ticker = exchange.fetch_ticker(symbol)
            fresh_price = float(fresh_ticker.get("last") or fresh_ticker.get("close") or 0.0)
            
            # 2. Calculate precise order size
            raw_amount = notional / fresh_price
            amount = float(exchange.amount_to_precision(symbol, raw_amount))
            
            # 3. Set Leverage
            exchange.set_leverage(leverage, symbol)
            
            # 4. Submit Entry Order
            order_side = "buy" if side == "BUY" else "sell"
            exit_side = "sell" if side == "BUY" else "buy"
            
            entry_order = exchange.create_order(symbol, "market", order_side, amount)
            
            # Wait a moment for the fill (simplified for Streamlit)
            import time
            time.sleep(1) 
            
            # 5. Fetch actual fill price
            fetch_order = exchange.fetch_order(entry_order['id'], symbol)
            fill_price = float(fetch_order.get("average") or fetch_order.get("price") or fresh_price)

            # 6. Calculate SL/TP Prices
            if side == "BUY":
                stop_price = fill_price * (1 - sl_pct / 100.0)
                take_price = fill_price * (1 + tp_pct / 100.0)
            else:
                stop_price = fill_price * (1 + sl_pct / 100.0)
                take_price = fill_price * (1 - tp_pct / 100.0)

            stop_price_fmt = exchange.price_to_precision(symbol, stop_price)
            take_price_fmt = exchange.price_to_precision(symbol, take_price)

            # 7. Submit Stop Loss and Take Profit
            sl_order = exchange.create_order(symbol, "STOP_MARKET", exit_side, amount, None, {"stopPrice": stop_price_fmt, "reduceOnly": True, "workingType": "MARK_PRICE"})
            tp_order = exchange.create_order(symbol, "TAKE_PROFIT_MARKET", exit_side, amount, None, {"stopPrice": take_price_fmt, "reduceOnly": True, "workingType": "MARK_PRICE"})

            st.session_state.last_order_result = {
                "side": side, 
                "amount": amount, 
                "fill_price": fill_price, 
                "sl_trigger": stop_price_fmt, 
                "tp_trigger": take_price_fmt
            }
            st.success(f"Trade Executed! Filled at {fill_price:.6f}")
            
            # Refresh the live dashboard to show the new position immediately
            st.rerun(scope="fragment")

        except Exception as exc:
            st.error(f"Trade Execution Failed: {exc}")

    b1, b2 = st.columns(2)
    with b1:
        if st.button("🟢 MARKET BUY / LONG", type="primary", use_container_width=True):
            execute_order("BUY")
    with b2:
        if st.button("🔴 MARKET SELL / SHORT", use_container_width=True):
            execute_order("SELL")

    if st.session_state.last_order_result:
        with st.expander("Last Execution Details", expanded=True):
            st.json(st.session_state.last_order_result)


# ----------------------------- Gemini Chat ----------------------------------
with right:
    st.subheader("🤖 Gemini AI Co-pilot")
    for message in st.session_state.messages:
        with st.chat_message(message["role"]):
            st.write(message["content"])

    question = st.chat_input("Ask for market analysis…")
    if question:
        st.session_state.messages.append({"role": "user", "content": question})
        if ai_client is None:
            answer = "Gemini not connected. Enter API key in sidebar."
        else:
            # Provide real-time context to Gemini
            context = f"Live Price: {st.session_state.get('live_price', 'Unknown')}\nBalance: {st.session_state.get('live_free', 'Unknown')}\nOpen Positions: {st.session_state.get('open_positions', 'None')}"
            prompt = f"You are a trading assistant. Current Market Context: {context}\nUser Question: {question}\nProvide a concise, risk-aware answer."
            try:
                response = ai_client.models.generate_content(model="gemini-2.5-flash", contents=prompt)
                answer = response.text
            except Exception as exc:
                answer = f"Gemini error: {exc}"
        st.session_state.messages.append({"role": "assistant", "content": answer})
        st.rerun()
