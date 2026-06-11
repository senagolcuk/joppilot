# jöppli — Remote Monitoring & Driving Platform

## Architecture Report

**Version:** 1.1  
**Date:** 2026-03-29  
**Status:** Approved — ready to build  
**Target simulator:** AWSIM on Ubuntu 24.04 / ROS 2 Jazzy

---

## 1. Overview

jöppli is a remote monitoring and driving interface for autonomous vehicles. An operator logs into a web UI, sees a live camera feed from the vehicle, and sends real-time driving commands — steering, throttle, gear, indicators, and emergency stop.

The system consists of three machines:

- **AWSIM PC** — runs the simulator and the Greengrass v2 edge runtime, which hosts the jöppli components. Connects outbound to AWS services. No open ports.
- **EC2 instance** — serves the web application and vends temporary AWS credentials to operators. Does not touch video or commands.
- **Operator browser** — loads the React app from EC2, then connects directly to AWS services for video and commands.

Video and commands travel through separate AWS managed services. EC2 is deliberately kept out of both data paths.

---

## 2. System Architecture

### 2.1 Three data planes

The system separates into three independent planes, each with its own transport:

**Video plane (KVS WebRTC):** The AWSIM PC encodes camera frames as H.264 via GStreamer and sends them through Amazon Kinesis Video Streams WebRTC. The AWSIM PC is the WebRTC *master*, the browser is the *viewer*. After ICE negotiation through KVS signaling, video flows peer-to-peer. EC2 never touches a frame.

**Command plane (IoT Core MQTT via Greengrass):** The browser publishes driving commands to AWS IoT Core over MQTT-over-WebSocket. The Greengrass MQTT bridge on the AWSIM PC receives these messages and routes them to the jöppli client via Greengrass local IPC. The jöppli client translates them to ROS 2 messages. EC2 never touches a command.

**Auth plane (Cognito via EC2):** The browser logs in through the EC2 auth API, which validates credentials against Cognito and returns temporary, scoped AWS credentials for KVS and IoT Core access.

### 2.2 Design principles

- AWSIM PC initiates all connections outbound. No exposed ports, no VPN.
- EC2 is a static file server and credential vending machine. Nothing more.
- Video and commands are decoupled. If one drops, the other continues.
- All AWS credentials on the browser are temporary and scoped to the minimum required permissions.
- Greengrass manages all credential rotation, MQTT connectivity, component lifecycle, logging, and OTA updates on the AWSIM PC. The jöppli application code never manages AWS credentials or MQTT connections directly.

---

## 3. Machine 1: AWSIM PC

### 3.1 Greengrass v2 edge runtime

The AWSIM PC runs the AWS IoT Greengrass v2 edge runtime. Greengrass provisions its own X.509 device certificate during initial setup and manages all connectivity to AWS. The jöppli application code is packaged as Greengrass components and has no direct AWS dependencies.

Greengrass provides:

- **Component lifecycle management** — install, start, stop, restart, crash recovery for all jöppli components.
- **MQTT bridge** — routes messages between IoT Core and local Greengrass IPC. The jöppli client never opens its own MQTT connection.
- **Credential provider** — vends temporary STS credentials to components that need to call AWS APIs (e.g., KVS). No long-lived secrets in application code.
- **Log manager** — ships component logs to CloudWatch automatically.
- **Device shadow sync** — keeps vehicle state synchronized between the AWSIM PC and the cloud.
- **OTA deployments** — update jöppli components remotely by creating a Greengrass deployment. Greengrass handles download, stop-old, install-new, rollback-on-failure.

### 3.2 Greengrass components on the AWSIM PC

