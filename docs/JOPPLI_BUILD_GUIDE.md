# jöppli — Step-by-Step Build Guide

**Starting point:** Ubuntu 24.04.3 LTS, ROS 2 Jazzy, Cyclone DDS configured, EC2 instance created.

**What you'll build:** A working remote monitoring & driving platform — live camera from AWSIM in a browser, with real-time steering/throttle/gear commands flowing back.

---

## Phase 0: Install AWSIM & Local Dependencies (AWSIM PC)

You have Ubuntu + ROS 2 but haven't installed AWSIM yet. Do this first — everything else depends on having a running simulator.

### 0.1 Install AWSIM

```bash
# Download the latest AWSIM binary release
# Check https://github.com/autowarefoundation/AWSIM/releases for the latest
mkdir -p ~/awsim && cd ~/awsim
# Download the appropriate release for Ubuntu 24.04 / ROS 2 Jazzy
# Example (check for actual latest URL):
wget https://github.com/autowarefoundation/AWSIM/releases/download/<version>/AWSIM_<version>.zip
unzip AWSIM_<version>.zip

# Make the binary executable
chmod +x AWSIM.x86_64
```

**Verify AWSIM launches:**

```bash
# Source ROS 2
source /opt/ros/jazzy/setup.bash

# Launch AWSIM
./AWSIM.x86_64
```

You should see the Unity simulator window. Check that ROS 2 topics are publishing:

```bash
# In another terminal
source /opt/ros/jazzy/setup.bash
ros2 topic list
```

Look for `/sensing/camera/traffic_light/image_raw` — that's the camera topic the camera-bridge will subscribe to.

**Checkpoint:** AWSIM runs, camera topic is publishing.

### 0.2 Install system dependencies on the AWSIM PC

```bash
# Python + ROS 2 development packages
sudo apt update && sudo apt install -y \
  python3-pip python3-venv python3-colcon-common-extensions \
  python3-rosdep ros-jazzy-cv-bridge \
  curl unzip default-jre

# GStreamer (for video encoding)
sudo apt install -y \
  gstreamer1.0-tools \
  gstreamer1.0-plugins-base \
  gstreamer1.0-plugins-good \
  gstreamer1.0-plugins-ugly \
  gstreamer1.0-x \
  libgstreamer1.0-dev \
  libgstreamer-plugins-base1.0-dev \
  gstreamer1.0-plugins-bad  # for x264enc

# Verify GStreamer
gst-inspect-1.0 x264enc
# Should print plugin details, not "No such element"

# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

# Configure AWS CLI with your IAM admin user credentials
aws configure
# Enter: Access Key ID, Secret Access Key, Region (e.g. eu-central-1), output format (json)
```

**Checkpoint:** `gst-inspect-1.0 x264enc` works, `aws sts get-caller-identity` returns your account.

---

## Phase 1: AWS Resource Provisioning

This is build step 1 from the architecture. You'll create every AWS resource the system needs before touching any application code.

Pick a single AWS region for everything and stick to it (e.g., `eu-central-1`). Replace `REGION` and `ACCOUNT` throughout with your actual values.

### 1.1 Create the S3 artifact bucket

```bash
aws s3 mb s3://joppli-component-artifacts --region REGION
```

### 1.2 Create the KVS signaling channel

```bash
aws kinesisvideo create-signaling-channel \
  --channel-name joppli-vehicle-001 \
  --channel-type SINGLE_MASTER \
  --region REGION
```

**Verify:**

```bash
aws kinesisvideo describe-signaling-channel \
  --channel-name joppli-vehicle-001 --region REGION
```

### 1.3 Create the IoT Core Thing

```bash
# Create the thing
aws iot create-thing --thing-name joppli-vehicle-001

# Create keys and certificate (save these!)
aws iot create-keys-and-certificate \
  --set-as-active \
  --certificate-pem-outfile joppli-vehicle-001.cert.pem \
  --public-key-outfile joppli-vehicle-001.public.key \
  --private-key-outfile joppli-vehicle-001.private.key

# Note the certificateArn from the output — you'll need it
# Example: arn:aws:iot:REGION:ACCOUNT:cert/XXXX
```

**Keep these files safe.** You'll copy them to the AWSIM PC for Greengrass.

### 1.4 Create and attach the IoT Core policy

