# Google Pub/Sub POC - Complete Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Project Structure](#project-structure)
5. [Configuration Options](#configuration-options)
6. [Local Development Setup (Emulator)](#local-development-setup-emulator)
7. [Google Cloud Setup (Production)](#google-cloud-setup-production)
8. [API Endpoints](#api-endpoints)
9. [Testing Guide](#testing-guide)
10. [Monitoring & Debugging](#monitoring--debugging)
11. [Best Practices](#best-practices)
12. [Troubleshooting](#troubleshooting)

---

## Overview

This POC demonstrates a complete Google Cloud Pub/Sub implementation using Node.js and Express. It provides REST API endpoints for:
- Creating topics
- Publishing messages
- Creating subscriptions
- Real-time message consumption with console logging

**Key Features:**
- ✅ Dual configuration support (Local Emulator + Google Cloud)
- ✅ RESTful API endpoints
- ✅ Real-time message processing
- ✅ Automatic message acknowledgment
- ✅ Support for message attributes
- ✅ Graceful shutdown handling

---

## Architecture

```
┌─────────────────┐
│   Client/API    │
│   (curl/HTTP)   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────┐
│      Express Server (Port 3002)     │
│  ┌───────────────────────────────┐  │
│  │  POST /topics                 │  │
│  │  POST /publish                │  │
│  │  POST /subscriptions          │  │
│  │  GET  /topics                 │  │
│  │  GET  /subscriptions          │  │
│  │  DELETE /subscriptions/:name  │  │
│  └───────────────────────────────┘  │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│   Google Cloud Pub/Sub Client       │
│   (@google-cloud/pubsub)            │
└────────┬────────────────────────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌─────────┐ ┌──────────────────┐
│ Emulator│ │ Google Cloud     │
│ (Local) │ │ Pub/Sub Service  │
└─────────┘ └──────────────────┘
```

---

## Prerequisites

### Required Software
- **Node.js**: v14 or higher
- **npm**: v6 or higher
- **curl**: For testing (or any HTTP client)

### For Local Development (Emulator)
- **Google Cloud SDK**: For running the emulator
  ```bash
  # macOS
  brew install --cask google-cloud-sdk
  
  # Linux
  curl https://sdk.cloud.google.com | bash
  ```

### For Google Cloud (Production)
- **Google Cloud Account**: With billing enabled
- **Google Cloud Project**: Created and configured
- **Service Account**: With Pub/Sub permissions

---

## Project Structure

```
google-pub-sub-poc/
├── server.js                    # Main application file
├── package.json                 # Dependencies and scripts
├── .env                        # Environment configuration (gitignored)
├── .env.example                # Environment template
├── .gitignore                  # Git ignore rules
├── README.md                   # Quick start guide
├── SETUP_GUIDE.md             # Detailed setup instructions
├── POC_DOCUMENTATION.md       # This file
└── essential-hawk-*.json      # GCP service account key (gitignored)
```

---

## Configuration Options

### Environment Variables

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `GOOGLE_CLOUD_PROJECT_ID` | Yes | Your GCP project ID | `essential-hawk-444711-m3` |
| `GOOGLE_APPLICATION_CREDENTIALS` | Cloud only | Path to service account JSON | `/path/to/key.json` |
| `PUBSUB_EMULATOR_HOST` | Emulator only | Emulator host and port | `localhost:8085` |
| `PORT` | No | Server port (default: 3000) | `3002` |

### Configuration Modes

#### Mode 1: Local Emulator
```env
GOOGLE_CLOUD_PROJECT_ID=test-project
PUBSUB_EMULATOR_HOST=localhost:8085
PORT=3002
```

#### Mode 2: Google Cloud
```env
GOOGLE_CLOUD_PROJECT_ID=essential-hawk-444711-m3
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account-key.json
PORT=3002
```

---

## Local Development Setup (Emulator)

### Step 1: Install Dependencies
```bash
npm install
```

### Step 2: Install Pub/Sub Emulator
```bash
gcloud components install pubsub-emulator
gcloud components update
```

### Step 3: Configure Environment
Create `.env` file:
```env
GOOGLE_CLOUD_PROJECT_ID=test-project
PUBSUB_EMULATOR_HOST=localhost:8085
PORT=3002
```

### Step 4: Start Emulator
**Terminal 1** - Start the emulator:
```bash
gcloud beta emulators pubsub start --project=test-project
```

Expected output:
```
[pubsub] This is the Google Pub/Sub fake.
[pubsub] Server started, listening on 8085
```

### Step 5: Start Application
**Terminal 2** - Start the server:
```bash
npm start
```

Expected output:
```
🚀 Google Pub/Sub POC Server running on port 3002

Available endpoints:
  POST   /topics              - Create a new topic
  GET    /topics              - List all topics
  POST   /publish             - Publish a message to a topic
  POST   /subscriptions       - Create a subscription and start listening
  GET    /subscriptions       - List active subscriptions
  DELETE /subscriptions/:name - Delete a subscription
  GET    /health              - Health check
```

### Step 6: Test the Setup
```bash
# Health check
curl http://localhost:3002/health

# Create topic
curl -X POST http://localhost:3002/topics \
  -H "Content-Type: application/json" \
  -d '{"topicName": "test-topic"}'

# Create subscription
curl -X POST http://localhost:3002/subscriptions \
  -H "Content-Type: application/json" \
  -d '{"topicName": "test-topic", "subscriptionName": "test-sub"}'

# Publish message
curl -X POST http://localhost:3002/publish \
  -H "Content-Type: application/json" \
  -d '{"topicName": "test-topic", "message": "Hello Emulator!"}'
```

**Check your server console** - you should see the message logged!

---

## Google Cloud Setup (Production)

### Step 1: Create Google Cloud Project

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Click **New Project**
3. Enter project name: `pubsub-poc`
4. Note the **Project ID** (e.g., `essential-hawk-444711-m3`)

### Step 2: Enable Pub/Sub API

1. Navigate to **APIs & Services** > **Library**
2. Search for "Cloud Pub/Sub API"
3. Click **Enable**

**Or use this direct link:**
```
https://console.developers.google.com/apis/api/pubsub.googleapis.com/overview?project=YOUR_PROJECT_ID
```

### Step 3: Create Service Account

1. Go to **IAM & Admin** > **Service Accounts**
2. Click **Create Service Account**
3. Enter details:
   - **Name**: `pubsub-poc-service-account`
   - **Description**: `Service account for Pub/Sub POC`
4. Click **Create and Continue**

### Step 4: Grant Permissions

Add these roles:
- **Pub/Sub Admin** (full access)
- OR **Pub/Sub Publisher** + **Pub/Sub Subscriber** (limited access)

Click **Continue** > **Done**

### Step 5: Create Service Account Key

1. Click on the service account
2. Go to **Keys** tab
3. Click **Add Key** > **Create new key**
4. Select **JSON** format
5. Click **Create**
6. Save the downloaded file securely

### Step 6: Configure Environment

Move the key file:
```bash
mkdir -p ~/.gcp
mv ~/Downloads/essential-hawk-*.json ~/.gcp/pubsub-key.json
```

Create `.env` file:
```env
GOOGLE_CLOUD_PROJECT_ID=essential-hawk-444711-m3
GOOGLE_APPLICATION_CREDENTIALS=/Users/yourusername/.gcp/pubsub-key.json
PORT=3002
```

### Step 7: Start Application

```bash
npm start
```

### Step 8: Test Cloud Setup

```bash
# Create topic in cloud
curl -X POST http://localhost:3002/topics \
  -H "Content-Type: application/json" \
  -d '{"topicName": "cloud-topic"}'

# Create subscription
curl -X POST http://localhost:3002/subscriptions \
  -H "Content-Type: application/json" \
  -d '{"topicName": "cloud-topic", "subscriptionName": "cloud-sub"}'

# Publish message
curl -X POST http://localhost:3002/publish \
  -H "Content-Type: application/json" \
  -d '{"topicName": "cloud-topic", "message": "Hello Cloud!"}'
```

### Step 9: Verify in Google Cloud Console

**View Topics:**
```
https://console.cloud.google.com/cloudpubsub/topic/list?project=YOUR_PROJECT_ID
```

**View Subscriptions:**
```
https://console.cloud.google.com/cloudpubsub/subscription/list?project=YOUR_PROJECT_ID
```

---

## API Endpoints

### 1. Health Check
**GET** `/health`

Check if server is running.

**Request:**
```bash
curl http://localhost:3002/health
```

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2025-12-17T06:00:00.000Z"
}
```

---

### 2. Create Topic
**POST** `/topics`

Create a new Pub/Sub topic.

**Request:**
```bash
curl -X POST http://localhost:3002/topics \
  -H "Content-Type: application/json" \
  -d '{
    "topicName": "my-topic"
  }'
```

**Response:**
```json
{
  "message": "Topic created successfully",
  "topic": "projects/essential-hawk-444711-m3/topics/my-topic"
}
```

**Error Response:**
```json
{
  "error": "Failed to create topic",
  "details": "Topic already exists"
}
```

---

### 3. List Topics
**GET** `/topics`

List all topics in the project.

**Request:**
```bash
curl http://localhost:3002/topics
```

**Response:**
```json
{
  "topics": [
    "projects/essential-hawk-444711-m3/topics/my-topic",
    "projects/essential-hawk-444711-m3/topics/another-topic"
  ],
  "count": 2
}
```

---

### 4. Publish Message
**POST** `/publish`

Publish a message to a topic.

**Request (String Message):**
```bash
curl -X POST http://localhost:3002/publish \
  -H "Content-Type: application/json" \
  -d '{
    "topicName": "my-topic",
    "message": "Hello World!",
    "attributes": {
      "priority": "high",
      "source": "api"
    }
  }'
```

**Request (JSON Message):**
```bash
curl -X POST http://localhost:3002/publish \
  -H "Content-Type: application/json" \
  -d '{
    "topicName": "my-topic",
    "message": {
      "userId": "12345",
      "action": "login",
      "timestamp": "2025-12-17T06:00:00Z"
    },
    "attributes": {
      "type": "user-event"
    }
  }'
```

**Response:**
```json
{
  "message": "Message published successfully",
  "messageId": "17393895727084501",
  "topic": "my-topic"
}
```

---

### 5. Create Subscription
**POST** `/subscriptions`

Create a subscription and start listening for messages.

**Request:**
```bash
curl -X POST http://localhost:3002/subscriptions \
  -H "Content-Type: application/json" \
  -d '{
    "topicName": "my-topic",
    "subscriptionName": "my-subscription"
  }'
```

**Response:**
```json
{
  "message": "Subscription created and listening for messages",
  "subscription": "projects/essential-hawk-444711-m3/subscriptions/my-subscription",
  "topic": "my-topic"
}
```

**Server Console Output (when messages arrive):**
```
=== Received Message ===
Message ID: 17393895727084501
Data: "Hello World!"
Attributes: {"priority":"high","source":"api"}
Published: Wed Dec 17 2025 11:30:00 GMT+0530 (India Standard Time)
========================
```

---

### 6. List Subscriptions
**GET** `/subscriptions`

List all active subscriptions.

**Request:**
```bash
curl http://localhost:3002/subscriptions
```

**Response:**
```json
{
  "activeSubscriptions": [
    "my-subscription",
    "another-subscription"
  ],
  "count": 2
}
```

---

### 7. Delete Subscription
**DELETE** `/subscriptions/:subscriptionName`

Stop listening and delete a subscription.

**Request:**
```bash
curl -X DELETE http://localhost:3002/subscriptions/my-subscription
```

**Response:**
```json
{
  "message": "Subscription deleted successfully",
  "subscription": "my-subscription"
}
```

---

## Testing Guide

### Complete Workflow Test

```bash
# 1. Health check
curl http://localhost:3002/health

# 2. Create topic
curl -X POST http://localhost:3002/topics \
  -H "Content-Type: application/json" \
  -d '{"topicName": "demo-topic"}'

# 3. List topics
curl http://localhost:3002/topics

# 4. Create subscription (starts listening)
curl -X POST http://localhost:3002/subscriptions \
  -H "Content-Type: application/json" \
  -d '{
    "topicName": "demo-topic",
    "subscriptionName": "demo-sub"
  }'

# 5. Publish test messages
curl -X POST http://localhost:3002/publish \
  -H "Content-Type: application/json" \
  -d '{
    "topicName": "demo-topic",
    "message": "Test message 1",
    "attributes": {"test": "true"}
  }'

curl -X POST http://localhost:3002/publish \
  -H "Content-Type: application/json" \
  -d '{
    "topicName": "demo-topic",
    "message": {"data": "Test message 2", "count": 42},
    "attributes": {"type": "json"}
  }'

