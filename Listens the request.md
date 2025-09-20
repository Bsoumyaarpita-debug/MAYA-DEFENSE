import socket
import json

# HQ2 IP and port where HQ2 listens for motion detection requests
HQ2_IP = "192.168.31.148"#
HQ2_PORT = 6000          # Must match HQ2 listening port

def send_motion_alert_to_hq2(device_id, gps_coords, motion_status):
    message = {
        "device_id": device_id,
        "gps": gps_coords,
        "motion": motion_status  # True or False
    }
    json_message = json.dumps(message).encode('utf-8')

    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.connect((HQ2_IP, HQ2_PORT))
            s.sendall(json_message)
        print(f"Motion alert sent to HQ2: {message}")
    except Exception as e:
        print(f"Failed to send motion alert to HQ2: {e}")

# Example usage
if __name__ == "__main__":
    device_id = "FieldNode001"
    gps_coords = {"lat": 12.34, "lon": 56.78}
    motion_status = True  # PIR sensor detected motion

    send_motion_alert_to_hq2(device_id, gps_coords, motion_status)
