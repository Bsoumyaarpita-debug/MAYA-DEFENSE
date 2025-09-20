import numpy as np
from PIL import Image
import io
import tensorflow as tf
from sklearn.ensemble import IsolationForest

# --- TensorFlow Image Classification ---

# Load or define a simple pretrained TensorFlow model (for example)
model = tf.keras.applications.mobilenet_v2.MobileNetV2(weights='imagenet', include_top=True)

def preprocess_image(image_bytes):
    # Convert raw bytes to PIL image
    image = Image.open(io.BytesIO(image_bytes)).convert('RGB').resize((224, 224))
    # Preprocess for MobileNetV2
    img_array = tf.keras.preprocessing.image.img_to_array(image)
    img_array = tf.keras.applications.mobilenet_v2.preprocess_input(img_array)
    return np.expand_dims(img_array, axis=0)

def classify_image(image_bytes):
    preprocessed = preprocess_image(image_bytes)
    preds = model.predict(preprocessed)
    decoded = tf.keras.applications.mobilenet_v2.decode_predictions(preds, top=3)[0]
    # Return top label
    return decoded[0][1], decoded[0][2]  # label, confidence

# --- Isolation Forest Anomaly Detection ---

# Example synthetic sensor telemetry data for training
train_data = np.array([
    [20, 0.5, 100], [21, 0.45, 105], [19, 0.55, 98],  # normal
    [20.5, 0.52, 102], [20, 0.51, 101]
])

# Train Isolation Forest model
iso_forest = IsolationForest(contamination=0.1, random_state=42)
iso_forest.fit(train_data)

def detect_anomaly(sensor_features):
    # sensor_features: numpy array like [temperature, humidity, light]
    pred = iso_forest.predict([sensor_features])
    # prediction: 1 = normal, -1 = anomaly
    return pred[0] == -1

# --- Example Usage ---

# Simulated image bytes (replace with actual image bytes from ESP32-CAM)
with open('example_snapshot.jpg', 'rb') as f:
    img_bytes = f.read()

label, confidence = classify_image(img_bytes)
print(f"Image classified as: {label} (confidence {confidence:.2f})")

# Simulated sensor data for anomaly detection
sensor_sample = np.array([22, 0.6, 110])
if detect_anomaly(sensor_sample):
    print("Sensor data anomaly detected!")
else:
    print("Sensor data normal.")
