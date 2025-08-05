# Bitget-Trading-Bot
Bot de trading crypto Bitget auto
from flask import Flask, render_template, request, redirect
import os
import requests
import time
import hmac
import hashlib
import json
from dotenv import load_dotenv

load_dotenv()

app = Flask(__name__)

# Charger les clés API
API_KEY = os.getenv("API_KEY")
API_SECRET = os.getenv("API_SECRET")
PASSPHRASE = os.getenv("PASSPHRASE")

BASE_URL = "https://api.bitget.com"

HISTORY_FILE = "order_history.json"

# Signature
def generate_signature(timestamp, method, request_path, body=""):
    message = f"{timestamp}{method}{request_path}{body}"
    signature = hmac.new(API_SECRET.encode(), message.encode(), hashlib.sha256).hexdigest()
    return signature

# Headers
def get_headers(method, path, body=""):
    timestamp = str(int(time.time() * 1000))
    signature = generate_signature(timestamp, method, path, body)
    return {
        "ACCESS-KEY": API_KEY,
        "ACCESS-SIGN": signature,
        "ACCESS-TIMESTAMP": timestamp,
        "ACCESS-PASSPHRASE": PASSPHRASE,
        "Content-Type": "application/json"
    }

# Enregistrer l'ordre
def save_order(order_data):
    try:
        if not os.path.exists(HISTORY_FILE):
            with open(HISTORY_FILE, "w") as f:
                json.dump([], f)
        with open(HISTORY_FILE, "r+") as f:
            data = json.load(f)
            data.append(order_data)
            f.seek(0)
            json.dump(data, f, indent=4)
    except Exception as e:
        print(f"Erreur en sauvegardant l'ordre: {e}")

# Créer un ordre
def create_order(symbol, price, size, side):
    path = "/api/v1/mix/order/placeOrder"
    url = BASE_URL + path

    body = {
        "symbol": symbol,
        "marginCoin": "USDT",
        "orderType": "limit",
        "side": side,
        "size": str(size),
        "price": str(price),
        "timeInForceValue": "normal"
    }

    headers = get_headers("POST", path, json.dumps(body))
    response = requests.post(url, headers=headers, json=body)

    try:
        result = response.json()
        if result.get("code") == "00000":
            save_order({
                "symbol": symbol,
                "price": price,
                "size": size,
                "side": side,
                "timestamp": time.strftime("%Y-%m-%d %H:%M:%S")
            })
            return "Ordre créé avec succès"
        else:
            return f"Erreur: {result.get('msg')}"
    except Exception as e:
        return f"Erreur inattendue : {str(e)}"

@app.route("/", methods=["GET", "POST"])
def index():
    message = ""
    if request.method == "POST":
        symbol = request.form.get("symbol")
        price = request.form.get("price")
        size = request.form.get("size")
        side = request.form.get("side")  # facultatif

        if not side:
            side = "buy"

        message = create_order(symbol, price, size, side)

    return render_template("index.html", message=message)

@app.route("/history")
def history():
    try:
        with open(HISTORY_FILE, "r") as f:
            data = json.load(f)
    except:
        data = []
    return render_template("history.html", history=data)

if __name__ == "__main__":
    app.run(debug=True)

<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <title>Bot de Trading Bitget</title>
</head>
<body>
  <h1>Placer un Ordre Bitget</h1>
  <form method="POST">
    <label>Symbole (ex: CYCUSDT):</label><br>
    <input name="symbol" required><br><br>

    <label>Prix:</label><br>
    <input name="price" required><br><br>

    <label>Taille (montant USDT):</label><br>
    <input name="size" required><br><br>

    <label>Type (buy ou sell, facultatif):</label><br>
    <input name="side" placeholder="buy par défaut"><br><br>

    <button type="submit">Envoyer l'ordre</button>
  </form>

  <p style="color:green;">{{ message }}</p>
  <a href="/history">Voir l'historique</a>
</body>
</html>

<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <title>Historique des Ordres</title>
</head>
<body>
  <h2>Historique des Ordres</h2>
  <ul>
    {% for order in history %}
      <li>
        {{ order.timestamp }} | {{ order.side.upper() }} {{ order.size }} {{ order.symbol }} à {{ order.price }} USDT
      </li>
    {% endfor %}
  </ul>
  <a href="/">← Retour</a>
</body>
</html>

.env
API_KEY=bg_5877859ff6c196345ebd46d321a77C06
API_SECRET=bcffa24f0ad08107725d583d47e6462c31a7591be10b5b9c335db88099125fad
PASSPHRASE=bitget2025

