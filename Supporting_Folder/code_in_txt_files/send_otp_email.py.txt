import sys
import smtplib
import ssl
from email.mime.text import MIMEText

# Ensure the script receives the required command-line arguments
if len(sys.argv) < 4:
    print("Usage: send_otp_email.py <email> <otp> <nameSurname>")
    sys.exit(1)

recipient_email = sys.argv[1]
otp = sys.argv[2]
nameSurname = sys.argv[3]

# Sender email credentials (should be securely stored in production)
EMAIL = "thebestbankingsystemproject@gmail.com"
PASSWORD = "skqo kamo qcti hvdb"

# Abort if credentials are missing
if not EMAIL or not PASSWORD:
    print("GMAIL_USER or GMAIL_PASS environment variable not set.")
    sys.exit(1)

# Compose the plain text email body with the OTP and user name
body = f"""Dear {nameSurname},

Your OTP is: {otp}

This code is valid for 1 minute.

Thank you for using our banking system!
"""

# Create the email message object
msg = MIMEText(body)
msg['Subject'] = "Your One-Time Password (OTP)"
msg['From'] = EMAIL
msg['To'] = recipient_email

# Send the email using Gmail's SMTP server with SSL
context = ssl.create_default_context()
try:
    with smtplib.SMTP_SSL("smtp.gmail.com", 465, context=context) as server:
        server.login(EMAIL, PASSWORD)
        server.sendmail(EMAIL, recipient_email, msg.as_string())
    print("A one-time password (OTP) has been sent to your email.")
except Exception as e:
    print("Failed to send email: invalid or non-existent email")
    sys.exit(2)