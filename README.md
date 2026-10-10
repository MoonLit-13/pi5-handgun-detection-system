# Raspberry Pi 5-Based CCTV Handgun Detection and Security Alert System

A thesis project focused on real-time handgun detection from CCTV network streams using **YOLOv8n** and **Raspberry Pi 5**, with a web-based security dashboard, multi-camera incident monitoring, and human-confirmed SMS alerts.

## Overview

This project aims to develop an AI-assisted CCTV monitoring system that detects suspected handguns in video streams and notifies security personnel for verification.

When a potential handgun is detected, the system generates an alert, highlights the detected object with a bounding box, and preserves a timestamped evidence frame. An authorized security officer reviews the incident before the system can send an SMS notification to selected recipients.

The system is initially designed for **2–4 IP CCTV cameras** and aims to help security personnel review a person's possible movement between monitored areas through a chronological incident timeline.

> **Important:** An AI detection is a potential alert, not confirmation that a firearm or security threat is present. SMS notifications about an incident require authorized human confirmation.

## Objectives

- Detect suspected handguns in CCTV video streams using YOLOv8n.
- Process RTSP camera feeds on a Raspberry Pi 5.
- Display detection alerts and bounding boxes through a web dashboard.
- Save timestamped evidence and maintain an incident history.
- Support monitoring of multiple cameras.
- Organize camera observations into a chronological incident timeline.
- Require authorized human confirmation before incident SMS notifications.
- Evaluate detection accuracy, processing speed, latency, and hardware resource usage.

## Technology Stack

| Technology | Purpose |
|---|---|
| Raspberry Pi 5 | Main edge-processing hardware |
| Ubuntu Server LTS (ARM64) | Operating system |
| Python 3 | Core application and processing logic |
| Ultralytics YOLOv8n | Handgun object detection |
| OpenCV | RTSP stream acquisition, frame processing, and bounding-box rendering |
| RTSP | Video stream transport from IP cameras or an NVR |
| FastAPI | Backend API and web application server |
| WebSockets | Real-time alerts and system-status updates |
| HTML, CSS, JavaScript | Browser-based security dashboard |
| SQLite | Local database for events, decisions, and SMS records |
| PySerial | Serial communication with a compatible GSM/LTE modem |
| AT commands | Modem control and SMS submission |
| ByteTrack (optional) | Tracking objects across frames within an individual camera |
| Person re-identification (optional) | Experimental cross-camera person association |

The selected versions and hardware compatibility will be verified during implementation and testing.

## System Architecture

```text
       IP CCTV Cameras / NVR
          RTSP Video Feeds
                  |
                  v
       +----------------------+
       | OpenCV RTSP Readers  |
       | One reader per camera|
       +----------------------+
                  |
                  v
       +----------------------+
       | Latest-Frame Buffers |
       | One buffer per camera|
       +----------------------+
                  |
                  v
       +----------------------+
       | Inference Scheduler  |
       |      YOLOv8n         |
       +----------------------+
                  |
                  v
       +----------------------+
       | Detection Event      |
       | Timestamp, camera ID,|
       | confidence, evidence |
       +----------------------+
                  |
                  v
       +----------------------+
       | FastAPI Backend      |
       | Event Management     |
       +----------------------+
                  |
          WebSockets / API
                  |
                  v
       +----------------------+
       | Security Dashboard   |
       | Alert and Evidence   |
       | Officer Review       |
       +----------------------+
                  |
                  v
       +----------------------+
       | Authorized Human     |
       | Confirmation         |
       +----------------------+
                  |
            Confirmed only
                  |
                  v
       +----------------------+
       | Python SMS Service   |
       | PySerial + AT Commands|
       +----------------------+
                  |
                  v
          GSM/LTE Modem
                  |
                  v
         Selected Recipients

       SQLite: Event and Audit Records
```

## How It Works

1. **Video acquisition:** OpenCV connects to each configured camera through RTSP and continuously reads video frames.
2. **Frame management:** Each camera has its own background reader and latest-frame buffer. New frames replace older unprocessed frames to reduce unnecessary processing delay.
3. **Handgun detection:** An inference scheduler selects recent frames and passes them to YOLOv8n.
4. **Event generation:** When a detection meets the configured threshold, the system records a potential incident with the camera identifier, timestamp, confidence score, and bounding-box coordinates.
5. **Evidence preservation:** The system saves the specific frame associated with the detection rather than relying on the continuously overwritten latest-frame buffer.
6. **Dashboard alert:** The browser dashboard displays the alert, affected camera, timestamp, evidence image, and red bounding box.
7. **Officer review:** Authorized security personnel review the evidence and confirm or reject the incident.
8. **SMS notification:** After confirmation, the backend submits an SMS to the configured recipients through a compatible GSM/LTE modem.
9. **Audit logging:** The system records detection events, officer decisions, and available SMS submission or delivery status.

## Multi-Camera Processing

The proposed system supports an initial target of 2–4 cameras.

Each camera uses an independent RTSP reader and latest-frame buffer. A shared inference scheduler can then select frames fairly across cameras, avoiding the assumption that the Raspberry Pi must run a separate simultaneous inference worker for every camera.

The scheduler should support:

- Fair frame selection across cameras.
- Camera-specific timestamps and connection status.
- Detection of stale frames.
- RTSP timeout handling and reconnection.
- Evidence capture for each detection event.
- Monitoring of inference latency, FPS, CPU, and RAM.

A latest-frame buffer reduces frame backlog but does not increase the model's inference capacity. Actual performance will be measured on the target hardware.