| Component | Type | Purpose |
|---|---|---|
| `com.joppli.camera-bridge` | Custom | ROS 2 node: subscribes to camera topics, feeds GStreamer, runs KVS producer |
| `com.joppli.command-bridge` | Custom | ROS 2 node: receives commands via Greengrass IPC, publishes to ROS 2 control topics |
| `com.joppli.status-reporter` | Custom | ROS 2 node: subscribes to vehicle state, publishes status via Greengrass IPC |
| `aws.greengrass.clientdevices.mqtt.Bridge` | Built-in | Routes MQTT messages between IoT Core and Greengrass local IPC |
| `aws.greengrass.LogManager` | Built-in | Ships component logs to CloudWatch |
| `aws.greengrass.ShadowManager` | Built-in | Syncs device shadow (vehicle state) with IoT Core |
| `aws.greengrass.TokenExchangeService` | Built-in | Vends temporary AWS credentials to custom components |

Note: the three custom components (`camera-bridge`, `command-bridge`, `status-reporter`) may be consolidated into a single component if coupling them simplifies deployment. The separation is logical, not necessarily physical.

### 3.3 Video pipeline (camera-bridge component)

The camera-bridge subscribes to the AWSIM camera topics and feeds frames into a GStreamer encoding pipeline:

```
ROS 2 subscriber
  └─ /sensing/camera/traffic_light/image_raw (sensor_msgs/msg/Image)
  └─ /sensing/camera/traffic_light/camera_info (sensor_msgs/msg/CameraInfo)
       │
       ▼
GStreamer pipeline
  appsrc (raw BGR frames)
  → videoconvert
  → x264enc (tune=zerolatency, speed-preset=ultrafast, bitrate=2000)
  → KVS WebRTC sink (master on channel "joppli-vehicle-001")
```

The KVS Producer SDK authenticates using temporary credentials obtained from the Greengrass Token Exchange Service — no credentials in the component code.

**Parameters:**

| Setting | Value |
|---|---|
| Frame rate | 30 fps |
| Resolution | Native from AWSIM (typically 1280x720) |
| Codec | H.264 (software x264, or vaapih264enc if GPU available) |
| Encoding preset | ultrafast / zerolatency |
| Target bitrate | 2000 kbps (adjustable) |

### 3.4 Command reception (command-bridge component)

The command-bridge receives messages from the Greengrass local IPC (routed from IoT Core by the MQTT bridge) and publishes them to ROS 2:

| Greengrass IPC topic (subscribe) | ROS 2 topic (publish) | Message type |
|---|---|---|
| `joppli/v001/cmd/control` | `/control/command/control_cmd` | `autoware_control_msgs/msg/Control` |
| `joppli/v001/cmd/emergency` | `/control/command/emergency_cmd` | `tier4_vehicle_msgs/msg/VehicleEmergencyStamped` |
| `joppli/v001/cmd/gear` | `/control/command/gear_cmd` | `autoware_vehicle_msgs/msg/GearCommand` |
| `joppli/v001/cmd/hazard_lights` | `/control/command/hazard_lights_cmd` | `autoware_vehicle_msgs/msg/HazardLightsCommand` |
| `joppli/v001/cmd/turn_indicators` | `/control/command/turn_indicators_cmd` | `autoware_vehicle_msgs/msg/TurnIndicatorsCommand` |

The component uses the Greengrass IPC SDK (`awsiot.greengrasscoreipc`) to subscribe — not a direct MQTT client.

### 3.5 Vehicle status (status-reporter component)

The status-reporter subscribes to vehicle state from the ROS 2 graph and publishes a consolidated status message via Greengrass IPC. The MQTT bridge forwards it to IoT Core.

| ROS 2 source | Greengrass IPC topic (publish) | Content |
|---|---|---|
| Various Autoware status topics | `joppli/v001/status` | `{ gear, speed, steering_angle, indicators, hazard_lights, emergency, timestamp }` |

Additionally, the status-reporter updates the Greengrass device shadow with the latest vehicle state. This allows a browser connecting mid-session to get an immediate snapshot via the IoT device shadow API rather than waiting for the next telemetry publish.

### 3.6 MQTT bridge configuration

The built-in Greengrass MQTT bridge is configured to route:

