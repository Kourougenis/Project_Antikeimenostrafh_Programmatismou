import sys
import smtplib
import ssl
from email.mime.text import MIMEText
import os

# Check command-line arguments for recipient email and full name
if len(sys.argv) < 3:
    print("Usage: python send_interest_info_email.py <recipient_email> <nameSurname>")
    sys.exit(1)

recipient_email = sys.argv[1]
nameSurname = sys.argv[2]

# Set sender email credentials (should be replaced with secure storage in production)
EMAIL = "thebestbankingsystemproject@gmail.com"
PASSWORD = "skqo kamo qcti hvdb"

# Abort if credentials are missing
if not EMAIL or not PASSWORD:
    print("GMAIL_USER or GMAIL_PASS environment variable not set.")
    sys.exit(1)

# Prepare the HTML email body with personalized greeting and interest info
html_body = f"""
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Welcome & Interest Growth</title>
  <style>
    body {{
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      background: #f9f9f9;
      margin: 0;
      padding: 0;
      color: #333;
    }}
    .container {{
      background: #ffffff;
      max-width: 600px;
      margin: 30px auto;
      padding: 30px;
      border-radius: 12px;
      box-shadow: 0 4px 20px rgba(0,0,0,0.1);
      text-align: center;
    }}
    h1 {{
      color: #2e86de;
      margin-bottom: 10px;
    }}
    p {{
      font-size: 16px;
      line-height: 1.6;
      margin: 15px 0;
    }}
    .highlight {{
      color: #27ae60;
      font-weight: 700;
      font-size: 18px;
    }}
    .emoji {{
      font-size: 30px;
      margin-bottom: 10px;
    }}
    .button {{
      display: inline-block;
      background-color: #2e86de;
      color: white;
      text-decoration: none;
      padding: 14px 28px;
      border-radius: 30px;
      font-weight: bold;
      margin-top: 25px;
      transition: background-color 0.3s ease;
    }}
    .button:hover {{
      background-color: #1b4f91;
    }}
    footer {{
      margin-top: 30px;
      font-size: 12px;
      color: #999;
    }}
  </style>
</head>
<body>
  <div class="container">
    <div class="emoji">💰✨</div>
    <h1>Welcome to The Best Banking System!</h1>
    <p><strong>Dear {nameSurname},</strong></p>
    <p>Thank you for choosing the most reliable and innovative bank.</p>
    <p>With every deposit you make, you earn a <span class="highlight">3% interest</span> compounded on <strong>June 30</strong> and <strong>December 31</strong>.</p>
    <p>Imagine your money growing effortlessly just because you trusted us.</p>
    <p>This isn’t just banking. It’s an <strong>investment in your future</strong>! 🚀</p>
    <a href="#" class="button">Learn More & Benefit Today</a>
    <footer>
      The Best Banking System – Security and growth by your side
    </footer>
  </div>
</body>
</html>
"""

# Create the MIME email object
msg = MIMEText(html_body, "html", "utf-8")
msg['Subject'] = 'Welcome! Let Your Money Thrive with Smart Saving.💰'
msg['From'] = EMAIL
msg['To'] = recipient_email

# Send the email using Gmail's SMTP server
context = ssl.create_default_context()
try:
    with smtplib.SMTP_SSL("smtp.gmail.com", 465, context=context) as server:
        server.login(EMAIL, PASSWORD)
        server.sendmail(EMAIL, recipient_email, msg.as_string())
    print("Interest details have been sent to your email!")
except Exception as e:
    print(f"Failed to send interest details email: {e}")