```bash
# Create the policy
cat > joppli-vehicle-policy.json << 'EOF'
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
EOF

aws iot create-policy \
  --policy-name joppli-vehicle-policy \
  --policy-document file://joppli-vehicle-policy.json

# Attach policy to the certificate
aws iot attach-policy \
  --policy-name joppli-vehicle-policy \
  --target <certificateArn>

# Attach thing to certificate
aws iot attach-thing-principal \
  --thing-name joppli-vehicle-001 \
  --principal <certificateArn>
```

### 1.5 Create the Greengrass Token Exchange Role (IAM)

```bash
# Trust policy — allows IoT credentials provider to assume this role
cat > gg-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "credentials.iot.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name JoppliGreengrassTokenExchangeRole \
  --assume-role-policy-document file://gg-trust-policy.json

# Permissions policy
cat > gg-permissions-policy.json << 'EOF'
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
      "Action": "s3:GetObject",
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
EOF

aws iam put-role-policy \
  --role-name JoppliGreengrassTokenExchangeRole \
  --policy-name JoppliGGPermissions \
  --policy-document file://gg-permissions-policy.json
```

### 1.6 Create the IoT Role Alias

```bash
aws iot create-role-alias \
  --role-alias JoppliGreengrassTokenExchangeAlias \
  --role-arn arn:aws:iam::ACCOUNT:role/JoppliGreengrassTokenExchangeRole

# The role alias policy allows Greengrass to assume the role via the alias
cat > gg-role-alias-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:AssumeRoleWithCertificate",
      "Resource": "arn:aws:iot:REGION:ACCOUNT:rolealias/JoppliGreengrassTokenExchangeAlias"
    }
  ]
}
EOF

aws iot create-policy \
  --policy-name JoppliGreengrassTokenExchangeAliasPolicy \
  --policy-document file://gg-role-alias-policy.json

# Attach to the same certificate
aws iot attach-policy \
  --policy-name JoppliGreengrassTokenExchangeAliasPolicy \
  --target <certificateArn>
```

### 1.7 Create Cognito User Pool (operator login)

```bash
aws cognito-idp create-user-pool \
  --pool-name joppli-operators \
  --auto-verified-attributes email \
  --username-attributes email \
  --policies '{"PasswordPolicy":{"MinimumLength":8,"RequireUppercase":true,"RequireLowercase":true,"RequireNumbers":true,"RequireSymbols":false}}' \
  --region REGION

# Note the UserPoolId from the output (e.g., eu-central-1_XXXXXXXXX)

# Create app client (no secret — used from backend)
aws cognito-idp create-user-pool-client \
  --user-pool-id <UserPoolId> \
  --client-name joppli-web \
  --explicit-auth-flows ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH \
  --region REGION

# Note the ClientId from the output

# Create a test operator user
aws cognito-idp admin-create-user \
  --user-pool-id <UserPoolId> \
  --username operator@joppli.com \
  --temporary-password TempPass123! \
  --user-attributes Name=email,Value=operator@joppli.com Name=email_verified,Value=true \
  --region REGION
```

### 1.8 Create Cognito Identity Pool

```bash
aws cognito-identity create-identity-pool \
  --identity-pool-name joppli-identity-pool \
  --allow-unauthenticated-identities \
  --cognito-identity-providers \
    ProviderName=cognito-idp.REGION.amazonaws.com/<UserPoolId>,ClientId=<ClientId>,ServerSideTokenCheck=false \
  --region REGION

# Note the IdentityPoolId from the output (e.g., eu-central-1:xxxx-xxxx-xxxx)
```

### 1.9 Create the Operator IAM Role (for browser credentials)

```bash
# Trust policy — allows Cognito identity pool to assume
cat > operator-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "cognito-identity.amazonaws.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "cognito-identity.amazonaws.com:aud": "<IdentityPoolId>"
        },
        "ForAnyValue:StringLike": {
          "cognito-identity.amazonaws.com:amr": "authenticated"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name joppli-operator-role \
  --assume-role-policy-document file://operator-trust-policy.json

# Permissions
cat > operator-permissions.json << 'EOF'
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
EOF

aws iam put-role-policy \
  --role-name joppli-operator-role \
  --policy-name JoppliOperatorPermissions \
  --policy-document file://operator-permissions.json
```

### 1.10 Set Identity Pool Roles

