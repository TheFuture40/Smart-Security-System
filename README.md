PIR_PIN = 22      # Motion sensor
BUTTON_PIN = 5    # Reset button
BUZZER_PIN = 6    # Audible alarm
- Defines which GPIO pins connect to the components
- Uses BCM numbering meaning it uses GPIO numbers rather than physical pin numbers



email_sent = False  # Global flag to track email status
- Prevents duplicate email alerts
- Gets reset when button is pressed



def setup_gpio():
    GPIO.setmode(GPIO.BCM)
    GPIO.setup(PIR_PIN, GPIO.IN)          # PIR as input
    GPIO.setup(BUTTON_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)  # Button with pull-up
    GPIO.setup(BUZZER_PIN, GPIO.OUT)      # Buzzer as output
A function that: 
- Configures all hardware components
- Includes pull-up resistor for button to prevent floating state



def beep_buzzer(times=1, duration=0.1):
    for _ in range(times):
        GPIO.output(BUZZER_PIN, GPIO.HIGH)
        time.sleep(duration)
        GPIO.output(BUZZER_PIN, GPIO.LOW)
A function that generates audible feedback patterns:
1 beep = Motion detected
3 beeps = Button pressed



def send_stream_email():
    msg = MIMEText(f"Stream: http://{get_ip_address()}:5000/video")
    # SMTP configuration...
An email function that:
- Sends email with current IP and stream link
- Uses my AWS SES SMTP credentials



def pir_callback(channel):
    if GPIO.input(PIR_PIN):
        beep_buzzer(1)  # Single beep
Sensor Function:
- Triggered when PIR detects movement
- Provides immediate audible feedback




def button_callback(channel):
    beep_buzzer(3)             # Triple beep
    send_stream_email()        # Send link
    global email_sent
    email_sent = False         # Reset state
Multi-function button:
- Sends stream link to my gmail
- Resets alert system
- Provides confirmation beeps(three)



while True:
    if GPIO.input(PIR_PIN):    # Manual motion check
    if not GPIO.input(BUTTON_PIN):  # Manual button check
- Active backup if interrupts fail
- Continuously checks sensor states



atexit.register(cleanup_gpio)
def cleanup_gpio():
    GPIO.cleanup()
- Ensures proper GPIO reset on exit
- Prevents pin state persistence

