# Smart-Security-System
Motion detecting alarm using Raspberry Pi, Pi Camera, and Python

import smtplib
from email.mime.text import MIMEText
import socket
import RPi.GPIO as GPIO
import time
import sys
import os
import atexit
# GPIO Configuration
PIR_PIN = 22      # GPIO22 for PIR motion sensor
BUTTON_PIN = 5    # GPIO5 for button
BUZZER_PIN = 6    # GPIO6 for buzzer
GPIO.setwarnings(True)
# State variables
email_sent = False
def cleanup_gpio():
    """Ensure proper GPIO cleanup"""
    print("Performing GPIO cleanup")
    try:
        GPIO.cleanup()
    except:
        pass
def setup_gpio():
    """Enhanced GPIO initialization with verification"""
    global email_sent
    email_sent = False
    
    print("Initializing GPIO...")
    try:
        # Clean up any previous configurations
        if GPIO.getmode() is not None:
            GPIO.cleanup()
            time.sleep(0.5)
        
        GPIO.setmode(GPIO.BCM)
        
        # Configure PIR sensor
        GPIO.setup(PIR_PIN, GPIO.IN)
        print(f"PIR sensor on GPIO{PIR_PIN} configured")
        
        # Configure button with pull-up
        GPIO.setup(BUTTON_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)
        print(f"Button on GPIO{BUTTON_PIN} configured")
        
        # Configure buzzer
        GPIO.setup(BUZZER_PIN, GPIO.OUT)
        GPIO.output(BUZZER_PIN, GPIO.LOW)
        print(f"Buzzer on GPIO{BUZZER_PIN} configured")
        
        # Quick hardware test
        print("Testing buzzer...")
        beep_buzzer(2, 0.05)
        return True
        
    except Exception as e:
        print(f"GPIO initialization failed: {str(e)}")
        return False
def beep_buzzer(times=1, duration=0.1):
    """Buzzer control with customizable beeps"""
    try:
        for _ in range(times):
            GPIO.output(BUZZER_PIN, GPIO.HIGH)
            time.sleep(duration)
            GPIO.output(BUZZER_PIN, GPIO.LOW)
            time.sleep(duration)
    except Exception as e:
        print(f"Buzzer control error: {str(e)}")
def get_ip_address():
    """Get reliable IP address"""
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
            s.connect(('8.8.8.8', 1))
            return s.getsockname()[0]
    except Exception:
        return '127.0.0.1'
def send_stream_email():
    """Send email with stream link"""
    try:
        ip_address = get_ip_address()
        msg = MIMEText(f"Raspberry Pi Stream: http://{ip_address}:5000/video")
        msg['Subject'] = "Camera Stream Access"
        msg['From'] = "kebefallou68@gmail.com"
        msg['To'] = "kebefallou68@gmail.com"
        with smtplib.SMTP("email-smtp.us-east-1.amazonaws.com", 587, timeout=15) as server:
            server.starttls()
            server.login("AWS SMTP", "AWS SMTP")
            server.send_message(msg)
        print("Email sent successfully")
        return True
    except Exception as e:
        print(f"Email sending failed: {str(e)}")
        return False
def pir_callback(channel):
    """Handle PIR motion detection"""
    global email_sent
    
    if GPIO.input(PIR_PIN):
        print("\nMotion detected!")
        beep_buzzer(1)  # Single beep for motion
        # No email sent for motion in this version
def button_callback(channel):
    """Handle button press - reset PIR and send email"""
    global email_sent
    
    if not GPIO.input(BUTTON_PIN):  # Button pressed
        print("\nButton pressed - sending stream link and resetting")
        beep_buzzer(3)  # Triple beep for button press
        
        # Send stream link email
        send_stream_email()
        
        # Reset PIR state
        email_sent = False
        print("System reset - ready for new motion detections")
def setup_interrupts():
    """Configure sensor and button interrupts"""
    try:
        # Setup PIR sensor interrupt
        GPIO.add_event_detect(PIR_PIN, GPIO.RISING, callback=pir_callback, bouncetime=300)
        
        # Setup button interrupt
        GPIO.add_event_detect(BUTTON_PIN, GPIO.FALLING, callback=button_callback, bouncetime=500)
        
        print("Interrupts successfully configured")
        return True
    except RuntimeError as e:
        print(f"Interrupt setup failed: {str(e)}")
        return False
def polling_mode():
    """Fallback polling mode"""
    print("Running in polling mode (fallback)")
    try:
        while True:
            # Check PIR sensor
            if GPIO.input(PIR_PIN):
                pir_callback(PIR_PIN)
                time.sleep(1)  # Debounce motion
            
            # Check button
            if not GPIO.input(BUTTON_PIN):
                button_callback(BUTTON_PIN)
                while not GPIO.input(BUTTON_PIN):  # Wait for release
                    time.sleep(0.1)
            
            time.sleep(0.1)
    except KeyboardInterrupt:
        pass
def main():
    # Register cleanup handler
    atexit.register(cleanup_gpio)
    
    if not setup_gpio():
        sys.exit(1)
    
    # Attempt interrupt setup
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
