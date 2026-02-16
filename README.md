# Parlona Voice Core

**Production-ready call analytics platform with speech-to-text, AI summarization, and entity extraction.**

Deploy your own call processing infrastructure using pre-built Docker images. Process phone calls in stereo WAV format with speaker diarization (agent/customer separation).

![Dashboard Screenshot](images/dashboard.png)

---

## Features

- **Speech-to-Text (STT)**: Powered by Faster-Whisper with stereo channel diarization
- **AI Summarization**: LLM-powered call summaries via OpenAI, Groq, or vLLM
- **Entity Extraction**: Automatic named entity recognition with speaker attribution
- **Sentiment Analysis**: Call sentiment scoring and labeling
- **REST API**: FastAPI-based endpoints for job management
- **PostgreSQL Storage**: Persistent storage for calls, dialogue turns, and insights
- **Multi-Machine Support**: Deploy across multiple servers (CPU + GPU)

---

## Quick Start (Single Machine)

```bash
# 1. Clone the deployment repository
git clone https://github.com/vashkelis/parlona-voice-core.git
cd parlona-voice-core

# 2. Configure environment
cp .env.example .env
nano .env  # Edit with your settings

# 3. Deploy
docker compose -f docker-compose.prod.yml up -d

# 4. Access API at http://localhost:8080
```

---

## Configuration

### Required Environment Variables

Edit `.env` file:

```bash
# API Authentication (generate strong random key)
CALL_API_KEY=$(openssl rand -hex 32)

# Redis Password
REDIS_PASSWORD=your_secure_redis_password

# PostgreSQL Credentials
POSTGRES_USER=parlonacore
POSTGRES_PASSWORD=your_secure_db_password
POSTGRES_DB=parlonacore

# Optional: OpenAI API (for LLM summarization)
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini
```

### Optional Configuration

```bash
# STT Model Selection
STT_MODEL_NAME=Systran/faster-whisper-small  # or tiny, medium, large-v2
STT_DIARIZATION_MODE=stereo_channels
STT_STEREO_SPEAKER_MAPPING=0:agent,1:customer

# GPU Support (if available)
STT_ENABLE_GPU=1
FORCE_CPU=0

# vLLM Configuration (for local LLM)
LLM_BACKEND=vllm
VLLM_API_BASE=http://vllm-host:8000/v1
VLLM_MODEL=lmsys/vicuna-7b-v1.5
```

---

## Deployment Options

### Option 1: Single Machine (All-in-One)

```bash
docker compose -f docker-compose.prod.yml up -d
```

Services: Redis, PostgreSQL, API, STT, Summary, Postprocess

### Option 2: Two Machines (STT on GPU)

**Machine #1 (CPU) - API Server:**
```bash
# Run without STT
docker compose -f docker-compose.cpu.yml up -d
```

**Machine #2 (GPU) - STT Worker:**
```bash
./docker-stt-gpu.sh <machine1-ip> <redis-password>
```

### Option 3: Three Machines (STT + vLLM on GPU)

**Machine #1 (CPU) - API + Workers:**
```bash
docker compose -f docker-compose.cpu.yml up -d
```

**Machine #2 (GPU) - STT Worker:**
```bash
./docker-stt-gpu.sh <machine1-ip> <redis-password>
```

**Machine #3 (GPU) - vLLM Server:**
```bash
./docker-vllm.sh lmsys/vicuna-7b-v1.5 your-api-key
```

Update Machine #1 `.env`:
```bash
LLM_BACKEND=vllm
VLLM_API_BASE=http://machine3-ip:8000/v1
VLLM_API_KEY=your-api-key
VLLM_MODEL=lmsys/vicuna-7b-v1.5
```

---

## Stereo WAV Format

This application processes **stereo WAV files** where:
- **Left channel (0)**: Agent/speaker 1
- **Right channel (1)**: Customer/speaker 2

The STT service automatically separates speakers based on audio channels.

### Asterisk Configuration

To record calls in stereo format from Asterisk:

**1. Install required modules:**
```bash
# In Asterisk CLI
module load app_mixmonitor.so
module load format_wav.so
```

