import os
import smtplib
import logging
from email.message import EmailMessage

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s"
)

def send_plain(to_email: str, subject: str, body: str) -> bool:
    host = os.getenv("SMTP_HOST")
    port = int(os.getenv("SMTP_PORT", "587"))
    user = os.getenv("SMTP_USER")
    password = os.getenv("SMTP_PASS")
    from_addr = os.getenv("FROM")

    msg = EmailMessage()
    msg["From"] = from_addr
    msg["To"] = to_email
    msg["Subject"] = subject
    msg.set_content(body)

    try:
        logging.info("Connecting to SMTP server...")
        with smtplib.SMTP(host, port, timeout=10) as smtp:
            logging.info("Starting TLS...")
            smtp.starttls()

            logging.info("Authenticating...")
            smtp.login(user, password)

            logging.info("Sending message...")
            smtp.send_message(msg)

        logging.info("Email sent successfully.")
        return True

    except Exception as e:
        logging.error(f"Failed to send email: {e}")
        return False
