import streamlit as st
import pandas as pd
import os
import random
from io import BytesIO
import uuid
import re
import urllib.request
import urllib.parse
import json
import base64
import hashlib
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email import encoders 
from datetime import datetime

# =====================================================================
# 1. STREAMLIT APP ENGINE FRONT END WITH BRAND LOGO INTEGRATION
# =====================================================================

st.set_page_config(page_title="Risco Association", page_icon="🧢", layout="wide")

# Custom CSS Styles matching the sleek dark black and white aesthetic
st.markdown("""
    <style>
    .stApp { background-color: #000000; color: #FFFFFF; }
    h1, h2, h3, p, span, label, li { color: #FFFFFF !important; font-family: 'Impact', sans-serif !important; }
    .stButton>button { background-color: #FFFFFF !important; color: #000000 !important; border: 2px solid #FFFFFF !important; font-weight: bold !important; width: 100%; }
    .stButton>button:hover { background-color: #000000 !important; color: #FFFFFF !important; border: 2px solid #FFFFFF !important; }
    div[data-testid="stForm"] { border: 2px solid #FFFFFF !important; background-color: #111111 !important; padding: 20px; border-radius: 5px; }
    .product-card { border: 2px solid #FFFFFF; padding: 25px; border-radius: 5px; background-color: #111111; margin-bottom: 20px; }
    .tracking-card { border: 2px solid #FFFFFF; padding: 20px; border-radius: 5px; background-color: #111111; margin-top: 15px; }
    .brand-logo-container { display: flex; justify-content: center; align-items: center; padding: 20px 0; }
    .brand-logo { max-width: 160px; height: auto; border-radius: 50%; border: 3px solid #FFFFFF; box-shadow: 0px 4px 15px rgba(255, 255, 255, 0.2); }
    .metric-card { border: 1px solid #FFFFFF; border-radius: 5px; padding: 15px; background-color: #111111; text-align: center; }
    .low-stock-alert { color: #FF0000 !important; font-weight: bold; animation: blinker 1.5s linear infinite; }
    @keyframes blinker { 50% { opacity: 0; } }
    </style>
    """, unsafe_allow_html=True)

INVENTORY_FILE = "inventory.csv"
DB_FILE = "orders.csv"

# Production API Security Parameter Arrays - Securely Linked to Your Credentials
BULKSMS_TOKEN_ID = os.environ.get("BULKSMS_TOKEN_ID", "YOUR_BULKSMS_TOKEN_ID")
BULKSMS_TOKEN_SECRET = os.environ.get("BULKSMS_TOKEN_SECRET", "YOUR_BULKSMS_TOKEN_SECRET")

# SMTP Email Configuration Environment Keys
SMTP_SERVER = os.environ.get("SMTP_SERVER", "://gmail.com")
SMTP_PORT = int(os.environ.get("SMTP_PORT", 587))
SMTP_USER = os.environ.get("SMTP_USER", "YOUR_SMTP_EMAIL_HERE")
SMTP_PASSWORD = os.environ.get("SMTP_PASSWORD", "YOUR_SMTP_APP_PASSWORD_HERE")

# Hardcoded Admin Contact Data Parameters
ADMIN_EMAIL_NOTIFY = "resegomanyapetsa7@gmail.com"
ADMIN_MOBILE_ALERT = "0605948171" 

YOCO_SECRET_KEY = os.environ.get("YOCO_SECRET_KEY", "sk_test_placeholder_key")
PAYFAST_MERCHANT_ID = os.environ.get("PAYFAST_MERCHANT_ID", "10000100") 
PAYFAST_MERCHANT_KEY = os.environ.get("PAYFAST_MERCHANT_KEY", "46f0cd694581a") 
PAYFAST_PASSPHRASE = os.environ.get("PAYFAST_PASSPHRASE", "your_payfast_secure_salt_phrase")
PAYFLEX_API_KEY = os.environ.get("PAYFLEX_API_KEY", "payflex_test_key")

STOCK_THRESHOLD = 10 

# Initialize CSV Files
if not os.path.isfile(INVENTORY_FILE):
    pd.DataFrame([
        {"style": "Classic 5-Panel", "stock": 150, "price": 250}, 
        {"style": "Structured 6-Panel", "stock": 120, "price": 300}
    ]).to_csv(INVENTORY_FILE, index=False)

if "cart" not in st.session_state: st.session_state.cart = []
if "authenticated" not in st.session_state: st.session_state.authenticated = False
if "auth_user" not in st.session_state: st.session_state.auth_user = "Guest"

