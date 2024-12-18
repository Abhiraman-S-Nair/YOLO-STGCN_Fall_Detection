# Installing Libraries
```
!pip install ultralytics torch torchvision opencv-python mediapipe numpy matplotlib tqdm twilio
"""
- ultralytics: Provides easy-to-use YOLO implementations for object detection and classification tasks.
- torch: PyTorch library for building and training neural networks.
- torchvision: Useful for image processing, data transformations, and handling pre-trained models.
- opencv-python: OpenCV library for image and video processing (e.g., extracting video frames).
- mediapipe: Google's library for real-time keypoint and pose estimation (e.g., extracting pose data).
- numpy: Essential for numerical operations and handling keypoints as arrays.
- matplotlib: For visualizing pose keypoints, graphs, or training metrics.
- tqdm: Displays progress bars to monitor loops like model training or data preprocessing.
- twilio: Allows sending SMS or email notifications for alerts or updates (e.g., fall detection results or training completion).
"""
```

# YOLO Model Setup

## Function

```
from ultralytics import YOLO
import cv2
import os

# Initialize YOLO model
model = YOLO('yolov8n.pt')

def detect_bounding_boxes(video_path, output_folder):
    """
    Detects bounding boxes of people in a video and saves cropped frames.
    Args:
        video_path (str): Path to the input video.
        output_folder (str): Folder to save cropped frames.
    """
    cap = cv2.VideoCapture(video_path)
    frame_count = 0

    if not os.path.exists(output_folder):
        os.makedirs(output_folder)

    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break

        # Perform detection
        results = model(frame)
        for result in results:
            boxes = result.boxes.xyxy.cpu().numpy()
            confs = result.boxes.conf.cpu().numpy()
            classes = result.boxes.cls.cpu().numpy()

            # Save frames with bounding boxes
            for box, conf, cls in zip(boxes, confs, classes):
                if int(cls) == 0 and conf > 0.5:  # Class 0 is for 'person'
                    x1, y1, x2, y2 = map(int, box)
                    cropped_person = frame[y1:y2, x1:x2]
                    cv2.imwrite(f"{output_folder}/frame_{frame_count}.jpg", cropped_person)

        frame_count += 1

    cap.release()
    print(f"Bounding boxes detected and saved in {output_folder}")
```

## Execution

```
from google.colab import files

# Upload video
uploaded = files.upload()
video_path = list(uploaded.keys())[0]
output_folder = '/content/cropped_frames'

# Detect bounding boxes and save frames
detect_bounding_boxes(video_path, output_folder)

print(video_path)
```

# Pose Keypoint Extraction

## Function

```
import mediapipe as mp
import numpy as np
import cv2
import os

def extract_keypoints_and_labels(image_folder, output_keypoints_path, output_labels_path):
    """
    Extracts pose keypoints and labels from images.
    Args:
        image_folder (str): Path to the folder containing images.
        output_keypoints_path (str): Path to save the keypoints numpy file.
        output_labels_path (str): Path to save the labels numpy file.
    """
    mp_pose = mp.solutions.pose
    pose = mp_pose.Pose(static_image_mode=True)
    keypoints_list = []
    labels_list = []

    for image_name in os.listdir(image_folder):
        if image_name.endswith('.jpg') or image_name.endswith('.png'):
            image_path = os.path.join(image_folder, image_name)
            label = 1 if 'fallen' in image_name.lower() else 0

            image = cv2.imread(image_path)
            image_rgb = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
            results = pose.process(image_rgb)

            if results.pose_landmarks:
                keypoints = [
                    [lm.x, lm.y, lm.z]
                    for lm in results.pose_landmarks.landmark
                ]
            else:
                # If no pose is detected, add zeros
                keypoints = [[0, 0, 0] for _ in range(33)]

            keypoints_list.append(keypoints)
            labels_list.append(label)

    pose.close()

    # Save keypoints and labels as numpy arrays
    keypoints_array = np.array(keypoints_list)
    labels_array = np.array(labels_list)
    np.save(output_keypoints_path, keypoints_array)
    np.save(output_labels_path, labels_array)
    print(f"Keypoints saved to {output_keypoints_path}")
    print(f"Labels saved to {output_labels_path}")
```

