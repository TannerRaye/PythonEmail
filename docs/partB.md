import os
import mimetypes
import smtplib
import logging
from email.message import EmailMessage
from email.utils import make_msgid, formatdate

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s"
)

def send_html(
    to_email: str,
    subject: str,
    html_body: str,
    attachments: list[str] | None = None,
    inline_images: dict[str, str] | None = None,
    cc: list[str] | None = None,
    bcc: list[str] | None = None
):
    host = os.getenv("SMTP_HOST")
    port = int(os.getenv("SMTP_PORT", "587"))
    user = os.getenv("SMTP_USER")
    password = os.getenv("SMTP_PASS")
    from_addr = os.getenv("FROM")

    msg = EmailMessage()
    msg["From"] = from_addr
    msg["To"] = to_email
    msg["Subject"] = subject
    msg["Date"] = formatdate(localtime=True)
    msg["Message-ID"] = make_msgid()

    if cc:
        msg["Cc"] = ", ".join(cc)

    # Plain-text fallback + HTML version
    msg.set_content("Your email client does not support HTML.")
    msg.add_alternative(html_body, subtype="html")

    # Inline images (multipart/related)
    if inline_images:
        for cid, path in inline_images.items():
            with open(path, "rb") as f:
                img_data = f.read()

            mime_type, _ = mimetypes.guess_type(path)
            maintype, subtype = mime_type.split("/")

            msg.get_payload()[1].add_related(
                img_data,
                maintype=maintype,
                subtype=subtype,
                cid=f"<{cid}>"
            )

    # Attachments (multipart/mixed)
    if attachments:
        for file_path in attachments:
            mime_type, _ = mimetypes.guess_type(file_path)
            maintype, subtype = mime_type.split("/")

            with open(file_path, "rb") as f:
                file_data = f.read()

            msg.add_attachment(
                file_data,
                maintype=maintype,
                subtype=subtype,
                filename=os.path.basename(file_path)
            )

    # Final recipient list
    all_recipients = [to_email]
    if cc:
        all_recipients += cc
    if bcc:
        all_recipients += bcc

    try:
        logging.info("Connecting to SMTP server...")
        with smtplib.SMTP(host, port, timeout=10) as smtp:
            logging.info("Starting TLS...")
            smtp.starttls()

            logging.info("Authenticating...")
            smtp.login(user, password)

            logging.info("Sending HTML email...")
            smtp.send_message(msg, from_addr=from_addr, to_addrs=all_recipients)

        logging.info("HTML email sent successfully.")
        return True

    except Exception as e:
        logging.error(f"Failed to send HTML email: {e}")
        return False
