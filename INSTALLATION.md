# Installation & Setup Guide

Complete step-by-step guide to deploy the AI SDR system.

## Prerequisites

### System Requirements
- **OS**: Linux, macOS, or Windows (WSL2)
- **Python**: 3.10 or higher
- **RAM**: 8GB minimum (16GB+ recommended)
- **Disk**: 20GB for models and databases
- **Network**: Public internet access for Twilio webhooks

### External Services (Free Tier Available)
1. **Twilio** - Voice API provider
   - Sign up: https://www.twilio.com
   - Get: Account SID, Auth Token, Phone Number
   - Cost: ~$0.013/minute for inbound, ~$0.02/minute for outbound

2. **Deepgram** - Speech AI provider
   - Sign up: https://console.deepgram.com
   - Get: API Key
   - Cost: Free tier includes transcription minutes

3. **Ollama** - Local LLM inference
   - Download: https://ollama.ai
   - Install locally, no credentials needed

## Step-by-Step Installation

### 1. Prepare Environment

```bash
# Clone repository (if not already done)
cd /root/AI_SDR

# Create Python virtual environment
python3.10 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Verify Python version
python --version  # Should be 3.10+
```

### 2. Install Dependencies

```bash
# Upgrade pip
pip install --upgrade pip setuptools wheel

# Install all requirements
pip install -r requirements.txt

# Verify installation
python -c "import fastapi; import ollama; import deepgram; print('✅ All imports successful')"
```

**Common Issues:**
- `torch` installation fails: Install CUDA toolkit first for GPU support
- `chromadb` fails: Ensure SQLite development files are installed
- Memory error: Reduce number of packages, install in stages

### 3. Get External Credentials

#### Twilio Setup
1. Go to https://www.twilio.com/console
2. Note your **Account SID** and **Auth Token**
3. Buy a phone number in Console → Phone Numbers → Buy numbers
4. Note the **Phone Number** (e.g., +1234567890)

#### Deepgram Setup
1. Go to https://console.deepgram.com/
2. Create new API key in the settings
3. Copy the **API Key**

#### Ollama Setup
1. Download from https://ollama.ai
2. Install and start: `ollama serve`
3. Pull model: `ollama pull llama3:8b-instruct-q4_K_S`
4. Verify: `ollama list` (should show the model)

### 4. Configure Environment

```bash
# Create .env file from template
cp .env .env.backup  # Backup existing
cat > .env << 'EOF'
# === TWILIO (Required) ===
TWILIO_ACCOUNT_SID=your_account_sid_here
TWILIO_AUTH_TOKEN=your_auth_token_here
TWILIO_PHONE_NUMBER=+1234567890

# === PUBLIC URL (Required) ===
# For development: ngrok URL (e.g., https://xxx.ngrok-free.dev)
# For production: your domain (e.g., https://yourcompany.com)
PUBLIC_URL=https://your-ngrok-or-domain-url

# === DEEPGRAM (Required) ===
DEEPGRAM_API_KEY=your_deepgram_api_key_here

# === VOICE SETTINGS (Optional) ===
DEEPGRAM_VOICE=aura-2-thalia-en
DEEPGRAM_STT_MODEL=nova-2
DEEPGRAM_STT_FALLBACK_MODEL=nova-2-general

# === LLM SETTINGS (Optional) ===
OLLAMA_MODEL=llama3:8b-instruct-q4_K_S
EMBED_MODEL=sentence-transformers/all-MiniLM-L6-v2

# === INTERRUPT DETECTION (Optional) ===
INTERRUPT_ENABLED=true
INTERRUPT_MIN_ENERGY=800
INTERRUPT_DEBOUNCE_MS=400
INTERRUPT_BASELINE_FACTOR=2.8
INTERRUPT_MIN_SPEECH_MS=100
INTERRUPT_REQUIRE_TEXT=false

# === SILENCE DETECTION (Optional) ===
SILENCE_THRESHOLD_SEC=0.3

# === DATABASE ===
AGENT_DATABASE_URL=sqlite:///./agents.db

# === VECTOR DB ===
CHROMA_PATH=./chroma_db
CHUNK_SIZE=384
TOP_K=3

# === LOGGING (Optional) ===
LOG_LEVEL=INFO
LOG_FILE=server.log
EOF
```

Edit the file and replace placeholders with your actual credentials:
```bash
nano .env  # Or use your preferred editor
```

**Verification:**
```bash
# Check all required vars are set
grep -E "TWILIO_ACCOUNT_SID|TWILIO_AUTH_TOKEN|TWILIO_PHONE_NUMBER|PUBLIC_URL|DEEPGRAM_API_KEY" .env
# Should show 5 lines, none empty
```

### 5. Start Ollama (in separate terminal)

```bash
# Terminal 1: Start Ollama
ollama serve

# Should output:
# Loaded weights for the model: ...
# Listening on [::1]:11434
```

Keep this terminal running. The LLM API will be at `http://localhost:11434`.

### 6. Test Public URL (Development with ngrok)