```bash
aws cognito-identity set-identity-pool-roles \
  --identity-pool-id <IdentityPoolId> \
  --roles authenticated=arn:aws:iam::ACCOUNT:role/joppli-operator-role \
  --region REGION
```

**Checkpoint for Phase 1:** Run these verification commands:

```bash
aws kinesisvideo describe-signaling-channel --channel-name joppli-vehicle-001
aws iot describe-thing --thing-name joppli-vehicle-001
aws cognito-idp describe-user-pool --user-pool-id <UserPoolId>
aws s3 ls s3://joppli-component-artifacts
aws iam get-role --role-name JoppliGreengrassTokenExchangeRole
aws iam get-role --role-name joppli-operator-role
```

All should return valid responses. Save all the IDs/ARNs somewhere — you'll reference them constantly.

---

## Phase 2: Install & Provision Greengrass v2 (AWSIM PC)

### 2.1 Download and install Greengrass

```bash
# Download Greengrass v2 nucleus
curl -s https://d2s8p88vqu9w66.cloudfront.net/releases/greengrass-nucleus-latest.zip \
  -o greengrass-nucleus-latest.zip
unzip greengrass-nucleus-latest.zip -d GreengrassInstaller

# Get your IoT credentials endpoint
aws iot describe-endpoint --endpoint-type iot:CredProv --region REGION
# Note the endpoint (e.g., xxxx.credentials.iot.REGION.amazonaws.com)

aws iot describe-endpoint --endpoint-type iot:Data-ATS --region REGION
# Note the data endpoint (e.g., xxxx-ats.iot.REGION.amazonaws.com)

# Download Amazon Root CA
curl -o AmazonRootCA1.pem https://www.amazontrust.com/repository/AmazonRootCA1.pem
```

### 2.2 Copy certificates to the AWSIM PC

Make sure the three files from step 1.3 are on the AWSIM PC:
- `joppli-vehicle-001.cert.pem`
- `joppli-vehicle-001.private.key`
- `AmazonRootCA1.pem`

### 2.3 Run Greengrass installer

```bash
sudo -E java -Droot="/greengrass/v2" -Dlog.store=FILE \
  -jar ./GreengrassInstaller/lib/Greengrass.jar \
  --aws-region REGION \
  --root /greengrass/v2 \
  --thing-name joppli-vehicle-001 \
  --thing-policy-name joppli-vehicle-policy \
  --tes-role-name JoppliGreengrassTokenExchangeRole \
  --tes-role-alias-name JoppliGreengrassTokenExchangeAlias \
  --component-default-user ggc_user:ggc_group \
  --provision false \
  --setup-system-service true \
  --certificate-file-path ./joppli-vehicle-001.cert.pem \
  --private-key-path ./joppli-vehicle-001.private.key \
  --root-ca-path ./AmazonRootCA1.pem
```

Note: Since you already created the thing, cert, and policies manually in Phase 1, use `--provision false`. If you prefer auto-provisioning, use `--provision true` and skip steps 1.3–1.6, but that requires IAM admin creds on the AWSIM PC.

### 2.4 Verify Greengrass is running

```bash
sudo systemctl status greengrass

# Check component status
sudo /greengrass/v2/bin/greengrass-cli component list

# Check logs
sudo tail -f /greengrass/v2/logs/greengrass.log
```

You should see the Greengrass core connecting to IoT Core. Verify in the AWS Console under **IoT Core > Manage > Greengrass devices > Core devices** — `joppli-vehicle-001` should appear as "Healthy".

**Checkpoint:** Greengrass running, visible in AWS console as a healthy core device.

---

## Phase 3: Build & Deploy the Camera-Bridge Component

This is the first custom Greengrass component. It subscribes to AWSIM's camera topic and streams video via KVS WebRTC.

### 3.1 Build the KVS WebRTC Producer SDK (C++)

```bash
# Install build dependencies
sudo apt install -y cmake build-essential libssl-dev libcurl4-openssl-dev \
  liblog4cplus-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev \
  gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad \
  gstreamer1.0-plugins-ugly

# Clone and build KVS WebRTC SDK
cd ~
git clone --recursive https://github.com/awslabs/amazon-kinesis-video-streams-webrtc-sdk-c.git
cd amazon-kinesis-video-streams-webrtc-sdk-c
mkdir build && cd build
cmake .. -DBUILD_SAMPLE=ON
make -j$(nproc)

# The built libraries will be in build/
# Note the paths — your component will need them
```

