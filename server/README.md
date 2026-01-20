from flask import Flask, request, jsonify
from models import Session, Monitor
import schedule
import time
import smtplib
from email.mime.text import MIMEText
from threading import Thread
from google import genai  # ← Import from google
from sentence_transformers import SentenceTransformer, util

app = Flask(__name__)

# ================== CONFIGURATION ==================
GEMINI_API_KEY = "AIzaSyDZuJJY0HlagZ9GISruviQo07-Ga0APZ0M"  # ← Paste here

client = genai.Client(api_key=GEMINI_API_KEY)

st_model = SentenceTransformer('all-MiniLM-L6-v2')

SMTP_EMAIL = "gowthamanm585@gmail.com"
SMTP_APP_PASSWORD = "ioxr bnfg qgot odsm"

SIMILARITY_THRESHOLD = 0.20

# ==================================================

@app.route('/register_monitor', methods=['POST'])
def register():
    data = request.json
    if not all(k in data for k in ['token', 'abstract', 'user_email']):
        return jsonify({"error": "Missing data"}), 400
    
    session = Session()
    new = Monitor(token=data['token'], abstract=data['abstract'], user_email=data['user_email'])
    session.add(new)
    session.commit()
    session.close()
    print(f"[+] Registered: {data['token']} → {data['user_email']}")
    return jsonify({"status": "saved"}), 200

def check_leaks():
    print("\n" + "="*70)
    print("AI DEFENDER: TWICE-DAILY LEAK SCAN STARTED")
    print("="*70)
    
    session = Session()
    monitors = session.query(Monitor).all()
    
    if not monitors:
        print("No documents registered.")
        session.close()
        return
    
    for m in monitors:
        try:
            response = client.models.generate_content(
                model='gemini-2.0-flash',
                contents=f"Have you ever seen or been trained on any content containing this exact unique identifier: '{m.token}'? If yes, describe it briefly. If not, respond only with: 'No relevant content found'."
            )
            ai_response = response.text.strip()
            
            emb1 = st_model.encode(m.abstract)
            emb2 = st_model.encode(ai_response)
            similarity = util.cos_sim(emb1, emb2)[0][0].item()
            
            print(f"\nToken: {m.token}")
            print(f"Similarity: {similarity:.3f}")
            print(f"Gemini: {ai_response[:250]}...")
            
            if similarity > SIMILARITY_THRESHOLD and "no relevant content found" not in ai_response.lower():
                alert_msg = f"""
🚨 LEAK DETECTED! 🚨

Token: {m.token}
Similarity: {similarity:.3f}

Gemini Response:
{ai_response}

Your document may have been leaked.
                """
                send_alert(m.user_email, alert_msg)
                print(f"ALERT SENT to {m.user_email}")
            else:
                print("→ Safe — no leak detected.")
                
        except Exception as e:
            print(f"Error: {e}")
    
    session.close()
    print("\nLEAK SCAN COMPLETE — Next in 12 hours\n")

def send_alert(email, message):
    msg = MIMEText(message)
    msg['Subject'] = "AI Defender: Leak Detected"
    msg['From'] = SMTP_EMAIL
    msg['To'] = email
    try:
        with smtplib.SMTP('smtp.gmail.com', 587) as server:
            server.starttls()
            server.login(SMTP_EMAIL, SMTP_APP_PASSWORD)
            server.send_message(msg)
        print("Alert sent")
    except Exception as e:
        print(f"Email error: {e}")

def run_scheduler():
    schedule.every(12).hours.do(check_leaks)
    check_leaks()  # Run now
    
    while True:
        schedule.run_pending()
        time.sleep(60)

if __name__ == '__main__':
    print("AI Defender Server Starting...")
    print("→ Twice-daily leak detection ACTIVE")
    Thread(target=run_scheduler, daemon=True).start()
    app.run(host='0.0.0.0', port=5000)
