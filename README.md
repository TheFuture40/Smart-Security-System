# 🔐 Raspberry Pi Motion Detection Security System with Email Alerts

This project is a smart home security system built using a **Raspberry Pi**, **PIR motion sensor**, **push button**, and **buzzer**. When motion is detected, the system provides audio feedback and can send an email alert with a **live video stream link** via AWS SES.

---

## 📦 Components

| Component         | GPIO Pin | Description                        |
|-------------------|----------|------------------------------------|
| PIR Motion Sensor | GPIO22   | Detects motion                     |
| Push Button       | GPIO5    | Sends email + resets alert state  |
| Buzzer            | GPIO6    | Audible alert (1 beep = motion, 3 beeps = reset) |

- Uses **BCM** pin numbering (not physical board pins).

---

## ⚙️ Features

- 🟡 Detects motion using a PIR sensor
- 🔔 Buzzer feedback (1 beep = motion, 3 beeps = reset/email sent)
- 📧 Sends a **camera stream link** to your email using **AWS SES**
- 💡 Dual-mode logic: **interrupts** or **polling fallback**
- 🧹 Performs automatic GPIO cleanup

---

## 🧠 How It Works

1. The system sets up GPIOs and tests the buzzer.
2. When motion is detected:
   - A **single beep** is triggered.
   - Motion is only logged; no email is sent unless the button is pressed.
3. When the button is pressed:
   - The system **sends an email** with the local IP and stream URL.
   - The system **resets** its internal state for the next alert.

---

## 📋 Prerequisites

- Raspberry Pi with RPi.GPIO installed
- AWS SES (Simple Email Service) SMTP credentials
- Flask video stream server running on port 5000
- Sudo privileges to run script (`sudo python3 main.py`)

---

## 🧪 Code Overview

```python
import smtplib
from email.mime.text import MIMEText
import socket
import RPi.GPIO as GPIO
import time
import sys
import os
import atexit

PIR_PIN = 22
BUTTON_PIN = 5
BUZZER_PIN = 6
GPIO.setwarnings(True)

email_sent = False

def cleanup_gpio():
    print("Performing GPIO cleanup")
    try:
        GPIO.cleanup()
    except:
        pass

def setup_gpio():
    global email_sent
    email_sent = False
    print("Initializing GPIO...")
    try:
        if GPIO.getmode() is not None:
            GPIO.cleanup()
            time.sleep(0.5)
        GPIO.setmode(GPIO.BCM)
        GPIO.setup(PIR_PIN, GPIO.IN)
        print(f"PIR sensor on GPIO{PIR_PIN} configured")
        GPIO.setup(BUTTON_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)
        print(f"Button on GPIO{BUTTON_PIN} configured")
        GPIO.setup(BUZZER_PIN, GPIO.OUT)
        GPIO.output(BUZZER_PIN, GPIO.LOW)
        print(f"Buzzer on GPIO{BUZZER_PIN} configured")
        print("Testing buzzer...")
        beep_buzzer(2, 0.05)
        return True
    except Exception as e:
        print(f"GPIO initialization failed: {str(e)}")
        return False

def beep_buzzer(times=1, duration=0.1):
    try:
        for _ in range(times):
            GPIO.output(BUZZER_PIN, GPIO.HIGH)
            time.sleep(duration)
            GPIO.output(BUZZER_PIN, GPIO.LOW)
            time.sleep(duration)
    except Exception as e:
        print(f"Buzzer control error: {str(e)}")

def get_ip_address():
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
            s.connect(('8.8.8.8', 1))
            return s.getsockname()[0]
    except Exception:
        return '127.0.0.1'

def send_stream_email():
    try:
        ip_address = get_ip_address()
        msg = MIMEText(f"Raspberry Pi Stream: http://{ip_address}:5000/video")
        msg['Subject'] = "Camera Stream Access"
        msg['From'] = "your@email.com"
        msg['To'] = "your@email.com"

        with smtplib.SMTP("email-smtp.us-east-1.amazonaws.com", 587, timeout=15) as server:
            server.starttls()
            server.login("YOUR_AWS_SES_USERNAME", "YOUR_AWS_SES_PASSWORD")
            server.send_message(msg)
        print("Email sent successfully")
        return True
    except Exception as e:
        print(f"Email sending failed: {str(e)}")
        return False

def pir_callback(channel):
    global email_sent
    if GPIO.input(PIR_PIN):
        print("\nMotion detected!")
        beep_buzzer(1)

def button_callback(channel):
    global email_sent
    if not GPIO.input(BUTTON_PIN):
        print("\nButton pressed - sending stream link and resetting")
        beep_buzzer(3)
        send_stream_email()
        email_sent = False
        print("System reset - ready for new motion detections")

def setup_interrupts():
    try:
        GPIO.add_event_detect(PIR_PIN, GPIO.RISING, callback=pir_callback, bouncetime=300)
        GPIO.add_event_detect(BUTTON_PIN, GPIO.FALLING, callback=button_callback, bouncetime=500)
        print("Interrupts successfully configured")
        return True
    except RuntimeError as e:
        print(f"Interrupt setup failed: {str(e)}")
        return False

def polling_mode():
    print("Running in polling mode (fallback)")
    try:
        while True:
            if GPIO.input(PIR_PIN):
                pir_callback(PIR_PIN)
                time.sleep(1)
            if not GPIO.input(BUTTON_PIN):
                button_callback(BUTTON_PIN)
                while not GPIO.input(BUTTON_PIN):
                    time.sleep(0.1)
            time.sleep(0.1)
    except KeyboardInterrupt:
        pass

def main():
    atexit.register(cleanup_gpio)
    if not setup_gpio():
        sys.exit(1)
    if not setup_interrupts():
        print("\nFalling back to polling mode")
        polling_mode()
    else:
        print("\nSystem ready:")
        print("- Motion will trigger single beep")
        print("- Button press sends email + triple beep + resets")
        print("Press CTRL+C to exit")
        try:
            while True:
                time.sleep(1)
        except KeyboardInterrupt:
            pass

if __name__ == "__main__":
    if os.geteuid() != 0:
        print("ERROR: Must run with sudo privileges")
        sys.exit(1)
    print("Starting Raspberry Pi Security System")
    main()
    print("Program terminated")