## Execution

```
# Mount Drive to load dataset
from google.colab import drive
drive.mount('/content/drive')

# Paths to your datasets
train_images_path = '/content/drive/MyDrive/STGCN_Dataset/images/train'
validation_images_path = '/content/drive/MyDrive/STGCN_Dataset/images/validation'

# Output paths for keypoints and labels
train_keypoints_path = '/content/drive/MyDrive/STGCN_Dataset/train_keypoints.npy'
train_labels_path = '/content/drive/MyDrive/STGCN_Dataset/train_labels.npy'
validation_keypoints_path = '/content/drive/MyDrive/STGCN_Dataset/validation_keypoints.npy'
validation_labels_path = '/content/drive/MyDrive/STGCN_Dataset/validation_labels.npy'

# Extract keypoints and labels for train and validation datasets
extract_keypoints_and_labels(train_images_path, train_keypoints_path, train_labels_path)
extract_keypoints_and_labels(validation_images_path, validation_keypoints_path, validation_labels_path)

# Verify train keypoints and labels
train_keypoints = np.load(train_keypoints_path)
train_labels = np.load(train_labels_path)
print("Train Keypoints Shape:", train_keypoints.shape)
print("Train Labels Shape:", train_labels.shape)

# Verify validation keypoints and labels
validation_keypoints = np.load(validation_keypoints_path)
validation_labels = np.load(validation_labels_path)
print("Validation Keypoints Shape:", validation_keypoints.shape)
print("Validation Labels Shape:", validation_labels.shape)
```

# STGCN Model Definition

```
import torch
import torch.nn as nn
from torch.utils.data import Dataset, DataLoader

# Define the STGCN layer
class STGCNLayer(nn.Module):
    """
    A single Spatio-Temporal Graph Convolutional Network (ST-GCN) layer.
    This layer performs spatial and temporal convolutions on the input data.

    Args:
        in_channels (int): Number of input channels (features per keypoint).
        out_channels (int): Number of output channels (features).
        kernel_size_spatial (int): Kernel size for the spatial convolution.
        kernel_size_temporal (int): Kernel size for the temporal convolution.
    """
    def __init__(self, in_channels, out_channels, kernel_size_spatial=25, kernel_size_temporal=3):
        super(STGCNLayer, self).__init__()
        self.spatial_conv = nn.Conv2d(in_channels, out_channels, kernel_size=(1, kernel_size_spatial), padding=(0, kernel_size_spatial // 2))
        self.temporal_conv = nn.Conv2d(out_channels, out_channels, kernel_size=(kernel_size_temporal, 1), padding=(kernel_size_temporal // 2))
        self.bn = nn.BatchNorm2d(out_channels)

    def forward(self, x):
        """
        Forward pass through the STGCN layer.

        Args:
            x (torch.Tensor): Input tensor with shape (batch_size, in_channels, num_frames, num_keypoints).

        Returns:
            torch.Tensor: Output tensor after spatial and temporal convolutions.
        """
        x = self.spatial_conv(x)  # Spatial convolution
        x = torch.relu(x)
        x = self.temporal_conv(x)  # Temporal convolution
        x = self.bn(torch.relu(x))  # Batch normalization
        return x

# Define the full STGCN model
class STGCN(nn.Module):
    """
    The full Spatio-Temporal Graph Convolutional Network (ST-GCN) model.

    Args:
        num_keypoints (int): Number of keypoints in the pose data.
        num_classes (int): Number of classes for the classification task.
    """
    def __init__(self, num_keypoints, num_classes):
        super(STGCN, self).__init__()
        self.stgcn1 = STGCNLayer(1, 64, num_keypoints)  # Change input channels to 1
        self.stgcn2 = STGCNLayer(64, 64, num_keypoints)
        self.fc = nn.Linear(64, num_classes)

    def forward(self, x):
        """
        Forward pass through the STGCN model.

        Args:
            x (torch.Tensor): Input tensor with shape (batch_size, num_keypoints, num_frames).

        Returns:
            torch.Tensor: Output tensor containing class scores for each input sample.
        """
        if x.dim() == 3:
            x = x.unsqueeze(1)  # Shape becomes (batch_size, 1, num_keypoints, num_frames)

        x = x.permute(0, 1, 3, 2)  # Reorder to (batch_size, in_channels, num_frames, num_keypoints)
        x = self.stgcn1(x)
        x = self.stgcn2(x)
        x = x.mean(dim=[2, 3])  # Global average pooling (across frames and keypoints)
        return self.fc(x)


# Dataset class
class PoseDataset(Dataset):
    """
    A custom PyTorch Dataset for handling pose keypoints and labels.

    Args:
        keypoints (list or numpy.ndarray): List or array containing pose keypoints
                                           with shape (N, num_keypoints, 3).
        labels (list or numpy.ndarray): List or array containing class labels.
    """
    def __init__(self, keypoints, labels):
        self.keypoints = torch.tensor(keypoints, dtype=torch.float32)  # Shape: (N, num_keypoints, 3)
        self.labels = torch.tensor(labels, dtype=torch.long)

    def __len__(self):
        """
        Returns the number of samples in the dataset.

        Returns:
            int: Total number of samples.
        """
        return len(self.labels)

    def __getitem__(self, idx):
        """
        Retrieves the keypoints and label for a given index.

        Args:
            idx (int): Index of the sample to retrieve.

        Returns:
            tuple: A tuple containing the keypoints tensor and the label.
        """
        return self.keypoints[idx], self.labels[idx]
```

