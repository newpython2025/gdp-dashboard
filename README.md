import os
from binance.client import Client
import pandas as pd
import sqlite3
import pandas_ta as ta
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
import matplotlib.pyplot as plt
import backtrader as bt
import streamlit as st

# Configuración de la API de Binance (¡NO incluir claves directamente en el código!)
# Mejor usar variables de entorno o un archivo de configuración seguro.
api_key = os.environ.get("BINANCE_API_KEY")  # Ejemplo: export BINANCE_API_KEY="tu_clave"
api_secret = os.environ.get("BINANCE_API_SECRET") # Ejemplo: export BINANCE_API_SECRET="tu_secreto"

if not api_key or not api_secret:
    raise ValueError("Debes configurar las variables de entorno BINANCE_API_KEY y BINANCE_API_SECRET.")

client = Client(api_key, api_secret)

# Obtener datos en tiempo real (ejemplo: BTC/USDT)
def obtener_datos_en_tiempo_real(symbol, interval, limit=100): # Añadido límite
    try:
        klines = client.get_klines(symbol=symbol, interval=interval, limit=limit)
    except Exception as e:
        st.error(f"Error al obtener datos de Binance: {e}") # Mostrar error en Streamlit
        return None

    data = pd.DataFrame(klines, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume', 'close_time', 'quote_asset_volume', 'number_of_trades', 'taker_buy_base_asset_volume', 'taker_buy_quote_asset_volume', 'ignore'])
    data['timestamp'] = pd.to_datetime(data['timestamp'], unit='ms')
    data[['open', 'high', 'low', 'close', 'volume']] = data[['open', 'high', 'low', 'close', 'volume']].astype(float)
    return data

# Conectar a la base de datos
conn = sqlite3.connect('trading_data.db')
cursor = conn.cursor()

# Crear tabla si no existe (mejor usar parámetros en execute)
cursor.execute('''
CREATE TABLE IF NOT EXISTS market_data (
    timestamp DATETIME PRIMARY KEY,
    open REAL,  -- Usar REAL para floats
    high REAL,
    low REAL,
    close REAL,
    volume REAL
)
''')
conn.commit()

# Función para guardar datos en la base de datos (manejar errores)
def guardar_datos_en_db(data):
    try:
        data.to_sql('market_data', conn, if_exists='append', index=False)
    except Exception as e:
        st.error(f"Error al guardar datos en la base de datos: {e}")

# Streamlit
st.title('Herramienta de Day Trading en Criptomonedas')

symbol = st.text_input("Símbolo (ej: BTCUSDT)", "BTCUSDT") # Input para el símbolo
interval = st.selectbox("Intervalo", ["1m", "5m", "1h", "1d"]) # Selectbox para el intervalo
limit = st.number_input("Límite de velas", min_value=10, value=100) # Input para el límite

if st.button("Obtener y analizar datos"): # Botón para iniciar el proceso
    datos = obtener_datos_en_tiempo_real(symbol, interval, limit)
    if datos is None: # Si hubo un error al obtener los datos, no continuar
        st.stop()

    guardar_datos_en_db(datos)

    # Calcular indicadores (manejar posibles errores)
    try:
        datos['RSI'] = ta.rsi(datos['close'], length=14)
        datos.ta.macd(append=True)
    except Exception as e:
        st.error(f"Error al calcular indicadores: {e}")
        st.stop()

    # Señales de trading
    datos['Signal'] = 0
    datos.loc[datos['RSI'] < 30, 'Signal'] = 1
    datos.loc[datos['RSI'] > 70, 'Signal'] = -1

    # ... (resto del código igual, pero con st.write y st.pyplot)

    st.write(datos) # Mostrar datos en Streamlit

    # ... (Preparar datos para ML, entrenar modelo, etc.)

    # Visualización en Streamlit
    fig, ax = plt.subplots(figsize=(12, 6)) # Crear la figura y los ejes
    ax.plot(datos['close'], label='Precio de cierre')
    ax.plot(datos[datos['Signal'] == 1].index, datos['close'][datos['Signal'] == 1], '^', markersize=10, color='g', lw=0, label='Compra')
    ax.plot(datos[datos['Signal'] == -1].index, datos['close'][datos['Signal'] == -1], 'v', markersize=10, color='r', lw=0, label='Venta')
    ax.legend()
    st.pyplot(fig) # Mostrar el gráfico en Streamlit

    # ... (Backtesting)

    # ... (Mostrar resultados del backtesting en Streamlit)

conn.close() # Cerrar la conexión a la base de datos al final