### 3.2 Create the camera-bridge Python component

```bash
mkdir -p ~/joppli-components/camera-bridge
cd ~/joppli-components/camera-bridge
```

Create `camera_bridge.py`:

```python
#!/usr/bin/env python3
"""
com.joppli.camera-bridge
Subscribes to AWSIM camera topic, encodes H.264 via GStreamer,
sends via KVS WebRTC as master.
"""
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import Image
from cv_bridge import CvBridge
import gi
gi.require_version('Gst', '1.0')
from gi.repository import Gst, GLib
import threading
import json
import os

# Greengrass IPC for getting credentials
from awsiot.greengrasscoreipc.clientv2 import GreengrassCoreIPCClientV2

class CameraBridge(Node):
    def __init__(self):
        super().__init__('camera_bridge')
        self.bridge = CvBridge()

        # Initialize GStreamer
        Gst.init(None)

        # TODO: Build GStreamer pipeline with appsrc -> x264enc -> KVS WebRTC sink
        # This requires KVS Producer SDK Python bindings or subprocess approach

        # Subscribe to AWSIM camera topic
        self.subscription = self.create_subscription(
            Image,
            '/sensing/camera/traffic_light/image_raw',
            self.image_callback,
            10
        )
        self.get_logger().info('camera-bridge started')

    def image_callback(self, msg):
        # Convert ROS Image to OpenCV, push to GStreamer pipeline
        cv_image = self.bridge.imgmsg_to_cv2(msg, desired_encoding='bgr8')
        # Push frame to GStreamer appsrc
        # (implementation depends on your GStreamer pipeline setup)
        pass

def main():
    rclpy.init()
    node = CameraBridge()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### 3.3 Create the Greengrass component recipe

Create `recipe.yaml`:

```yaml
---
RecipeFormatVersion: "2020-01-25"
ComponentName: com.joppli.camera-bridge
ComponentVersion: "1.0.0"
ComponentDescription: "Streams AWSIM camera to KVS WebRTC"
ComponentPublisher: joppli
ComponentDependencies:
  aws.greengrass.TokenExchangeService:
    VersionRequirement: ">=2.0.0"
    DependencyType: HARD
ComponentConfiguration:
  DefaultConfiguration:
    kvs_channel_name: "joppli-vehicle-001"
    camera_topic: "/sensing/camera/traffic_light/image_raw"
    frame_rate: 30
    bitrate: 2000
Manifests:
  - Platform:
      os: linux
    Artifacts:
      - URI: "s3://joppli-component-artifacts/com.joppli.camera-bridge/1.0.0/camera-bridge.zip"
        Unarchive: ZIP
    Lifecycle:
      install: |
        pip3 install --user opencv-python-headless cv_bridge PyGObject
      run: |
        source /opt/ros/jazzy/setup.bash
        python3 {artifacts:decompressedPath}/camera-bridge/camera_bridge.py
      shutdown: |
        kill $!
```

### 3.4 Package and deploy

```bash
# Zip the component code
cd ~/joppli-components
zip -r camera-bridge.zip camera-bridge/

# Upload to S3
aws s3 cp camera-bridge.zip \
  s3://joppli-component-artifacts/com.joppli.camera-bridge/1.0.0/camera-bridge.zip

# Create the Greengrass deployment (from CLI or AWS Console)
# First deployment — include built-in components too
aws greengrassv2 create-deployment \
  --target-arn arn:aws:iot:REGION:ACCOUNT:thing/joppli-vehicle-001 \
  --deployment-name joppli-gg-deployment \
  --components '{
    "com.joppli.camera-bridge": {
      "componentVersion": "1.0.0"
    },
    "aws.greengrass.TokenExchangeService": {
      "componentVersion": "2.0.0"
    },
    "aws.greengrass.LogManager": {
      "componentVersion": "2.0.0",
      "configurationUpdate": {
        "merge": "{\"logsUploaderConfiguration\":{\"componentLogsConfigurationMap\":{\"com.joppli.camera-bridge\":{\"logFileRegex\":\"com.joppli.camera-bridge.*\\.log\",\"logFileDirectoryPath\":\"/greengrass/v2/logs\"}}}}"
      }
    }
  }' \
  --region REGION