## Security Dashboard

The proposed dashboard will run in a browser on a separate security PC, while the Raspberry Pi operates headlessly as the processing unit.

### Debug Mode

Designed for development and testing.

- Camera preview and bounding-box display.
- Detection confidence and inference FPS.
- CPU and RAM usage.
- RTSP connection status and error logs.
- Configurable detection thresholds and sampling rates.

### Operation Mode

Designed for routine security monitoring.

- Camera status and multi-camera overview.
- Detection alerts and timestamped evidence.
- Chronological incident timeline.
- Officer confirmation and rejection controls.
- SMS recipient selection or approved recipient groups.
- SMS submission status and event history.

FastAPI provides the backend, while WebSockets deliver event and status updates to the dashboard. The detection service should continue operating independently of the browser session.

For the initial prototype, annotated video may be displayed using MJPEG. An alternative is to transmit video separately and draw detection boxes in the browser using synchronized detection metadata.

## Multi-Camera Incident Timeline

The system aims to help officers review observations across connected camera locations.

The first implementation will correlate events using:

- Camera identifiers and configured locations.
- Event timestamps.
- Saved evidence frames.
- Known paths between monitored areas.
- Optional within-camera person tracking.

ByteTrack may be used to track objects across successive frames within a single camera. Its tracking identifiers are not consistent across different cameras.

Cross-camera person re-identification may be investigated as an optional extension. Such associations are uncertain and must be presented as possible matches for human review, not verified identity.

## SMS Notification Workflow

SMS functionality will be implemented as a separate backend component.

1. A detection creates a pending incident.
2. The dashboard presents the incident for review.
3. An authorized officer confirms the incident.
4. The backend validates the authorization and approved recipients.
5. Python communicates with the compatible GSM/LTE modem using PySerial and AT commands.
6. The system records the modem response and any delivery report supported by the modem and network.

Detection alone must never trigger an incident SMS. Duplicate submissions, modem failures, disconnections, and pending incidents must be handled explicitly.

A successful modem submission does not necessarily mean the recipient received the message.

## Database and Event Logging

SQLite will be used for lightweight local data storage.

Proposed records include:

| Record | Example data |
|---|---|
| Camera | Camera ID, name, location, connection status |
| Detection event | Event ID, timestamp, camera ID, confidence, evidence path |
| Person observation | Camera ID, timestamp, local tracking ID |
| Officer review | Event ID, decision, reviewer, decision timestamp |
| SMS record | Event ID, recipients, submission status, modem response |
| System health | CPU, RAM, FPS, stream errors, reconnection events |

Surveillance evidence, camera credentials, and incident records should be protected through access controls, secure configuration, and appropriate retention policies.

## Evaluation and Testing

The system will be evaluated using both object-detection metrics and system-performance measurements.

| Metric | Purpose |
|---|---|
| Precision | Measures the proportion of detections that are correct |
| Recall | Measures how many actual target instances are detected |
| F1-score | Balances precision and recall |
| mAP@50 | Measures detection performance at an IoU threshold of 0.50 |
| FPS per camera | Measures effective detection throughput |
| Inference latency | Measures the time required to process a frame |
| Alert latency | Measures the delay between detection and dashboard notification |
| SMS submission latency | Measures the time from confirmation to modem submission result |
| CPU and RAM usage | Measures resource consumption on the Raspberry Pi |
| Stream recovery | Evaluates recovery from camera or network failures |
| False-alert rate | Measures incorrect detection events in the test scenarios |

Testing will begin with one camera and gradually scale to two, three, and four cameras. Different sampling rates, lighting conditions, viewing angles, and occlusion scenarios will be evaluated using appropriately labeled data and controlled trials.

Training and evaluation data will be kept separate. Performance results will be reported from actual experiments rather than assumed in advance.

## Implementation Roadmap

- [ ] Set up Raspberry Pi 5 and Ubuntu Server LTS.
- [ ] Establish RTSP connectivity with one CCTV camera.
- [ ] Integrate YOLOv8n and display detections.
- [ ] Save timestamped detection evidence.
- [ ] Implement one reader and latest-frame buffer per camera.
- [ ] Add fair multi-camera inference scheduling and reconnection handling.
- [ ] Build the FastAPI backend and browser dashboard.
- [ ] Implement Debug Mode and Operation Mode.
- [ ] Add SQLite event storage and officer review controls.
- [ ] Integrate a compatible GSM/LTE modem for confirmation-gated SMS.
- [ ] Build the multi-camera incident timeline.
- [ ] Benchmark performance across 1–4 cameras.
- [ ] Document test results, limitations, and future improvements.

## Limitations

- Detection accuracy depends on the training data, image quality, lighting, camera angle, and occlusion.
- Real-time performance on Raspberry Pi 5 must be established through testing.
- Multiple cameras increase processing and network demands.
- Within-camera tracking does not automatically provide cross-camera identity matching.
- Person re-identification can produce incorrect associations.
- SMS submission and delivery depend on modem compatibility and mobile-network availability.
- Human review is required before an incident is treated as confirmed.

## License

This project is intended for academic research and educational development. The repository license should be selected and added in accordance with the project's intended reuse permissions and the licensing requirements of its dependencies.

## Disclaimer

This system is an assistive monitoring prototype. It is not a substitute for professional security procedures, human judgment, or independent verification. Detection outputs should be reviewed before action is taken.

---

**Project status:** In development — proposed architecture and features are subject to implementation, compatibility checks, and experimental validation.