# Training the STGCN Model

## Function

```
from tqdm import tqdm

def train_stgcn(train_keypoints, train_labels, val_keypoints, val_labels, model_path, num_epochs=10, batch_size=16):
    """
    Trains the ST-GCN model for pose classification and validates its performance.

    Args:
        train_keypoints (list or numpy.ndarray): Training keypoints data with shape (num_samples, num_frames, num_keypoints).
        train_labels (list or numpy.ndarray): Training labels corresponding to the keypoints.
        val_keypoints (list or numpy.ndarray): Validation keypoints data with shape (num_samples, num_frames, num_keypoints).
        val_labels (list or numpy.ndarray): Validation labels corresponding to the keypoints.
        model_path (str): Path to save the trained model.
        num_epochs (int): Number of training epochs. Default is 10.
        batch_size (int): Batch size for training and validation. Default is 16.
    """
    # Define the device
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

    # Convert inputs to tensors, if not already done
    train_keypoints = torch.tensor(train_keypoints, dtype=torch.float32).to(device)
    train_labels = torch.tensor(train_labels, dtype=torch.int64).to(device)
    val_keypoints = torch.tensor(val_keypoints, dtype=torch.float32).to(device)
    val_labels = torch.tensor(val_labels, dtype=torch.int64).to(device)

    # Create DataLoader
    train_dataset = torch.utils.data.TensorDataset(train_keypoints, train_labels)
    val_dataset = torch.utils.data.TensorDataset(val_keypoints, val_labels)

    train_loader = torch.utils.data.DataLoader(train_dataset, batch_size=batch_size, shuffle=True)
    val_loader = torch.utils.data.DataLoader(val_dataset, batch_size=batch_size, shuffle=False)

    # Initialize the model with correct input channels and output classes
    model = STGCN(num_keypoints=train_keypoints.shape[2], num_classes=2)  # Assuming 33 keypoints and 2 classes
    model = model.to(device)

    # Define Loss and Optimizer
    criterion = nn.CrossEntropyLoss()
    optimizer = torch.optim.Adam(model.parameters(), lr=0.001)

    for epoch in range(num_epochs):
        model.train()
        running_loss = 0.0
        for keypoints, labels in tqdm(train_loader, desc=f"Training Epoch {epoch+1}"):
            keypoints, labels = keypoints.to(device), labels.to(device)
            optimizer.zero_grad()

            # Forward pass
            outputs = model(keypoints)
            loss = criterion(outputs, labels)

            # Backward pass and optimization
            loss.backward()
            optimizer.step()

            running_loss += loss.item()

        avg_train_loss = running_loss / len(train_loader)
        print(f"Epoch [{epoch+1}/{num_epochs}], Loss: {avg_train_loss:.4f}")

        # Validation step
        model.eval()
        correct = 0
        total = 0
        with torch.no_grad():
            for keypoints, labels in val_loader:
                keypoints, labels = keypoints.to(device), labels.to(device)
                outputs = model(keypoints)
                _, predicted = torch.max(outputs, 1)
                total += labels.size(0)
                correct += (predicted == labels).sum().item()

        accuracy = 100 * correct / total
        print(f"Validation Accuracy: {accuracy:.2f}%")

    # Save the trained model
    torch.save(model.state_dict(), model_path)
```