**2. Configure extensions.conf:**
```ini
[default]

; Record inbound calls
exten => _X.,1,NoOp(Incoming call from ${CALLERID(num)})
 same => n,MixMonitor(/var/recordings/${UNIQUEID}.wav,b,/usr/local/bin/submit-to-parlona.sh ${UNIQUEID} ${CALLERID(num)} ${EXTEN})
 same => n,Dial(PJSIP/${EXTEN})
 same => n,Hangup()

; Record outbound calls
exten => _9X.,1,NoOp(Outgoing call to ${EXTEN:1})
 same => n,MixMonitor(/var/recordings/${UNIQUEID}.wav,b,/usr/local/bin/submit-to-parlona.sh ${UNIQUEID} ${CALLERID(num)} ${EXTEN:1})
 same => n,Dial(PJSIP/${EXTEN:1})
 same => n,Hangup()
```

**3. Create submission script (`/usr/local/bin/submit-to-parlona.sh`):**
```bash
#!/bin/bash
# Submit recorded call to Parlona for processing

CALL_ID=$1
CALLER_NUMBER=$2
CALLED_NUMBER=$3
RECORDING_FILE="/var/recordings/${CALL_ID}.wav"

# Wait for recording to complete
sleep 2

# Submit to Parlona API
curl -X POST \
  -H "X-API-Key: YOUR_CALL_API_KEY" \
  -F "file=@${RECORDING_FILE}" \
  -F "call_id=${CALL_ID}" \
  -F "direction=${DIRECTION}" \
  -F "customer_number=${CALLER_NUMBER}" \
  -F "agent_id=${CALLED_NUMBER}" \
  http://parlona-api-host:8080/v1/jobs/upload

# Optional: Move processed file
mv "${RECORDING_FILE}" "/var/recordings/processed/${CALL_ID}.wav"
```

**4. Make executable:**
```bash
chmod +x /usr/local/bin/submit-to-parlona.sh
```

**5. Asterisk mixmonitor.conf for stereo:**
```ini
[general]
; Enable stereo recording
stereorx=yes
stereotx=yes
```

---

## API Usage

### Submit Audio for Processing

```bash
curl -X POST \
  -H "X-API-Key: YOUR_CALL_API_KEY" \
  -F "file=@call_recording.wav" \
  -F "call_id=unique-call-id" \
  -F "direction=inbound" \
  -F "customer_number=+1234567890" \
  -F "agent_id=agent_001" \
  http://localhost:8080/v1/jobs/upload
```

### Check Job Status

```bash
curl -H "X-API-Key: YOUR_CALL_API_KEY" \
  http://localhost:8080/v1/jobs/{job_id}
```

### List Processed Calls

```bash
curl -H "X-API-Key: YOUR_CALL_API_KEY" \
  http://localhost:8080/v1/calls
```

### Get Call Details with Transcript

```bash
curl -H "X-API-Key: YOUR_CALL_API_KEY" \
  http://localhost:8080/v1/calls/{call_id}
```

---

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    Parlona Voice Core                      │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   Redis     │◀───│  STT Worker │    │  vLLM       │     │
│  │  (Queue)    │    │   (GPU)     │    │  (GPU)      │     │
│  └──────┬──────┘    └─────────────┘    └─────────────┘     │
│         │                                                  │
│  ┌──────▼──────┐    ┌─────────────┐    ┌─────────────┐     │
│  │     API     │───▶│  Summary    │    │ PostgreSQL  │     │
│  │   Server    │    │  Service    │    │  (Storage)  │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│         │                                                  │
│  ┌──────▼──────┐                                           │
│  │ Postprocess │                                           │
│  │  Service    │                                           │
│  └─────────────┘                                           │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Docker Images

All images are available on Docker Hub:

| Service | Image |
|---------|-------|
| API | `parlona/voicecore-api:1.2.0` |
| STT | `parlona/voicecore-stt:1.2.0` |
| Summary | `parlona/voicecore-summary:1.2.0` |
| Postprocess | `parlona/voicecore-postprocess:1.2.0` |
| Frontend | `parlona/voicecore-frontend:1.2.0` |

---

## Troubleshooting

### Check Service Status
```bash
docker compose -f docker-compose.prod.yml ps
docker compose -f docker-compose.prod.yml logs -f [service-name]
```

### Redis Connection Issues
```bash
# Test Redis from container
docker exec -it redis redis-cli -a YOUR_PASSWORD ping
```

### GPU Not Detected
```bash
# Check GPU availability
docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi
```

### Model Download Fails
```bash
# Set to allow online downloads
WHISPER_LOCAL_ONLY=0
HF_HUB_OFFLINE=0
```

---

## License

Apache 2.0 - See LICENSE file

---

## Support

- GitHub Issues: https://github.com/vashkelis/parlona-voice-core/issues
- Documentation: https://github.com/vashkelis/parlona-voice-core/wiki