```

### 3.5 Validate video streaming

After deployment, check the component is running:

```bash
sudo /greengrass/v2/bin/greengrass-cli component list
sudo tail -f /greengrass/v2/logs/com.joppli.camera-bridge.log
```

Then go to the **AWS Console > Kinesis Video Streams > Signaling channels > joppli-vehicle-001** and use the built-in media viewer to verify you see the AWSIM camera feed.

**Checkpoint:** Camera frames from AWSIM are visible in the KVS console viewer.

---

## Phase 4: EC2 Setup — Nginx + Auth API

### 4.1 SSH into your EC2 instance

```bash
ssh -i your-key.pem ec2-user@<EC2-PUBLIC-IP>
```

### 4.2 Install Nginx and Node.js

```bash
# Amazon Linux 2023
sudo dnf update -y
sudo dnf install -y nginx

# Node.js 20 LTS
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -
sudo dnf install -y nodejs

node --version  # Should be v20.x
```

### 4.3 Build the Auth API

```bash
mkdir -p ~/joppli-auth && cd ~/joppli-auth
npm init -y
npm install express cors @aws-sdk/client-cognito-identity-provider \
  @aws-sdk/client-cognito-identity @aws-sdk/client-sts dotenv
```

Create `.env`:

```bash
AWS_REGION=REGION
COGNITO_USER_POOL_ID=<UserPoolId>
COGNITO_CLIENT_ID=<ClientId>
COGNITO_IDENTITY_POOL_ID=<IdentityPoolId>
PORT=3000
```

Create `server.js`:

```javascript
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const {
  CognitoIdentityProviderClient,
  InitiateAuthCommand
} = require('@aws-sdk/client-cognito-identity-provider');
const {
  CognitoIdentityClient,
  GetIdCommand,
  GetCredentialsForIdentityCommand
} = require('@aws-sdk/client-cognito-identity');

const app = express();
app.use(cors());
app.use(express.json());

const cognitoIdp = new CognitoIdentityProviderClient({ region: process.env.AWS_REGION });
const cognitoIdentity = new CognitoIdentityClient({ region: process.env.AWS_REGION });

app.post('/api/auth/login', async (req, res) => {
  try {
    const { email, password } = req.body;

    // 1. Authenticate against Cognito User Pool
    const authResult = await cognitoIdp.send(new InitiateAuthCommand({
      AuthFlow: 'USER_PASSWORD_AUTH',
      ClientId: process.env.COGNITO_CLIENT_ID,
      AuthParameters: { USERNAME: email, PASSWORD: password }
    }));

    const idToken = authResult.AuthenticationResult.IdToken;
    const refreshToken = authResult.AuthenticationResult.RefreshToken;

    // 2. Get identity ID from Identity Pool
    const providerName = `cognito-idp.${process.env.AWS_REGION}.amazonaws.com/${process.env.COGNITO_USER_POOL_ID}`;
    const identityResult = await cognitoIdentity.send(new GetIdCommand({
      IdentityPoolId: process.env.COGNITO_IDENTITY_POOL_ID,
      Logins: { [providerName]: idToken }
    }));

    // 3. Get temporary AWS credentials
    const credentialsResult = await cognitoIdentity.send(new GetCredentialsForIdentityCommand({
      IdentityId: identityResult.IdentityId,
      Logins: { [providerName]: idToken }
    }));

    res.json({
      jwt: idToken,
      refresh_token: refreshToken,
      aws_credentials: {
        access_key_id: credentialsResult.Credentials.AccessKeyId,
        secret_access_key: credentialsResult.Credentials.SecretKey,
        session_token: credentialsResult.Credentials.SessionToken,
        expiration: credentialsResult.Credentials.Expiration.toISOString()
      }
    });
  } catch (err) {
    console.error('Login error:', err);
    res.status(401).json({ error: err.message });
  }
});