| Source | Direction | Destination |
|---|---|---|
| IoT Core `joppli/v001/cmd/#` | Cloud → Local | Greengrass local pub/sub |
| Greengrass local `joppli/v001/status` | Local → Cloud | IoT Core `joppli/v001/status` |

This configuration is defined in the Greengrass deployment and can be updated remotely without touching the AWSIM PC.

### 3.7 Authentication and credentials

Greengrass provisions its own X.509 certificate during initial setup (`greengrass-cli` or fleet provisioning). This certificate authenticates the Greengrass core device with IoT Core. Custom components never see this certificate.

For AWS API access (e.g., KVS Producer SDK), components request temporary credentials from the Greengrass Token Exchange Service. This service assumes an IAM role (the Greengrass Token Exchange Role) and returns scoped STS credentials. Credentials rotate automatically.

### 3.8 Dependencies on the AWSIM PC

- AWS IoT Greengrass v2 runtime
- Docker (optional, if components are containerized)
- Python 3, `rclpy`, `cv_bridge`
- GStreamer 1.0 with plugins: base, good, ugly, x264
- Amazon Kinesis Video Streams Producer SDK (C++ with Python bindings)
- Greengrass IPC SDK (`awsiot.greengrasscoreipc`)

---

## 4. Machine 2: EC2 Instance

### 4.1 What runs here

Two processes, both lightweight:

1. **Nginx** — serves the static React app, terminates TLS, reverse-proxies `/api/*`
2. **Auth API** — validates operator credentials and vends temporary AWS credentials

### 4.2 Auth API

A single endpoint that matters:

```
POST /api/auth/login
  Request:  { email, password }
  Response: { jwt, aws_credentials: { access_key_id, secret_access_key, session_token, expiration } }
```

**Internal flow:**

1. Validate credentials against Cognito user pool
2. Receive Cognito JWT tokens (ID token, access token, refresh token)
3. Exchange Cognito ID token with Cognito identity pool for an identity ID
4. Call STS via the identity pool to get temporary AWS credentials
5. Return JWT + temporary credentials to the browser

The temporary credentials are scoped to an IAM role that permits:

- `kinesisvideo:ConnectAsViewer` on `arn:aws:kinesisvideo:*:*:channel/joppli-vehicle-001/*`
- `kinesisvideo:GetSignalingChannelEndpoint` on the same channel
- `kinesisvideo:GetIceServerConfig` on the same channel
- `iot:Connect` with client ID `joppli-operator-*`
- `iot:Subscribe` on `joppli/v001/*`
- `iot:Publish` on `joppli/v001/cmd/*`
- `iot:Receive` on `joppli/v001/*`

### 4.3 Nginx configuration

- Serves static React build from `/var/www/joppli/`
- Reverse-proxies `/api/*` to the auth API (localhost:3000)
- TLS terminated with ACM certificate via Route 53 DNS
- HTTPS only, HTTP redirects to HTTPS

### 4.4 Instance sizing

`t3.micro` (2 vCPU, 1 GB RAM). EC2 does no video processing and no message relay. Nginx serving a static bundle + a small auth API is well within capacity.

### 4.5 Dependencies

- Nginx
- Node.js + Express (or Python + FastAPI) for the auth API
- AWS SDK for Cognito and STS calls
- Certbot or ACM for TLS

---

## 5. Machine 3: Operator Browser

### 5.1 React app components

The app has four concerns:

**Auth module** — login form, credential storage (in-memory only, not localStorage), automatic credential refresh before expiry using the JWT refresh token.

**Video viewer** — uses `amazon-kinesis-video-streams-webrtc` JS SDK. Connects as a WebRTC *viewer* to the KVS signaling channel. Renders the received `MediaStream` in a native `<video>` element with hardware-accelerated H.264 decoding.

**Command publisher** — uses `aws-iot-device-sdk-js-v2` (or `mqtt.js`). Connects to IoT Core over MQTT-over-WebSocket using the temporary credentials. Publishes commands when the operator interacts with the UI.