## Execution

```
# Load dataset
train_keypoints = np.load('/content/drive/MyDrive/STGCN_Dataset/train_keypoints.npy')
train_labels = np.load('/content/drive/MyDrive/STGCN_Dataset/train_labels.npy')

validation_keypoints = np.load('/content/drive/MyDrive/STGCN_Dataset/validation_keypoints.npy')
validation_labels = np.load('/content/drive/MyDrive/STGCN_Dataset/validation_labels.npy')

# Reshape the keypoints for STGCN
train_keypoints = train_keypoints.transpose(0, 2, 1)  # Shape: (332, 33, 3)
validation_keypoints = validation_keypoints.transpose(0, 2, 1)  # Shape: (86, 33, 3)

# Ensure the shapes are correct
print(f'Train Keypoints shape: {train_keypoints.shape}')
print(f'Train Labels shape: {train_labels.shape}')
print(f'Validation Keypoints shape: {validation_keypoints.shape}')
print(f'Validation Labels shape: {validation_labels.shape}')

# Train STGCN model
model_path = '/content/drive/MyDrive/STGCN_Dataset/stgcn_model.pth'

#train_stgcn(train_keypoints, train_labels, validation_keypoints, validation_labels, model_path, num_epochs=10, batch_size=16)

import torch

# Convert labels from numpy arrays to PyTorch tensors and ensure they are integers
train_labels = torch.tensor(train_labels).long()
val_labels = torch.tensor(validation_labels).long()

print(f"Train Keypoints shape: {train_keypoints.shape}, dtype: {train_keypoints.dtype}")
print(f"Train Labels shape: {train_labels.shape}, dtype: {train_labels.dtype}")
print(f"Validation Keypoints shape: {validation_keypoints.shape}, dtype: {validation_keypoints.dtype}")
print(f"Validation Labels shape: {val_labels.shape}, dtype: {val_labels.dtype}")

model_path = '/content/drive/MyDrive/STGCN_Dataset/stgcn_model.pth'

train_stgcn(train_keypoints, train_labels, validation_keypoints, val_labels, model_path, num_epochs=10, batch_size=16)

model_path = '/content/drive/MyDrive/STGCN_Dataset/stgcn_model.pth'
torch.save(model.state_dict(), model_path)
```

# Prediction

## Function