app.post('/api/auth/refresh', async (req, res) => {
  try {
    const { refresh_token } = req.body;
    const authResult = await cognitoIdp.send(new InitiateAuthCommand({
      AuthFlow: 'REFRESH_TOKEN_AUTH',
      ClientId: process.env.COGNITO_CLIENT_ID,
      AuthParameters: { REFRESH_TOKEN: refresh_token }
    }));

    const idToken = authResult.AuthenticationResult.IdToken;
    const providerName = `cognito-idp.${process.env.AWS_REGION}.amazonaws.com/${process.env.COGNITO_USER_POOL_ID}`;

    const identityResult = await cognitoIdentity.send(new GetIdCommand({
      IdentityPoolId: process.env.COGNITO_IDENTITY_POOL_ID,
      Logins: { [providerName]: idToken }
    }));

    const credentialsResult = await cognitoIdentity.send(new GetCredentialsForIdentityCommand({
      IdentityId: identityResult.IdentityId,
      Logins: { [providerName]: idToken }
    }));

    res.json({
      jwt: idToken,
      aws_credentials: {
        access_key_id: credentialsResult.Credentials.AccessKeyId,
        secret_access_key: credentialsResult.Credentials.SecretKey,
        session_token: credentialsResult.Credentials.SessionToken,
        expiration: credentialsResult.Credentials.Expiration.toISOString()
      }
    });
  } catch (err) {
    console.error('Refresh error:', err);
    res.status(401).json({ error: err.message });
  }
});

