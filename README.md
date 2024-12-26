import face_recognition
import cv2
import os
import time
from datetime import datetime
import pyttsx3
import telebot

# Telegram Configuration
TELEGRAM_BOT_TOKEN = "your_telegram_bot_token"
TELEGRAM_CHAT_ID = "your_telegram_chat_id"
bot = telebot.TeleBot('7472630823:AAFnYdjo9ylyjvY_OECHhaLlDLlfklDhAAw')

# Initialize Text-to-Speech
engine = pyttsx3.init()

# Function to load known faces and names
def load_known_faces(known_faces_dir):
    known_encodings = []
    known_names = []

    for file_name in os.listdir(known_faces_dir):
        file_path = os.path.join(known_faces_dir, file_name)
        try:
            image = face_recognition.load_image_file(file_path)
            face_locations = face_recognition.face_locations(image)
            if len(face_locations) == 0:
                print(f"Warning: No face found in {file_name}. Skipping.")
                continue
            face_encodings = face_recognition.face_encodings(image, face_locations)
            if face_encodings:
                known_encodings.append(face_encodings[0])
                known_names.append(os.path.splitext(file_name)[0])
        except Exception as e:
            print(f"Error processing file {file_name}: {e}")
    
    print("Loaded known faces and encodings successfully!")
    return known_encodings, known_names

# Function to send Telegram alerts
def send_telegram_alert(image_path, message="⚠ Unknown face detected!"):
    try:
        with open(image_path, "rb") as photo:
            bot.send_message(1402008649, message)
            bot.send_photo(1402008649, photo)
        print("Telegram alert sent successfully.")
    except Exception as e:
        print(f"Failed to send Telegram alert: {e}")

# Function to save unknown faces
def save_unknown_face(frame, location, save_dir="unknown_faces"):
    os.makedirs(save_dir, exist_ok=True)
    top, right, bottom, left = location
    unknown_face = frame[top:bottom, left:right]
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    file_name = os.path.join(save_dir, f"unknown_{timestamp}.jpg")
    cv2.imwrite(file_name, unknown_face)
    print(f"Unknown face saved to {file_name}")
    send_telegram_alert(file_name)

# Function to announce messages
def announce(message):
    engine.say(message)
    engine.runAndWait()

# Function to simulate door lock/unlock
def simulate_door_action(action="unlock", duration=5):
    if action == "unlock":
        print("🔓 Door unlocked.")
        announce("Door unlocked.")
    elif action == "lock":
        print("🔒 Door locked.")
        announce("Door locked.")
    time.sleep(duration)
    print("🔒 Door automatically locked.")
    announce("Door automatically locked.")

# Smart Home Face Lock Simulation
def smart_home_face_lock(tolerance=0.6):
    known_faces_dir = r"d:/dsp/known_faces"
    known_encodings, known_names = load_known_faces(known_faces_dir)

    # using the camera index as a password
    camera_index = int(input("Enter The password: ") or 0)
    video_capture = cv2.VideoCapture(camera_index)
    if not video_capture.isOpened():
        print("Error: Cannot access the webcam.")
        return

    print("Starting Smart Home Face ID Lock System. Press 'q' to quit.")
    prev_time = time.time()

    while True:
        ret, frame = video_capture.read()
        if not ret:
            print("Error: Failed to read frame from camera.")
            break

        rgb_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        face_locations = face_recognition.face_locations(rgb_frame)
        face_encodings = face_recognition.face_encodings(rgb_frame, face_locations)
        for (top, right, bottom, left), face_encoding in zip(face_locations, face_encodings):
            matches = face_recognition.compare_faces(known_encodings, face_encoding, tolerance=tolerance)
            name = "Unknown"
            access_granted = False
            if True in matches:
                matched_idx = matches.index(True)
                name = known_names[matched_idx]
                access_granted = True
                print(f"Access Granted: Welcome {name}!")
                announce(f"Access Granted. Welcome {name}!")
                # Log recognized face
                with open("recognition_log.txt", "a") as log_file:
                    log_file.write(f"{datetime.now()}: Recognized {name}\n")
                if name.lower() == "admin":  # Example: Admin-specific alert
                    bot.send_message(TELEGRAM_CHAT_ID, f"🔓 Admin {name} has unlocked the door.")
                simulate_door_action("unlock")
            else:
                save_unknown_face(frame, (top, right, bottom, left))
                print(f"Access Denied: Unknown face detected.")
                announce("Access Denied. Unknown face detected.")
                simulate_door_action("lock", duration=0)
            # Visualize result
            color = (0, 255, 0) if access_granted else (0, 0, 255)
            cv2.rectangle(frame, (left, top), (right, bottom), color, 2)
            cv2.putText(frame, name, (left, top - 10), cv2.FONT_HERSHEY_SIMPLEX, 0.9, color, 2)    
        cv2.imshow('Smart Home Face ID Lock', frame)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
    video_capture.release()
    cv2.destroyAllWindows()
    print("Video feed ended successfully.")
# Main entry point
if __name__ == "__main__":
    smart_home_face_lock()