```
import torch
import cv2
import numpy as np

def predict_fall_in_video(video_path, model, device, pose_extractor, frame_skip=50):
    """
    Predict whether a fall occurs in a given video using a trained ST-GCN model.

    Parameters:
        video_path (str): Path to the input video.
        model (torch.nn.Module): Trained ST-GCN model.
        device (torch.device): Device to run the model on ('cuda' or 'cpu').
        pose_extractor (function): Function to extract pose keypoints from video frames.
        frame_skip (int): Number of frames to skip during processing to reduce RAM usage.

    Returns:
        dict: Dictionary with per-frame predictions and overall result.
    """
    model.eval()  # Set model to evaluation mode

    # Step 1: Extract frames from the video
    cap = cv2.VideoCapture(video_path)
    frames = []
    frame_count = 0
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        if frame_count % frame_skip == 0:  # Process every `frame_skip` frame
            frames.append(frame)
        frame_count += 1
    cap.release()

    # Step 2: Extract pose keypoints from the selected frames
    keypoints_sequence = []
    for frame in frames:
        keypoints = pose_extractor(frame)  # Extract keypoints for the frame
        if keypoints is not None:
            keypoints_sequence.append(keypoints)
        else:
            keypoints_sequence.append(np.zeros((33, 4)))  # Handle missing frames

    # Step 3: Convert keypoints to tensor and reshape for ST-GCN
    keypoints_sequence = np.array(keypoints_sequence)  # Shape: (T, J, C)
    keypoints_sequence = np.expand_dims(keypoints_sequence, axis=0)  # Add batch dimension

    # Optionally collapse the keypoints to a single channel, e.g., by averaging x, y, z, visibility
    keypoints_sequence = keypoints_sequence.mean(axis=-1, keepdims=True)  # Average across the last axis (x, y, z, visibility)

    # Convert to tensor and move to the appropriate device
    keypoints_tensor = torch.tensor(keypoints_sequence, dtype=torch.float32).to(device)

    # Ensure the tensor is of the correct shape: [batch_size, channels, height, width]
    keypoints_tensor = keypoints_tensor.permute(0, 3, 1, 2)  # Reorder to [batch_size, channels, height, width]

    # Step 4: Pass keypoints through the ST-GCN model
    with torch.no_grad():
        outputs = model(keypoints_tensor)  # Model inference
        predictions = torch.argmax(outputs, dim=1).cpu().numpy()  # Get class predictions

    # Step 5: Analyze predictions
    has_fallen = 1 in predictions  # Check if any frame predicts 'fallen' (class 1)
    frame_results = {f"Frame {i * frame_skip + 1}": "Fallen" if pred == 1 else "Not Fallen"
                     for i, pred in enumerate(predictions)}

    # Return results
    return {
        "Frame Results": frame_results,
        "Overall Result": "Fall Detected" if has_fallen else "No Fall Detected"
    }
```

## Execution

### Setting up GPU Acceleration

```
import torch
if torch.cuda.is_available():
    device = torch.device("cuda")
    print("Using GPU:", torch.cuda.get_device_name(0))
else:
    device = torch.device("cpu")
    print("Using CPU")
```

### Initializing the Model
```
# Initialize your model
model = STGCN(num_keypoints=len(keypoints_list), num_classes=2)  # Assuming 33 keypoints and 2 classes  # Use your actual model class name here
#device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model.to(device)

# Load the saved state_dict
model_path = '/content/drive/MyDrive/STGCN_Dataset/stgcn_model.pth'
model.load_state_dict(torch.load(model_path, map_location=device), strict=False)


# Set the model to evaluation mode
model.eval()
```