# 6. List active subscriptions
curl http://localhost:3002/subscriptions

# 7. Clean up - delete subscription
curl -X DELETE http://localhost:3002/subscriptions/demo-sub
```

### Load Testing

Publish multiple messages rapidly:
```bash
for i in {1..10}; do
  curl -X POST http://localhost:3002/publish \
    -H "Content-Type: application/json" \
    -d "{\"topicName\": \"demo-topic\", \"message\": \"Message $i\"}"
done
```

---

## Monitoring & Debugging

### Local Monitoring (Emulator)

**Server Logs:**
- All messages are logged to console
- Check Terminal 2 (where server is running)

**Emulator Logs:**
- Check Terminal 1 (where emulator is running)
- Shows all Pub/Sub operations

### Google Cloud Monitoring

#### 1. Cloud Console Metrics

**Topics:**
```
https://console.cloud.google.com/cloudpubsub/topic/list?project=YOUR_PROJECT_ID
```

Click on a topic to see:
- Publish rate
- Message size
- Error rate

**Subscriptions:**
```
https://console.cloud.google.com/cloudpubsub/subscription/list?project=YOUR_PROJECT_ID
```

Click on a subscription to see:
- Delivery rate
- Oldest unacked message age
- Backlog size
- Acknowledgment rate

#### 2. Cloud Logging

View detailed logs:
```
https://console.cloud.google.com/logs/query?project=YOUR_PROJECT_ID
```

Filter for Pub/Sub logs:
```
resource.type="pubsub_topic"
OR resource.type="pubsub_subscription"
```

#### 3. Metrics Explorer

Create custom dashboards:
```
https://console.cloud.google.com/monitoring/metrics-explorer?project=YOUR_PROJECT_ID
```

Useful metrics:
- `pubsub.googleapis.com/topic/send_request_count`
- `pubsub.googleapis.com/subscription/pull_request_count`
- `pubsub.googleapis.com/subscription/num_undelivered_messages`

---

## Best Practices

### 1. Message Design

**Good:**
```json
{
  "message": {
    "eventType": "user.login",
    "userId": "12345",
    "timestamp": "2025-12-17T06:00:00Z",
    "metadata": {
      "ip": "192.168.1.1",
      "userAgent": "Mozilla/5.0"
    }
  },
  "attributes": {
    "priority": "normal",
    "source": "auth-service"
  }
}
```

**Avoid:**
- Very large messages (>10MB)
- Sensitive data without encryption
- Missing timestamps
- Unclear message structure

### 2. Error Handling

Always handle errors:
```javascript
try {
  const messageId = await topic.publishMessage({
    data: dataBuffer,
    attributes: attributes
  });
} catch (error) {
  console.error('Publish failed:', error);
  // Implement retry logic or dead letter queue
}
```

### 3. Subscription Management

- Use meaningful subscription names
- Set appropriate acknowledgment deadlines
- Monitor unacknowledged messages
- Implement dead letter topics for failed messages

### 4. Security

**Local Development:**
- Use emulator (no credentials needed)
- Never commit `.env` files

**Production:**
- Use service accounts with minimal permissions
- Rotate keys regularly
- Store credentials securely (not in code)
- Use Secret Manager for sensitive data

### 5. Cost Optimization

- Delete unused topics and subscriptions
- Use message filtering when possible
- Set appropriate message retention
- Monitor quota usage

---

## Troubleshooting

### Common Issues

#### 1. "Could not load the default credentials"

**Cause:** Missing or incorrect `GOOGLE_APPLICATION_CREDENTIALS`

**Solution:**
```bash
# Check if file exists
ls -la $GOOGLE_APPLICATION_CREDENTIALS