# Core Security Signature Builder for Payfast Integration
def generate_payfast_signature(data_dict, passphrase=None):
    filtered = {k: v for k, v in data_dict.items() if v != "" and k != "signature"}
    sorted_keys = sorted(filtered.keys())
    encoded_str = ""
    for k in sorted_keys:
        val = urllib.parse.quote_plus(str(filtered[k]).replace('%2A', '*'))
        encoded_str += f"{k}={val}&"
    encoded_str = encoded_str[:-1]
    if passphrase:
        encoded_str += f"&passphrase={urllib.parse.quote_plus(passphrase.strip())}"
    return hashlib.md5(encoded_str.encode('utf-8')).hexdigest()

# API Gateway Routers
def init_yoco_payment(amount_cents, tracking_id):
    return f"https://yoco.com{YOCO_SECRET_KEY}&id={tracking_id}&amount={amount_cents}"

def init_payfast_payment(amount, tracking_id, item_name):
    base_url = "https://payfast.co.za" if "10000100" in PAYFAST_MERCHANT_ID else "https://payfast.co.za"
    params = {
        "merchant_id": PAYFAST_MERCHANT_ID,
        "merchant_key": PAYFAST_MERCHANT_KEY,
        "amount": f"{amount:.2f}",
        "item_name": item_name,
        "m_payment_id": tracking_id,
        "return_url": "https://riscoassociation.co.za",
        "cancel_url": "https://riscoassociation.co.za",
        "notify_url": "https://your-api-domain.com" 
    }
    params["signature"] = generate_payfast_signature(params, PAYFAST_PASSPHRASE)
    return f"{base_url}?{urllib.parse.urlencode(params)}"

def init_payflex_payment(amount, tracking_id):
    return f"https://payflex.co.za{tracking_id}&total={amount}"

def send_transactional_sms(phone_number, message_text):
    sanitized_phone = re.sub(r'[\s-]', '', str(phone_number))
    if sanitized_phone.startswith("0"): sanitized_phone = "+27" + sanitized_phone[1:]
    if "YOUR_BULKSMS" in BULKSMS_TOKEN_ID or not BULKSMS_TOKEN_ID:
        st.toast(f"📱 SMS Simulated: To {sanitized_phone} -> {message_text}", icon="ℹ️")
        return True
    url = "https://bulksms.com"
    payload = [{"to": sanitized_phone, "body": message_text}]
    data = json.dumps(payload).encode('utf-8')
    req = urllib.request.Request(url, data=data, headers={'Content-Type': 'application/json'})
    auth_encoded = base64.b64encode(f"{BULKSMS_TOKEN_ID}:{BULKSMS_TOKEN_SECRET}".encode()).decode()
    req.add_header("Authorization", f"Basic {auth_encoded}")
    try:
        with urllib.request.urlopen(req) as res: return res.status in (200, 201)
    except: return False

def trigger_admin_email_report(subject_text, body_text, excel_data=None, file_name="report.xlsx"):
    if "YOUR_SMTP" in SMTP_USER or not SMTP_USER:
        st.toast(f"📧 Email Notification Simulated: {subject_text}", icon="ℹ️")
        return True
    
    msg = MIMEMultipart()
    msg['From'] = SMTP_USER
    msg['To'] = ADMIN_EMAIL_NOTIFY
    msg['Subject'] = subject_text
    msg.attach(MIMEText(body_text, 'plain'))
    
    if excel_data:
        part = MIMEBase('application', 'octet-stream')
        part.set_payload(excel_data)
        encoders.encode_base64(part)
        part.add_header('Content-Disposition', f'attachment; filename="{file_name}"')
        msg.attach(part)
        
    try:
        server = smtplib.SMTP(SMTP_SERVER, SMTP_PORT)
        server.starttls()
        server.login(SMTP_USER, SMTP_PASSWORD)
        server.sendmail(SMTP_USER, ADMIN_EMAIL_NOTIFY, msg.as_string())
        server.quit()
        st.toast("📧 Automated ledger documentation email dispatched successfully!", icon="📩")
        return True
    except Exception as e:
        st.error(f"Email delivery configuration layout failure: {e}")
        return False

def get_inventory(): 
    return pd.read_csv(INVENTORY_FILE)

def update_stock(style, qty):
    df = get_inventory()
    df.loc[df["style"] == style, "stock"] -= qty
    new_stock = int(df.loc[df["style"] == style, "stock"].values[0])
    df.to_csv(INVENTORY_FILE, index=False)
    
    if new_stock 
            <img class="brand-logo" src="{LOGO_URL}" alt="Risco Association Logo">
        </div>
        """, unsafe_allow_html=True)
    
    st.header("👤 Client Portal")
    if not st.session_state.authenticated:
        user = st.text_input("Username", key="login_username")
        if st.button("Sign In"):
            if user.strip(): 
                st.session_state.authenticated = True
                st.session_state.auth_user = user
                st.rerun()
    else:
        st.write(f"Logged in as: **{st.session_state.auth_user}**")
        if st.button("Logout"): 
            st.session_state.authenticated = False
            st.session_state.auth_user = "Guest"
            st.rerun()

    st.header("🛒 Cart")
    if not st.session_state.cart: 