### Fall Detected Case
```
from google.colab import files

# Upload video
uploaded = files.upload()
video_path = list(uploaded.keys())[0]

import cv2
import mediapipe as mp
import numpy as np

# Open video using OpenCV
cap = cv2.VideoCapture(video_path)

# List to store frames
frames = []

while cap.isOpened():
    ret, frame = cap.read()
    if not ret:
        break
    # Optionally preprocess (resize, etc.)
    frame_resized = cv2.resize(frame, (640, 480))  # Resize to 640x480 for faster processing
    frames.append(frame_resized)

cap.release()

# Initialize MediaPipe pose model
mp_pose = mp.solutions.pose
pose = mp_pose.Pose()

# List to store keypoints for each frame
keypoints_list = []

for frame in frames:
    # Convert frame to RGB (MediaPipe works with RGB frames)
    frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)

    # Process the frame to extract pose landmarks
    results = pose.process(frame_rgb)

    # Extract keypoints if detected
    if results.pose_landmarks:
        keypoints = []
        for landmark in results.pose_landmarks.landmark:
            keypoints.append([landmark.x, landmark.y, landmark.z])
        keypoints_list.append(keypoints)
    else:
        keypoints_list.append(None)  # If no pose is detected, append None

pose.close()

# Print the number of frames and some example keypoints for debugging
print(f"Total frames: {len(frames)}")
print(f"First frame keypoints (if available): {keypoints_list[0]}")

import torch
import cv2
import numpy as np
import mediapipe as mp

# Define pose extraction function using MediaPipe
def pose_extractor(frame):
    mp_pose = mp.solutions.pose
    pose = mp_pose.Pose()

    # Convert frame to RGB for MediaPipe
    frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
    results = pose.process(frame_rgb)

    # Extract keypoints
    if results.pose_landmarks:
        keypoints = np.array([[landmark.x, landmark.y, landmark.z, landmark.visibility]
                              for landmark in results.pose_landmarks.landmark])
        return keypoints
    else:
        return None  # No keypoints detected

# Call the prediction function with video path and model
video_path = '/content/person_falling_new.mp4'  # Replace with your video path
fall_detected_result = predict_fall_in_video(video_path, model, device, pose_extractor)

# Print the frame results and overall result
print("Per-frame results:", fall_detected_results['Frame Results'])
print("Overall result:", fall_detected_results['Overall Result'])
```

### No Fall Detected Case
```
import cv2
import mediapipe as mp
import numpy as np

# Path to your uploaded video
video_path = '/content/person_walking.mp4'  # Replace with your video path

# Open video using OpenCV
cap = cv2.VideoCapture(video_path)

# List to store frames
frames = []

while cap.isOpened():
    ret, frame = cap.read()
    if not ret:
        break
    # Optionally preprocess (resize, etc.)
    frame_resized = cv2.resize(frame, (640, 480))  # Resize to 640x480 for faster processing
    frames.append(frame_resized)

cap.release()

# Initialize MediaPipe pose model
mp_pose = mp.solutions.pose
pose = mp_pose.Pose()

# List to store keypoints for each frame
keypoints_list = []

for frame in frames:
    # Convert frame to RGB (MediaPipe works with RGB frames)
    frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)

    # Process the frame to extract pose landmarks
    results = pose.process(frame_rgb)

    # Extract keypoints if detected
    if results.pose_landmarks:
        keypoints = []
        for landmark in results.pose_landmarks.landmark:
            keypoints.append([landmark.x, landmark.y, landmark.z])
        keypoints_list.append(keypoints)
    else:
        keypoints_list.append(None)  # If no pose is detected, append None

pose.close()

# Print the number of frames and some example keypoints for debugging
print(f"Total frames: {len(frames)}")
print(f"First frame keypoints (if available): {keypoints_list[0]}")

import torch
import cv2
import numpy as np
import mediapipe as mp

# Define pose extraction function using MediaPipe
def pose_extractor2(frame):
    mp_pose = mp.solutions.pose
    pose = mp_pose.Pose()

    # Convert frame to RGB for MediaPipe
    frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
    results = pose.process(frame_rgb)

    # Extract keypoints
    if results.pose_landmarks:
        keypoints = np.array([[landmark.x, landmark.y, landmark.z, landmark.visibility]
                              for landmark in results.pose_landmarks.landmark])
        return keypoints
    else:
        return None  # No keypoints detected

# Call the prediction function with video path and model
video_path2 = '/content/person_walking.mp4'  # Replace with your video path
no_fall_detected_results = predict_fall_in_video(video_path2, model, device, pose_extractor2)

# Print the frame results and overall result
print("Per-frame results:", no_fall_detected_results['Frame Results'])
print("Overall result:", no_fall_detected_results['Overall Result'])
```