# Verify path in .env
cat .env | grep GOOGLE_APPLICATION_CREDENTIALS

# Set correct path
export GOOGLE_APPLICATION_CREDENTIALS="/correct/path/to/key.json"
```

#### 2. "API has not been used in project"

**Cause:** Pub/Sub API not enabled

**Solution:**
```
https://console.developers.google.com/apis/api/pubsub.googleapis.com/overview?project=YOUR_PROJECT_ID
```
Click "Enable"

#### 3. "Permission denied"

**Cause:** Service account lacks permissions

**Solution:**
1. Go to IAM & Admin > IAM
2. Find your service account
3. Add "Pub/Sub Admin" role

#### 4. "Connection refused" (Emulator)

**Cause:** Emulator not running

**Solution:**
```bash
# Start emulator
gcloud beta emulators pubsub start --project=test-project

# Verify it's running
lsof -i :8085
```

#### 5. Messages not received

**Checklist:**
- ✅ Subscription created successfully?
- ✅ Publishing to correct topic?
- ✅ Server console showing subscription created?
- ✅ No errors in server logs?

**Debug:**
```bash
# Check active subscriptions
curl http://localhost:3002/subscriptions

# Verify topic exists
curl http://localhost:3002/topics

# Try publishing again
curl -X POST http://localhost:3002/publish \
  -H "Content-Type: application/json" \
  -d '{"topicName": "your-topic", "message": "test"}'
