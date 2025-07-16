import time
import RPi.GPIO as GPIO
import requests
from smbus2 import SMBus
# For GPS, you would use something like 'gpsd' or serial, but here we'll mock it
import random

# WiFi and server details (for ThingSpeak)
API_KEY = "6W1WCDW0529DRH66"
SERVER = "https://api.thingspeak.com/update"

# LCD details
LCD_ADDR = 0x27
LCD_COLS = 16
LCD_ROWS = 2

# Pin definitions
TRIG_PIN = 18
ECHO_PIN = 19
LED_RED_1 = 13
LED_RED_2 = 17
LED_RED_3 = 15
LED_RED_4 = 16

# Set up GPIO
GPIO.setmode(GPIO.BCM)
GPIO.setup(TRIG_PIN, GPIO.OUT)
GPIO.setup(ECHO_PIN, GPIO.IN)
GPIO.setup(LED_RED_1, GPIO.OUT)
GPIO.setup(LED_RED_2, GPIO.OUT)
GPIO.setup(LED_RED_3, GPIO.OUT)
GPIO.setup(LED_RED_4, GPIO.OUT)

# LCD setup (assuming PCF8574 and HD44780, you might need a library)
bus = SMBus(1)
def lcd_clear():
    # Implement your LCD clear function
    pass
def lcd_print(row, message):
    # Implement your LCD print function
    pass

def read_distance():
    GPIO.output(TRIG_PIN, False)
    time.sleep(0.002)
    GPIO.output(TRIG_PIN, True)
    time.sleep(0.00001)
    GPIO.output(TRIG_PIN, False)
    while GPIO.input(ECHO_PIN) == 0:
        pulse_start = time.time()
    while GPIO.input(ECHO_PIN) == 1:
        pulse_end = time.time()
    pulse_duration = pulse_end - pulse_start
    distance = pulse_duration * 17150
    distance = round(distance, 2)
    return distance

def get_gps_location():
    # Replace with actual GPS reading
    lat = random.uniform(17, 18)  # Simulated latitude
    lon = random.uniform(78, 79)  # Simulated longitude
    return round(lat, 6), round(lon, 6)

def send_message(msg, lat, lon):
    print(f"SMS: {msg} http://www.google.com/maps/place/{lat},{lon}")
    lcd_clear()
    lcd_print(0, "Message sent")
    time.sleep(2)

def send_to_thingspeak(dust_level):
    data = {
        'api_key': API_KEY,
        'field1': dust_level
    }
    try:
        r = requests.post(SERVER, data=data)
        if r.status_code == 200:
            lcd_clear()
            lcd_print(0, "Data sent to")
            lcd_print(1, "web server")
            time.sleep(1)
        else:
            print("Failed to send data")
    except Exception as e:
        print(f"Error: {e}")

def main():
    lcd_clear()
    lcd_print(0, "Ultrasonic Test")
    time.sleep(2)
    lcd_clear()
    lcd_print(0, "WiFi Connected")
    time.sleep(1)

    while True:
        distance = read_distance()
        lcd_clear()
        lcd_print(0, f"Distance: {distance} cm")
        time.sleep(2)

        dustLevel = 0
        if distance == 9:
            dustLevel = 0
        elif distance == 7:
            dustLevel = 25
        elif distance == 5:
            dustLevel = 50
        elif distance == 3:
            dustLevel = 75
        elif distance == 0:
            dustLevel = 100

        GPIO.output(LED_RED_1, dustLevel >= 25)
        GPIO.output(LED_RED_2, dustLevel >= 50)
        GPIO.output(LED_RED_3, dustLevel >= 75)
        GPIO.output(LED_RED_4, dustLevel == 100)

        if dustLevel in [25, 50, 75, 100]:
            lcd_clear()
            lcd_print(0, f"Dust level: {dustLevel}%")
            time.sleep(2)
            msg = f"Dust is filled upto {dustLevel}% in the bin"
            lat, lon = get_gps_location()
            send_message(msg, lat, lon)
            time.sleep(1)

        send_to_thingspeak(dustLevel)
        time.sleep(15)
        lat, lon = get_gps_location()
        lcd_clear()
        lcd_print(0, f"lat: {lat}")
        lcd_print(1, f"lng: {lon}")
        time.sleep(1)

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        GPIO.cleanup()