# Twilio Integration with G-Drive

## Function
```
from google.colab import drive
drive.mount('/content/drive')

import cv2
import os

def save_to_gdrive_and_get_link(frame, file_name, gdrive_folder='MyDrive/Fall_Detection_Alerts'):
    """
    Saves the frame to Google Drive and retrieves a shareable link.

    Parameters:
        frame (numpy.ndarray): Image frame to save.
        file_name (str): Name for the saved file.
        gdrive_folder (str): Folder in Google Drive where the file will be saved.

    Returns:
        str: Google Drive shareable link for the saved file.
    """
    # Ensure Google Drive folder exists
    folder_path = f"/content/drive/{gdrive_folder}"
    os.makedirs(folder_path, exist_ok=True)

    # Save frame to Google Drive
    file_path = os.path.join(folder_path, file_name)
    cv2.imwrite(file_path, frame)
    print(f"Frame saved to: {file_path}")

    # Return the relative path in Google Drive for easier access
    relative_file_path = os.path.join(gdrive_folder, file_name)
    print(f"Manual location: {relative_file_path}")

    return relative_file_path  # Return relative path within MyDrive

def send_fall_notification_via_gdrive_simple(prediction_results, video_path, account_sid, auth_token, from_number, to_number):
    from twilio.rest import Client

    # Extract the overall result
    overall_result = prediction_results["Overall Result"]

    # Initialize Twilio client
    client = Client(account_sid, auth_token)

    if overall_result == "Fall Detected":
        # Extract a key frame from the video
        cap = cv2.VideoCapture(video_path)
        fallen_frame_index = None
        for i, (frame_key, frame_result) in enumerate(prediction_results["Frame Results"].items()):
            if frame_result == "Fallen":
                fallen_frame_index = i
                break

        # Extract and save the fallen frame
        if fallen_frame_index is not None:
            cap.set(cv2.CAP_PROP_POS_FRAMES, fallen_frame_index)
            ret, frame = cap.read()
            if ret:
                # Save to Google Drive
                fallen_frame_path = save_to_gdrive_and_get_link(frame, "fallen_frame.jpg")
                # Send MMS with a manual location
                message = client.messages.create(
                    body=f"Fall detected! Immediate assistance may be needed. Check Google Drive for the alert frame: {fallen_frame_path}",
                    from_=from_number,
                    to=to_number
                )
                print(f"Fall alert sent via SMS: SID {message.sid}")
            else:
                print("Failed to extract fallen frame.")
        else:
            print("No specific fallen frame detected in video.")
        cap.release()
    else:
        # Send a simple SMS indicating no fall was detected
        message = client.messages.create(
            body="Patient is okay. No fall detected.",
            from_=from_number,
            to=to_number
        )
        print(f"No fall detected notification sent: SID {message.sid}")
```
## Twilio Credentials

```
# Twilio account details (replace with your actual credentials)
account_sid = "twilio-sid"
auth_token = "twilio-auth-token"
from_number = "twilio-sender-client-number"
to_number = "reciver-number"
```

## Message Sending using Twilio

```
# Fall Detected Case
print("Testing Fall Detected Case:")
send_fall_notification_via_gdrive_simple(
    fall_detected_results,
    video_path,
    account_sid,
    auth_token,
    from_number,
    to_number
)

# No Fall Detected Case
print("\nTesting No Fall Detected Case:")
send_fall_notification_via_gdrive_simple(
    no_fall_detected_results,
    video_path,
    account_sid,
    auth_token,
    from_number,
    to_number
)
```