app.listen(process.env.PORT, () => {
  console.log(`Auth API running on port ${process.env.PORT}`);
});
```

### 4.4 Set up the Auth API as a systemd service

```bash
sudo tee /etc/systemd/system/joppli-auth.service << 'EOF'
[Unit]
Description=joppli Auth API
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/home/ec2-user/joppli-auth
ExecStart=/usr/bin/node server.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable joppli-auth
sudo systemctl start joppli-auth
sudo systemctl status joppli-auth
```

### 4.5 Configure Nginx

```bash
sudo tee /etc/nginx/conf.d/joppli.conf << 'EOF'
server {
    listen 80;
    server_name _;
    
    # For now, HTTP only. Add TLS later with ACM/Certbot.

    root /var/www/joppli;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
EOF

sudo mkdir -p /var/www/joppli
echo "<h1>joppli - coming soon</h1>" | sudo tee /var/www/joppli/index.html
sudo nginx -t
sudo systemctl restart nginx
```

### 4.6 Test the auth API

```bash
# From your local machine or the EC2 itself
curl -X POST http://<EC2-PUBLIC-IP>/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"operator@joppli.com","password":"<operator-password>"}'
```

You should get back a JSON response with `jwt` and `aws_credentials`.

**Checkpoint:** Login endpoint returns valid temporary AWS credentials.

---

## Phase 5: Browser Video Viewer (React App)

### 5.1 Scaffold the React app (on your development machine)

```bash
npx create-react-app joppli-web --template typescript
cd joppli-web
npm install tailwindcss @tailwindcss/vite
npm install amazon-kinesis-video-streams-webrtc
npm install aws-iot-device-sdk-v2  # for later phases
```

### 5.2 Build the video viewer component

The key integration is using the KVS WebRTC JS SDK to connect as a viewer:

```typescript
// src/services/videoService.ts
import { SignalingClient, Role } from 'amazon-kinesis-video-streams-webrtc';

export async function startViewer(
  credentials: { accessKeyId: string; secretAccessKey: string; sessionToken: string },
  region: string,
  channelName: string,
  videoElement: HTMLVideoElement
) {
  // 1. Get signaling channel endpoints
  // 2. Create SignalingClient as VIEWER
  // 3. Handle ICE candidates and SDP negotiation
  // 4. Attach remote stream to video element
  
  // See AWS KVS WebRTC JS SDK examples for full implementation:
  // https://github.com/awslabs/amazon-kinesis-video-streams-webrtc-sdk-js
}
```

### 5.3 Build and deploy to EC2

```bash
npm run build
# Copy build output to EC2
scp -i your-key.pem -r build/* ec2-user@<EC2-PUBLIC-IP>:/tmp/joppli-build/
# On EC2:
sudo cp -r /tmp/joppli-build/* /var/www/joppli/
```

**Checkpoint:** Open `http://<EC2-PUBLIC-IP>` in a browser, log in, and see the AWSIM camera feed streaming live.

---

## Phase 6: Command-Bridge Component (AWSIM PC)

### 6.1 Create the command-bridge component

```bash
mkdir -p ~/joppli-components/command-bridge
cd ~/joppli-components/command-bridge
```

Create `command_bridge.py`:

```python
#!/usr/bin/env python3
"""
com.joppli.command-bridge
Receives driving commands via Greengrass IPC, publishes to ROS 2.
"""
import rclpy
from rclpy.node import Node
import json
import threading
from awsiot.greengrasscoreipc.clientv2 import GreengrassCoreIPCClientV2

from autoware_control_msgs.msg import Control
from autoware_vehicle_msgs.msg import GearCommand, TurnIndicatorsCommand, HazardLightsCommand
from tier4_vehicle_msgs.msg import VehicleEmergencyStamped

class CommandBridge(Node):
    def __init__(self):
        super().__init__('command_bridge')
        
        # ROS 2 publishers
        self.control_pub = self.create_publisher(Control, '/control/command/control_cmd', 10)
        self.gear_pub = self.create_publisher(GearCommand, '/control/command/gear_cmd', 10)
        self.turn_pub = self.create_publisher(TurnIndicatorsCommand, '/control/command/turn_indicators_cmd', 10)
        self.hazard_pub = self.create_publisher(HazardLightsCommand, '/control/command/hazard_lights_cmd', 10)
        self.emergency_pub = self.create_publisher(VehicleEmergencyStamped, '/control/command/emergency_cmd', 10)
        
        # Greengrass IPC subscriptions
        self.ipc_client = GreengrassCoreIPCClientV2()
        
        # Subscribe to command topics from IoT Core (via MQTT bridge)
        topics = [
            'joppli/v001/cmd/control',
            'joppli/v001/cmd/gear',
            'joppli/v001/cmd/turn_indicators',
            'joppli/v001/cmd/hazard_lights',
            'joppli/v001/cmd/emergency'
        ]
        
        for topic in topics:
            self.ipc_client.subscribe_to_topic(
                topic=topic,
                on_stream_event=self._make_handler(topic)
            )
        
        self.get_logger().info('command-bridge started, subscribed to all command topics')
    
    def _make_handler(self, topic):
        def handler(event):
            payload = json.loads(event.message.payload.decode())
            if 'control' in topic:
                self._handle_control(payload)
            elif 'gear' in topic:
                self._handle_gear(payload)
            elif 'turn_indicators' in topic:
                self._handle_turn(payload)
            elif 'hazard_lights' in topic:
                self._handle_hazard(payload)
            elif 'emergency' in topic:
                self._handle_emergency(payload)
        return handler
    
    def _handle_control(self, payload):
        msg = Control()
        msg.steering_tire_angle = float(payload.get('steering_angle', 0.0))
        msg.longitudinal.acceleration = float(payload.get('acceleration', 0.0))
        self.control_pub.publish(msg)
    
    def _handle_gear(self, payload):
        msg = GearCommand()
        gear_map = {'park': 1, 'reverse': 2, 'neutral': 3, 'drive': 4}
        msg.command = gear_map.get(payload.get('gear', 'park'), 1)
        self.gear_pub.publish(msg)
    
    # Similar handlers for turn, hazard, emergency...

def main():
    rclpy.init()
    node = CommandBridge()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### 6.2 Update the Greengrass deployment to include MQTT bridge + command-bridge

```bash
aws greengrassv2 create-deployment \
  --target-arn arn:aws:iot:REGION:ACCOUNT:thing/joppli-vehicle-001 \
  --deployment-name joppli-gg-deployment-v2 \
  --components '{
    "com.joppli.camera-bridge": { "componentVersion": "1.0.0" },
    "com.joppli.command-bridge": { "componentVersion": "1.0.0" },
    "aws.greengrass.clientdevices.mqtt.Bridge": {
      "componentVersion": "2.0.0",
      "configurationUpdate": {
        "merge": "{\"mqttTopicMapping\":{\"commands-cloud-to-local\":{\"topic\":\"joppli/v001/cmd/#\",\"source\":\"IotCore\",\"target\":\"Pubsub\"},\"status-local-to-cloud\":{\"topic\":\"joppli/v001/status\",\"source\":\"Pubsub\",\"target\":\"IotCore\"}}}"
      }
    },
    "aws.greengrass.TokenExchangeService": { "componentVersion": "2.0.0" },
    "aws.greengrass.LogManager": { "componentVersion": "2.0.0" }
  }' \
  --region REGION
```

### 6.3 Test command flow

Use the AWS IoT Core MQTT test client (in the AWS Console) to publish a test command:

Topic: `joppli/v001/cmd/gear`
Payload: `{"gear": "drive"}`

Watch the command-bridge logs to confirm it received the message and published to ROS 2:

```bash
sudo tail -f /greengrass/v2/logs/com.joppli.command-bridge.log
```

And verify with `ros2 topic echo /control/command/gear_cmd`.

**Checkpoint:** Commands published from IoT Core MQTT test client arrive as ROS 2 messages.

---

## Phase 7: Browser Command UI

### 7.1 Add MQTT command publisher to the React app

Wire up WASD keys and UI buttons to publish MQTT messages via IoT Core. The key MQTT topics and payloads are:

- `joppli/v001/cmd/control` at 20 Hz: `{ steering_angle, acceleration }`
- `joppli/v001/cmd/gear` on change: `{ gear: "park"|"reverse"|"neutral"|"drive" }`
- `joppli/v001/cmd/turn_indicators` on click: `{ command: "left"|"right"|"none" }`
- `joppli/v001/cmd/hazard_lights` on click: `{ command: "on"|"off" }`
- `joppli/v001/cmd/emergency` on click: `{ emergency: true }`

### 7.2 Build, deploy, test

After building the React app and deploying to EC2, open the browser, log in, and try steering the vehicle in AWSIM using WASD keys.

**Checkpoint:** Keystrokes in the browser move the vehicle in AWSIM.

---

## Phase 8: Status-Reporter Component (AWSIM PC)

### 8.1 Create the status-reporter component

Subscribes to ROS 2 Autoware status topics and publishes a consolidated JSON to Greengrass IPC (which the MQTT bridge forwards to IoT Core).

Key ROS 2 topics to subscribe to:
- Vehicle speed, gear state, steering angle, indicator states

Publishes to Greengrass IPC topic: `joppli/v001/status`

Also updates the Greengrass device shadow with latest state (so browsers connecting mid-session get an immediate snapshot).

### 8.2 Deploy via Greengrass

Same pattern as before: package, upload to S3, update the Greengrass deployment to include `com.joppli.status-reporter`.

**Checkpoint:** Status messages appear in the IoT Core MQTT test client on topic `joppli/v001/status`.

---

## Phase 9: Browser Status Display

### 9.1 Subscribe to status in the React app

The browser subscribes to `joppli/v001/status` via the existing MQTT connection and displays: current gear, speed (km/h), steering angle, indicator states, and emergency status.

On initial connection, read the device shadow to get the last known state immediately.

### 9.2 Deploy

Build and deploy the React app to EC2 again.

**Checkpoint:** Browser shows live vehicle status alongside the camera feed.

---

## Phase 10: Polish

### 10.1 Connection status indicators

Show connection health for both the KVS video stream and the MQTT command channel in the browser UI.

### 10.2 Latency overlay

Display measured latency (command round-trip, video glass-to-glass) on the operator screen.

### 10.3 Credential auto-refresh

The browser should call `/api/auth/refresh` with the refresh token before the temporary credentials expire (typically after ~55 minutes of the 1-hour STS token).

### 10.4 Reconnection handling

If video or MQTT drops, automatically reconnect with backoff. Show a visual indicator during reconnection.

### 10.5 TLS/DNS (production)

Set up Route 53 for your domain, request an ACM certificate, and configure Nginx for HTTPS. Update the React app's API base URL to use the domain.

### 10.6 CloudWatch dashboards

Greengrass Log Manager already ships component logs. Create CloudWatch dashboards for: KVS stream health, IoT Core message throughput, command latency, connection drops.

---

## Quick Reference: All AWS Resource IDs to Track

Keep a file with these values — you'll reference them throughout:

```
AWS_REGION=
AWS_ACCOUNT_ID=
IOT_DATA_ENDPOINT=
IOT_CRED_ENDPOINT=
CERTIFICATE_ARN=
KVS_CHANNEL_NAME=joppli-vehicle-001
THING_NAME=joppli-vehicle-001
COGNITO_USER_POOL_ID=
COGNITO_CLIENT_ID=
COGNITO_IDENTITY_POOL_ID=
S3_ARTIFACT_BUCKET=joppli-component-artifacts
GG_TOKEN_EXCHANGE_ROLE_ARN=
OPERATOR_ROLE_ARN=
EC2_PUBLIC_IP=
```

---

## Performance Targets to Validate

| Metric | Target |
|---|---|
| Video frame rate | 30 fps |
| Video bitrate | ~2000 kbps |
| Glass-to-glass video latency | < 200 ms |
| Command publish rate | 20 Hz |
| Command latency (browser → ROS 2) | < 100 ms |