```

---

## Switching Between Emulator and Cloud

### From Emulator to Cloud

1. Stop the server (Ctrl+C)
2. Update `.env`:
   ```env
   GOOGLE_CLOUD_PROJECT_ID=essential-hawk-444711-m3
   GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json
   # PUBSUB_EMULATOR_HOST=localhost:8085  # Comment this out
   PORT=3002
   ```
3. Restart server: `npm start`

### From Cloud to Emulator

1. Stop the server (Ctrl+C)
2. Start emulator (if not running):
   ```bash
   gcloud beta emulators pubsub start --project=test-project
   ```
3. Update `.env`:
   ```env
   GOOGLE_CLOUD_PROJECT_ID=test-project
   PUBSUB_EMULATOR_HOST=localhost:8085
   # GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json  # Comment this out
   PORT=3002
   ```
4. Restart server: `npm start`

---

## Additional Resources

### Documentation
- [Google Cloud Pub/Sub Docs](https://cloud.google.com/pubsub/docs)
- [Node.js Client Library](https://googleapis.dev/nodejs/pubsub/latest/)
- [Pub/Sub Best Practices](https://cloud.google.com/pubsub/docs/publisher)

### Tools
- [Pub/Sub Emulator](https://cloud.google.com/pubsub/docs/emulator)
- [gcloud CLI](https://cloud.google.com/sdk/gcloud)
- [Postman Collection](https://www.postman.com/) - Import endpoints for testing

### Support
- [Stack Overflow](https://stackoverflow.com/questions/tagged/google-cloud-pubsub)
- [Google Cloud Community](https://www.googlecloudcommunity.com/)
- [GitHub Issues](https://github.com/googleapis/nodejs-pubsub/issues)

---

## Conclusion

This POC demonstrates a complete Google Pub/Sub implementation with:
- ✅ Dual environment support (Local + Cloud)
- ✅ RESTful API interface
- ✅ Real-time message processing
- ✅ Production-ready error handling
- ✅ Comprehensive documentation

**Next Steps:**
1. Explore message filtering
2. Implement dead letter queues
3. Add message ordering
4. Set up monitoring alerts
5. Implement retry policies
6. Add authentication to API endpoints

---

**Last Updated:** December 17, 2025  
**Version:** 1.0.0  
**Author:** POC Development Team