**Status display** — subscribes to `joppli/v001/status` via MQTT to show current vehicle state (gear, speed, indicators). Can also read the device shadow for an immediate snapshot on connect.

### 5.2 UI controls and MQTT mapping

| UI element | Interaction | MQTT topic | Payload | Rate |
|---|---|---|---|---|
| WASD keys / on-screen joystick | Continuous while held | `joppli/v001/cmd/control` | `{ steering_angle: float, acceleration: float }` | 20 Hz |
| Gear selector (P/R/N/D) | On change | `joppli/v001/cmd/gear` | `{ gear: "park"\|"reverse"\|"neutral"\|"drive" }` | On event |
| Turn indicator buttons (L/R/off) | On click | `joppli/v001/cmd/turn_indicators` | `{ command: "left"\|"right"\|"none" }` | On event |
| Hazard lights toggle | On click | `joppli/v001/cmd/hazard_lights` | `{ command: "on"\|"off" }` | On event |
| Emergency stop button | On click | `joppli/v001/cmd/emergency` | `{ emergency: true }` | On event |

### 5.3 Dependencies

- React + TypeScript
- `amazon-kinesis-video-streams-webrtc` (KVS WebRTC JS SDK)
- `aws-iot-device-sdk-js-v2` (IoT Core MQTT-over-WS)
- Tailwind CSS (styling)

---

## 6. AWS Resources

### 6.1 Resource inventory

| Resource | Type | Purpose |
|---|---|---|
| `joppli-vehicle-001` | KVS signaling channel | WebRTC signaling for camera stream |
| `joppli-vehicle-001` | IoT Core thing | AWSIM PC / Greengrass core device identity |
| `joppli-vehicle-001` | Greengrass core device | Registered Greengrass v2 core |
| `joppli-gg-deployment` | Greengrass deployment | Defines which components run on the AWSIM PC |
| `joppli-vehicle-001-cert` | IoT Core X.509 certificate | Greengrass core device authentication |
| `joppli-vehicle-policy` | IoT Core policy | Scoped MQTT permissions for the Greengrass core |
| `JoppliGreengrassTokenExchangeRole` | IAM role | Greengrass token exchange: KVS master + S3 artifact access |
| `JoppliGreengrassTokenExchangeAlias` | IoT role alias | Maps X.509 cert to the token exchange IAM role |
| `joppli-component-artifacts` | S3 bucket | Stores Greengrass component artifacts for deployment |
| `joppli-operators` | Cognito user pool | Operator login (email/password + optional MFA) |
| `joppli-identity-pool` | Cognito identity pool | Maps authenticated operators to IAM role |
| `joppli-operator-role` | IAM role | KVS viewer + IoT Core for browser operators |
| `joppli.yourdomain.com` | Route 53 hosted zone | DNS for the web app |
| `joppli.yourdomain.com` | ACM certificate | TLS for the EC2 instance |
| `joppli-ec2` | EC2 `t3.micro` | Nginx + auth API |

### 6.2 IoT Core policy (Greengrass core device)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iot:Connect",
        "iot:Publish",
        "iot:Subscribe",
        "iot:Receive",
        "greengrass:*"
      ],
      "Resource": "*"
    }
  ]
}
```

Note: Greengrass core devices require broad IoT permissions for component deployments, shadow sync, and telemetry. Fine-grained scoping is applied at the component level via Greengrass authorization policies, not the IoT Core policy.

### 6.3 Greengrass Token Exchange Role (IAM)

This role is assumed by the Greengrass Token Exchange Service to vend temporary credentials to custom components:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kinesisvideo:ConnectAsMaster",
        "kinesisvideo:GetSignalingChannelEndpoint",
        "kinesisvideo:CreateSignalingChannel",
        "kinesisvideo:GetIceServerConfig",
        "kinesisvideo:DescribeSignalingChannel"
      ],
      "Resource": "arn:aws:kinesisvideo:REGION:ACCOUNT:channel/joppli-vehicle-001/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::joppli-component-artifacts/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "arn:aws:logs:REGION:ACCOUNT:log-group:/greengrass/*"
    }
  ]
}
```

