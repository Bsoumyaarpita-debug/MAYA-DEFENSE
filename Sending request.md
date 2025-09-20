import socket
import json

# Configuration
LISTEN_IP = "192.168.31.148",
LISTEN_PORT = 5001      # Port to receive AI results from HQ2

def trigger_buzzer():
    # Add your buzzer triggering code here
    print("Buzzer triggered!")

def update_led_color(label):
    # Add your LED control code here based on label
    # Example: green for safe, yellow for suspicious, red for human
    colors = {
        "human": "red",
        "animal": "yellow",
        "suspicious": "yellow",
        "safe": "green"
    }
    color = colors.get(label, "green")
    print(f"LED set to {color}")

def start_hq1_ai_result_server():
    print(f"HQ1 listening for AI results on {LISTEN_IP}:{LISTEN_PORT}...")
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server_socket:
        server_socket.bind((LISTEN_IP, LISTEN_PORT))
        server_socket.listen()
        print("HQ1 ready to receive AI results.")

        while True:
            client_socket, addr = server_socket.accept()
            with client_socket:
                print(f"Connection from {addr}")
                data = client_socket.recv(2048)
                if not data:
                    continue
                try:
                    message = data.decode("utf-8")
                    result = json.loads(message)

                    device_id = result.get("device_id")
                    gps = result.get("gps")
                    label = result.get("label")
                    confidence = result.get("confidence")

                    print(f"AI Result received for {device_id} at {gps}: {label} ({confidence:.2%} confidence)")

                    # Trigger buzzer on motion detection
                    trigger_buzzer()

                    # Update LED color based on AI label
                    update_led_color(label)

                except Exception as e:
                    print(f"Error processing AI result: {e}")

if __name__ == "__main__":
    start_hq1_ai_result_server()