```bash
# Terminal 2: Start ngrok
ngrok http 9001

# Output example:
# Forwarding   https://abc123def456.ngrok-free.dev -> http://localhost:9001
# 
# Copy the https URL and update PUBLIC_URL in .env
```

### 7. Start FastAPI Application

```bash
# Terminal 3: Start application
uvicorn main:app --host 0.0.0.0 --port 9001 --reload

# Output should show:
# Uvicorn running on http://0.0.0.0:9001
# 
# Check health: curl http://localhost:9001/health
```

### 8. Configure Twilio Webhooks

1. Go to Twilio Console → Phone Numbers → Your Phone Number
2. Under "Voice Configuration":
   - **When a call comes in**: `PUBLIC_URL/twiml/voice` (POST)
   - **When a call ends**: `PUBLIC_URL/twiml/call_end` (POST)

3. Test: Call your Twilio phone number
   - Should hear greeting from agent
   - Transcription should appear in logs

## Verification Checklist

Run these tests to verify setup:

```bash
# 1. API is running
curl http://localhost:9001/health

# 2. Can create agent
curl -X POST http://localhost:9001/v1/convai/agents \
  -H "Content-Type: application/json" \
  -H "xi-api-key: default" \
  -d '{
    "name": "Test Agent",
    "system_prompt": "You are helpful.",
    "first_message": "Hello, how can I help?"
  }'

# 3. Ollama is responsive
curl http://localhost:11434/api/tags

# 4. Deepgram API key works
curl -X GET https://api.deepgram.com/v1/models \
  -H "Authorization: Token YOUR_DEEPGRAM_KEY"
```

## Directory Structure After Setup

```
/root/AI_SDR/
├── venv/                  # Python virtual environment
├── agents.db              # SQLite database (created on first run)
├── chroma_db/             # Vector database
├── server.log             # Application logs
├── .env                   # Your configuration (KEEP SECRET!)
├── .env.backup            # Backup of .env
├── main.py
├── voice_pipeline.py
├── models.py
├── schemas.py
├── utils.py
├── requirements.txt
├── README.md              # Main documentation
└── API_ENDPOINTS.md       # API reference
```

## Troubleshooting Installation

### ImportError: No module named 'torch'
```bash
# Install PyTorch (CPU only)
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu

# Or with GPU (CUDA 11.8)
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
```

### Connection refused: localhost:11434
- Ensure Ollama is running: `ollama serve`
- Check Ollama status: `ollama list`
- Model not loaded: `ollama pull llama3:8b-instruct-q4_K_S`

### Deepgram API key not working
- Verify key in console: https://console.deepgram.com/
- Check key is in .env (not in quotes)
- Try with curl: `curl https://api.deepgram.com/v1/models -H "Authorization: Token KEY"`

### Twilio webhook not receiving calls
- Verify PUBLIC_URL is accessible: `curl YOUR_PUBLIC_URL`
- Check webhook URL in Twilio Console is correct
- Check ngrok is running and PUBLIC_URL matches
- Check firewall isn't blocking inbound webhooks

### Out of memory
- Free up RAM: Close other applications
- Reduce model size: Use `mistral:latest` instead of `llama3`
- Increase swap: `fallocate -l 4G /swapfile && mkswap /swapfile && swapon /swapfile`

### Database locked
- Stop application: `pkill -f uvicorn`
- Remove lock: `rm agents.db-wal agents.db-shm`
- Restart application

## Production Deployment

### Using systemd (Ubuntu/Debian)

```bash
# Create service file
sudo cat > /etc/systemd/system/ai-sdr.service << 'EOF'
[Unit]
Description=AI SDR Voice Agent
After=network.target

[Service]
Type=simple
User=sdr
WorkingDirectory=/root/AI_SDR
Environment="PATH=/root/AI_SDR/venv/bin"
ExecStart=/root/AI_SDR/venv/bin/uvicorn main:app --host 0.0.0.0 --port 9001
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Enable and start
sudo systemctl enable ai-sdr
sudo systemctl start ai-sdr

# Monitor
sudo journalctl -u ai-sdr -f
```

### Using Docker

```dockerfile
FROM python:3.10-slim
WORKDIR /app
RUN apt-get update && apt-get install -y build-essential
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "9001"]
```

Build and run:
```bash
docker build -t ai-sdr .
docker run -p 9001:9001 --env-file .env ai-sdr
```

### Using Gunicorn (Production ASGI)

```bash
pip install gunicorn

gunicorn main:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:9001 \
  --access-logfile - \
  --error-logfile -
```

## Monitoring in Production

```bash
# Monitor logs
tail -f server.log

# Filter errors
grep "ERROR" server.log

# Count calls
grep "CALL_START" server.log | wc -l

# Performance metrics
grep "⏱️" server.log  # If latency tracking enabled
```

## Next Steps

1. **Create your first agent**: See `API_ENDPOINTS.md`
2. **Upload knowledge base**: See `MODULARIZATION_SUMMARY.md`
3. **Configure webhooks**: Test with your service
4. **Monitor performance**: Check logs and metrics
5. **Load test**: Test with concurrent calls

---

**Need help?** Check logs in `server.log` for detailed error information.
