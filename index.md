# Ball Tracking Robot
The Ball Tracking Robot is a fully autonomous robot that detects and chases a red ball using computer vision and ultrasonic sensors. Built with a Raspberry Pi, Pi Camera, Ultrasonic sensors, and differential drive motors, the robot locates the ball from a distance, turns toward it, drives forward while dynamically adjusting speed, and avoids obstacles using three ultrasonic sensors.  

The biggest challenge was getting the robot to reliably approach the ball instead of just detecting it from afar, this required deep debugging of vision thresholds, sensor interference, and control logic. Working on this project gave me much experience in real-time computer vision, sensor fusion, and iterative robotics development.


| **Engineer** | **School** | **Area of Interest** | **Grade** |
|:--:|:--:|:--:|:--:|
| Aaron Z | BASIS Independent Silicon Valley | Electrical Engineering | Incoming Sophomore

**Replace the BlueStamp logo below with an image of yourself and your completed project. Follow the guide [here](https://tomcam.github.io/least-github-pages/adding-images-github-pages-site.html) if you need help.**

![Headstone Image](logo.svg)
  
# Final Milestone

**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/F7M7imOVGug" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

For your final milestone, explain the outcome of your project. Key details to include are:
- What you've accomplished since your previous milestone
- What your biggest challenges and triumphs were at BSE
- A summary of key topics you learned about
- What you hope to learn in the future after everything you've learned at BSE



# Second Milestone

**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/cmrQJw0E5GU?si=SIbBNiwHCmCM1aL-" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

I achieved full integration of the Pi Camera with computer vision. The robot can now reliably detect a red ball using HSV color filtering and contour detection. I implemented real-time decision logic that allows the robot to turn toward the ball when it’s off-center and drive forward when it’s aligned.  

I also refined the motor control (differential drive) and integrated ultrasonic sensor data for obstacle avoidance. One of the biggest surprises was how much the camera view could be blocked before detection failed; I learned it can’t be obscured by more than ~30%.  

The main challenge was making the robot actually approach the ball instead of just spotting it and spinning. This was solved through extensive tuning of area thresholds, turning deadzones, and speed curves.

# First Milestone

**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/df3oGgHPZj8?si=Cq0kGAjQgxcFnNKI" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

I focused on building the foundation of the robot, assembling the drivetrain using two DC motors in a differential drive configuration and installing three HC-SR04 ultrasonic sensors (left, center, and right) for obstacle detection.  

I set up the Raspberry Pi with motor drivers using gpiozero and tested basic movement commands (forward, backward, and turning). This milestone established the mechanical base and low-level motor + sensor control that the computer vision system would later build upon.  

The main challenge was getting smooth and reliable motor response with the Pi, which I solved by switching to the pigpio factory for better PWM control. This setup provided the mobility needed for the full ball-tracking behavior in later milestones.

# Schematics 
Here's where you'll put images of your schematics. [Tinkercad](https://www.tinkercad.com/blog/official-guide-to-tinkercad-circuits) and [Fritzing](https://fritzing.org/learning/) are both great resoruces to create professional schematic diagrams, though BSE recommends Tinkercad becuase it can be done easily and for free in the browser. 

# Code
```python
import RPi.GPIO as GPIO
import time
import cv2
import numpy as np
from picamera2 import Picamera2
from gpiozero import Motor, DistanceSensor
from gpiozero.pins.pigpio import PiGPIOFactory

# ====================== SETUP ======================
GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)

try:
    factory = PiGPIOFactory()
    print("Connected to pigpio")
except Exception as e:
    print("Failed to connect to pigpio:", e)
    exit()

left = Motor(forward=23, backward=24, pin_factory=factory)
right = Motor(forward=27, backward=17, pin_factory=factory)

lsense = DistanceSensor(echo=15, trigger=14, pin_factory=factory)
centsense = DistanceSensor(echo=13, trigger=6, pin_factory=factory)
rsense = DistanceSensor(echo=9, trigger=10, pin_factory=factory)

motorspd = 0.85
min_speed = motorspd * 0.7

sensor_proximity = 10.0        
rerouting_proximity = 17.5

# Camera
picam2 = Picamera2()
config = picam2.create_preview_configuration(main={"size": (640, 480)})
picam2.configure(config)
picam2.start()
print("Camera started - Smart Ball Tracker Running")

last_action = ""
flag = 0
ball_lost_time = time.time()
lost_threshold = 80.0

def print_once(action):
    global last_action
    if action != last_action:
        print(f"\n→ {action}")
        last_action = action

# ====================== MOTOR FUNCTIONS ======================
def stop():
    left.stop()
    right.stop()

def driveforward(speed=None):
    if speed is None: speed = motorspd
    speed = max(min_speed, speed)
    left.forward(speed=speed)
    right.forward(speed=speed)

def drivebackward(speed=None):
    if speed is None: speed = motorspd
    speed = max(min_speed, speed)
    left.backward(speed=speed)
    right.backward(speed=speed)

def leftturn():
    left.backward(speed=min_speed)
    right.forward(speed=min_speed)

def rightturn():
    left.forward(speed=min_speed)
    right.backward(speed=min_speed)

def sharp_left():
    left.backward(speed=min_speed)
    right.stop()

def sharp_right():
    left.forward(speed=min_speed)
    right.stop()

def back_left():
    left.backward(speed=min_speed)
    right.stop()

def back_right():
    left.stop()
    right.backward(speed=min_speed)

# ====================== BALL DETECTION ======================
def segment_colour(frame):
    hsv_roi = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
    mask = cv2.inRange(hsv_roi, np.array([150, 140, 1]), np.array([190, 255, 255]))
    mask = cv2.erode(mask, np.ones((3,3),np.uint8))
    mask = cv2.dilate(mask, np.ones((8,8),np.uint8))
    cv2.imshow('mask', mask)
    return mask

def find_blob(blob):
    contours, _ = cv2.findContours(blob, cv2.RETR_CCOMP, cv2.CHAIN_APPROX_SIMPLE)
    if not contours:
        return (0,0,2,2), 0
    largest = max(contours, key=cv2.contourArea)
    return cv2.boundingRect(largest), cv2.contourArea(largest)


# ====================== MAIN LOOP ======================
try:
    print("Robot started. Press 'q' to quit.\n")
    
    while True:
        frame = picam2.capture_array()
        frame_bgr = cv2.cvtColor(frame, cv2.COLOR_RGB2BGR)
        height, width = frame_bgr.shape[:2]
        
        ldist = lsense.distance * 100
        cdist = centsense.distance * 100
        rdist = rsense.distance * 100
        
        mask_red = segment_colour(frame_bgr)
        loct, area = find_blob(mask_red)
        x, y, w, h = loct
        
        found = (w * h) > 350                        
        center_x = x + w/2 if found else 0
        
        if found:
            cv2.rectangle(frame_bgr, (x, y), (x+w, y+h), (0, 255, 0), 2)
            cv2.circle(frame_bgr, (int(center_x), int(y + h/2)), 6, (0, 110, 255), -1)
        
        # ==================== DECISION LOGIC ====================
        print_once(f"Ball: area={area:>6}  cx={center_x:>4.0f}  cdist={cdist:>5.1f}cm  l={ldist:>5.1f} r={rdist:>5.1f}")

        if found:
            # Improved "very close" detection - mainly relies on vision size + center sensor
            ball_is_very_close = (area > 65000) or ((cdist < 17) or (ldist < 17) or (rdist < 17))
            
            if ball_is_very_close:
                stop()
                print_once("BALL REACHED")
                
            elif ldist > sensor_proximity and cdist > sensor_proximity and rdist > sensor_proximity:
                # Dynamic speed - slow down as ball gets bigger
                speed = max(min_speed, motorspd * (1 - (area / 42000)))
                
                if center_x < 110:
                    leftturn()
                    print_once("Turning LEFT toward ball")
                    flag = 0
                elif center_x > width - 110:
                    rightturn()
                    print_once("Turning RIGHT toward ball")
                    flag = 1
                else:
                    driveforward(speed)
                    print_once(f"Moving FORWARD (speed: {speed:.2f})")
                    
            else:
                # Obstacle handling while ball is visible
                stop()
                if ldist < sensor_proximity or cdist < sensor_proximity or rdist < sensor_proximity:
                    drivebackward()
                    print_once("REVERSING - obstacle detected")
                    time.sleep(0.15)
                    stop()
                if cdist < sensor_proximity + 5 or area > 18000:
                    print_once("PARKED")
                elif ldist < rerouting_proximity:
                    back_left()
                    time.sleep(0.25)
                    print_once("Rerouting RIGHT")
                elif rdist < rerouting_proximity:
                    back_right()
                    time.sleep(0.25)
                    print_once("Rerouting LEFT")
                else:
                    drivebackward(0.5)
                    time.sleep(0.3)
                    stop()
        else:
            # Ball lost logic
            if ldist > sensor_proximity or cdist > sensor_proximity or rdist > sensor_proximity:
                print_once("Searching for ball...")
                sharp_left() if flag == 0 else sharp_right()
                time.sleep(0.13)
            else:
                drivebackward()
                print_once("REVERSING - obstacle while searching")
                time.sleep(0.45)
                stop()

        if ldist < sensor_proximity or cdist < sensor_proximity or rdist < sensor_proximity:
                    drivebackward()
                    print_once("REVERSING - obstacle detected")
                    time.sleep(0.15)
                    stop()

        cv2.imshow("Ball Tracker", frame_bgr)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            print("\nQuitting...")
            break
        
        time.sleep(0.01)

except Exception as e:
    print("Error:", e)
finally:
    stop()
    picam2.stop()
    cv2.destroyAllWindows()
    GPIO.cleanup()
    print("Robot stopped and cleaned up.")
```

# Bill of Materials
Here's where you'll list the parts in your project. To add more rows, just copy and paste the example rows below.
Don't forget to place the link of where to buy each component inside the quotation marks in the corresponding row after href =. Follow the guide [here]([url](https://www.markdownguide.org/extended-syntax/)) to learn how to customize this to your project needs. 

| **Part** | **Note** | **Price** | **Link** |
|:--:|:--:|:--:|:--:|
| Item Name | What the item is used for | $Price | <a href="https://www.amazon.com/Arduino-A000066-ARDUINO-UNO-R3/dp/B008GRTSV6/"> Link </a> |
| Item Name | What the item is used for | $Price | <a href="https://www.amazon.com/Arduino-A000066-ARDUINO-UNO-R3/dp/B008GRTSV6/"> Link </a> |
| Item Name | What the item is used for | $Price | <a href="https://www.amazon.com/Arduino-A000066-ARDUINO-UNO-R3/dp/B008GRTSV6/"> Link </a> |

# Other Resources/Examples
One of the best parts about Github is that you can view how other people set up their own work. Here are some past BSE portfolios that are awesome examples. You can view how they set up their portfolio, and you can view their index.md files to understand how they implemented different portfolio components.
- [Example 1](https://trashytuber.github.io/YimingJiaBlueStamp/)
- [Example 2](https://sviatil0.github.io/Sviatoslav_BSE/)
- [Example 3](https://arneshkumar.github.io/arneshbluestamp/)