### 6.4 IAM policy (operator — attached to Cognito identity pool role)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kinesisvideo:ConnectAsViewer",
        "kinesisvideo:GetSignalingChannelEndpoint",
        "kinesisvideo:GetIceServerConfig"
      ],
      "Resource": "arn:aws:kinesisvideo:REGION:ACCOUNT:channel/joppli-vehicle-001/*"
    },
    {
      "Effect": "Allow",
      "Action": "iot:Connect",
      "Resource": "arn:aws:iot:REGION:ACCOUNT:client/joppli-operator-*"
    },
    {
      "Effect": "Allow",
      "Action": "iot:Subscribe",
      "Resource": [
        "arn:aws:iot:REGION:ACCOUNT:topicfilter/joppli/v001/cmd/*",
        "arn:aws:iot:REGION:ACCOUNT:topicfilter/joppli/v001/status"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "iot:Publish",
      "Resource": "arn:aws:iot:REGION:ACCOUNT:topic/joppli/v001/cmd/*"
    },
    {
      "Effect": "Allow",
      "Action": "iot:Receive",
      "Resource": "arn:aws:iot:REGION:ACCOUNT:topic/joppli/v001/*"
    }
  ]
}
```

---

## 7. Auth Flow

The complete authentication sequence from browser to active driving session:

1. Operator navigates to `https://joppli.yourdomain.com` — Nginx serves the React app.
2. Operator enters email + password — React app calls `POST /api/auth/login` on EC2.
3. EC2 auth API validates credentials against Cognito user pool.
4. Cognito returns JWT tokens (ID, access, refresh).
5. EC2 auth API exchanges the Cognito ID token with the identity pool to get an identity ID.
6. EC2 auth API calls STS via the identity pool to get temporary AWS credentials scoped to the operator IAM role.
7. EC2 returns the JWT + temporary AWS credentials to the browser.
8. Browser uses temporary credentials to connect to KVS as a WebRTC viewer — video starts streaming.
9. Browser uses the same temporary credentials to connect to IoT Core over MQTT-over-WebSocket — command channel opens.
10. Operator is now driving. Video flows P2P via KVS, commands flow via IoT Core through the Greengrass MQTT bridge.
11. Before temporary credentials expire, the browser calls the auth API with the refresh token to get new credentials.

---

## 8. MQTT Topic Structure

```
joppli/
  v001/                          # Vehicle ID namespace
    cmd/                         # Browser → IoT Core → GG MQTT bridge → GG IPC → jöppli client
      control                    # Steering + acceleration (20 Hz)
      emergency                  # Emergency stop flag
      gear                       # Gear shift request
      hazard_lights              # Hazard light toggle
      turn_indicators            # Turn indicator command
    status                       # jöppli client → GG IPC → GG MQTT bridge → IoT Core → Browser
```

All topics are scoped under the vehicle ID (`v001`). This supports multi-vehicle operation in the future by adding `v002`, `v003`, etc. with separate IoT policies per operator.

---

## 9. Greengrass Deployment Configuration

The Greengrass deployment defines which components run on the AWSIM PC and how the MQTT bridge is configured.

### 9.1 MQTT bridge configuration (merge key)

```json
{
  "mqttTopicMapping": {
    "commands-cloud-to-local": {
      "topic": "joppli/v001/cmd/#",
      "source": "IotCore",
      "target": "Pubsub"
    },
    "status-local-to-cloud": {
      "topic": "joppli/v001/status",
      "source": "Pubsub",
      "target": "IotCore"
    }
  }
}
```

### 9.2 Component recipes

Each custom component is defined by a YAML recipe specifying:

