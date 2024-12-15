import face_recognition
import cv2
import os

# Function to load known faces and names
def load_known_faces(known_faces_dir):
    known_encodings = []
    known_names = []

    # Iterate through each file in the known_faces directory
    for file_name in os.listdir(known_faces_dir):
        file_path = os.path.join(known_faces_dir, file_name)

        try:
            print(f"Processing file: {file_path}")
            image = face_recognition.load_image_file(file_path)
            face_locations = face_recognition.face_locations(image)
            
            # Skip files where no faces are detected
            if len(face_locations) == 0:
                print(f"Warning: No face found in {file_name}. Skipping.")
                continue

            # Encode the first face found
            face_encodings = face_recognition.face_encodings(image, face_locations)
            if face_encodings:
                known_encodings.append(face_encodings[0])
                # Extract the name without file extension
                known_names.append(os.path.splitext(file_name)[0])
            else:
                print(f"Warning: Could not encode face in {file_name}. Skipping.")

        except Exception as e:
            print(f"Error processing file {file_name}: {e}")
    
    print("Loaded known faces and encodings successfully!")
    return known_encodings, known_names

# Function for real-time face detection and recognition
def face_detection_with_names():
    # Directory containing images of known faces
    known_faces_dir = r"d:/dsp/known_faces"  # Use absolute path
    known_encodings, known_names = load_known_faces(known_faces_dir)

    # Initialize the video capture (0 for default webcam)
    video_capture = cv2.VideoCapture(0)
    if not video_capture.isOpened():
        print("Error: Cannot access the webcam.")
        return

    print("Starting real-time face detection. Press 'q' to quit.")

    while True:
        ret, frame = video_capture.read()
        if not ret:
            print("Error: Failed to read frame from camera.")
            break

        # Convert the frame to RGB (face_recognition uses RGB images)
        rgb_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)

        # Detect faces and compute encodings
        face_locations = face_recognition.face_locations(rgb_frame)
        face_encodings = face_recognition.face_encodings(rgb_frame, face_locations)

        # Compare detected faces with known faces
        for (top, right, bottom, left), face_encoding in zip(face_locations, face_encodings):
            matches = face_recognition.compare_faces(known_encodings, face_encoding)
            name = "Unknown"

            # Check if there's a match
            if True in matches:
                matched_idx = matches.index(True)
                name = known_names[matched_idx]

            # Draw a rectangle around the face
            cv2.rectangle(frame, (left, top), (right, bottom), (0, 255, 0), 2)

            # Display the name of the person
            cv2.putText(frame, name, (left, top - 10), cv2.FONT_HERSHEY_SIMPLEX, 0.9, (0, 255, 0), 2)

        # Display the frame
        cv2.imshow('Face Detection with Names', frame)

        # Press 'q' to quit the video feed
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

    # Release the video capture and close windows
    video_capture.release()
    cv2.destroyAllWindows()
    print("Video feed ended successfully.")

# Main entry point
if __name__ == "__main__":
    face_detection_with_names()
