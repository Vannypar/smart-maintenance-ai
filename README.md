import streamlit as st
import pandas as pd
import numpy as np
import time
from telegram import Bot
from datetime import datetime

# === Configura Telegram Bot ===
TELEGRAM_TOKEN = 'INSERISCI_IL_TUO_TOKEN'
CHAT_ID = 'INSERISCI_ID_CHAT'
bot = Bot(token=TELEGRAM_TOKEN)

def send_telegram_alert(message):
    try:
        bot.send_message(chat_id=CHAT_ID, text=message)
        st.session_state['alerts_sent'].append(f"{datetime.now().strftime('%H:%M:%S')} - {message}")
    except Exception as e:
        st.error(f"Errore Telegram: {e}")

def generate_sensor_data():
    temp = np.random.normal(loc=70, scale=5)
    vib = np.random.normal(loc=0.02, scale=0.01)
    return temp, vib

# === Sidebar per configurare soglie ===
st.sidebar.header("⚙️ Configurazione")
TEMP_THRESHOLD = st.sidebar.slider("Soglia Temperatura (°C)", 60, 100, 80)
VIB_THRESHOLD = st.sidebar.slider("Soglia Vibrazione (g)", 0.01, 0.1, 0.04, step=0.005)
REFRESH_TIME = st.sidebar.slider("Frequenza aggiornamento (s)", 1, 10, 3)

if st.sidebar.button("Invia alert test"):
    send_telegram_alert("🔧 Alert test inviato dal sistema.")

# === Inizializzazione ===
if 'df' not in st.session_state:
    st.session_state.df = pd.DataFrame(columns=['timestamp', 'temperature', 'vibration'])
if 'alerts_sent' not in st.session_state:
    st.session_state.alerts_sent = []

# === Titolo ===
st.title("🧠 Smart Maintenance AI - Demo Tecnica")
placeholder = st.empty()

# === Dashboard Loop ===
for _ in range(50):
    temp, vib = generate_sensor_data()
    now = pd.Timestamp.now()
    st.session_state.df = pd.concat([
        st.session_state.df,
        pd.DataFrame([{'timestamp': now, 'temperature': temp, 'vibration': vib}])
    ], ignore_index=True)

    with placeholder.container():
        st.subheader("📈 Dati in tempo reale")
        st.line_chart(st.session_state.df.set_index('timestamp')[['temperature']])
        st.line_chart(st.session_state.df.set_index('timestamp')[['vibration']])

        col1, col2 = st.columns(2)
        col1.metric("Temperatura attuale", f"{temp:.2f} °C",
                    delta=f"{temp - TEMP_THRESHOLD:.1f}", delta_color="inverse" if temp > TEMP_THRESHOLD else "normal")
        col2.metric("Vibrazione attuale", f"{vib:.3f} g",
                    delta=f"{vib - VIB_THRESHOLD:.3f}", delta_color="inverse" if vib > VIB_THRESHOLD else "normal")

        alerts = []
        if temp > TEMP_THRESHOLD:
            alerts.append(f"⚠️ Temperatura elevata: {temp:.2f}°C")
        if vib > VIB_THRESHOLD:
            alerts.append(f"⚠️ Vibrazione anomala: {vib:.3f}g")
        if alerts:
            for alert in alerts:
                st.warning(alert)
            send_telegram_alert("\\n".join(alerts))

    time.sleep(REFRESH_TIME)

# === Storico Alert ===
st.expander("📜 Storico alert Telegram").write(st.session_state['alerts_sent'][-10:])