- **Artifacts** — S3 URI to the component archive (Python code + dependencies)
- **Lifecycle** — install, run, and shutdown commands
- **Dependencies** — other components required (e.g., Token Exchange Service)
- **Configuration** — parameters like KVS channel name, ROS 2 topic names, frame rate

Recipes and artifacts are stored in the `joppli-component-artifacts` S3 bucket. Updating a component is: upload new artifact to S3, create new Greengrass deployment, Greengrass handles the rest.

---

## 10. Performance Targets

| Metric | Target |
|---|---|
| Video frame rate | 30 fps |
| Video codec | H.264 (x264enc ultrafast/zerolatency) |
| Video bitrate | ~2000 kbps |
| Glass-to-glass video latency | < 200 ms (P2P via KVS WebRTC) |
| Command publish rate | 20 Hz (50 ms interval) |
| Command latency (browser → ROS 2) | < 100 ms (MQTT-over-WS → IoT Core → GG MQTT bridge → GG IPC → jöppli client → ROS 2) |
| Credential refresh (browser) | Before expiry (typically 1 hour STS tokens) |
| Credential refresh (AWSIM PC) | Automatic via Greengrass Token Exchange Service |

---

## 11. Technology Stack

| Component | Technology |
|---|---|
| Simulator | AWSIM (Ubuntu 24.04, ROS 2 Jazzy) |
| Edge runtime | AWS IoT Greengrass v2 |
| jöppli components | Python 3, rclpy, GStreamer, KVS Producer SDK, Greengrass IPC SDK |
| EC2 server | Nginx, Node.js + Express (auth API) |
| Frontend | React, TypeScript, Tailwind CSS |
| Video transport | Amazon Kinesis Video Streams (WebRTC) |
| Command transport | AWS IoT Core (MQTT) via Greengrass MQTT bridge |
| Auth | Amazon Cognito (user pool + identity pool) |
| DNS + TLS | Route 53 + ACM |
| Monitoring | CloudWatch (auto-shipped by Greengrass Log Manager) |
| EC2 instance | t3.micro (Amazon Linux 2023) |

---

## 12. Build Order

1. **AWS resource provisioning** — KVS channel, IoT Core thing, Cognito pools, IAM roles, S3 artifact bucket
2. **Greengrass setup** — install Greengrass v2 on the AWSIM PC, provision as core device, configure token exchange role alias
3. **camera-bridge component** — ROS 2 node with GStreamer pipeline + KVS WebRTC master. Deploy via Greengrass. Validate with KVS console viewer.
4. **EC2 setup** — Nginx + auth API. Deploy a minimal React app that can log in and receive credentials.
5. **Browser video viewer** — KVS WebRTC JS SDK integration. End-to-end camera feed in the browser.
6. **command-bridge component** — receives commands via Greengrass IPC, publishes to ROS 2. Configure MQTT bridge routing. Deploy via Greengrass.
7. **Browser command UI** — WASD/joystick, gear, indicators, e-stop. Wire to IoT Core MQTT.
8. **status-reporter component** — publishes vehicle state via Greengrass IPC + updates device shadow. Deploy via Greengrass.
9. **Browser status display** — subscribes to status topic, reads device shadow on connect.
10. **Polish** — connection status indicators, latency overlay, reconnection handling, credential auto-refresh, CloudWatch dashboards.

---

## 13. Future Considerations

- **Session recording:** Add a parallel KVS persistent stream for archival. Replay sessions from the KVS console or export to S3.
- **Multi-vehicle:** Add vehicle ID routing in MQTT topics and KVS channel naming. Cognito identity pool policies can scope operators to specific vehicles. Each vehicle is a separate Greengrass core device with its own deployment.
- **CloudWatch alarms:** Monitor KVS stream health, IoT Core message throughput, command latency, and connection drops.
- **Fleet provisioning:** Use Greengrass fleet provisioning to onboard new vehicles automatically with claim certificates, instead of manual per-device setup.
- **ML at the edge:** Deploy Greengrass ML inference components (e.g., object detection) alongside jöppli components on the vehicle.
