# Koyeb Deployment Guide

Complete guide for deploying the Parakeet-TDT 0.6B v3 Speech-to-Text service to Koyeb.

## Prerequisites

- ✅ GitHub repository with Parakeet service code
- ✅ Koyeb account with GPU access
- ✅ No authentication tokens required (model is publicly available)

## Step-by-Step Deployment

### 1. Create New Service on Koyeb

1. Log in to [Koyeb Dashboard](https://app.koyeb.com)
2. Click **"Create Service"**
3. Select **"GitHub"** as deployment source

### 2. Connect GitHub Repository

1. **Authorize GitHub** (if first time)
2. Select your repository containing the parakeet service
3. Branch: **`main`** (or your preferred branch)
4. **Auto-deploy**: ✅ Enable (deploys on git push)

### 3. Configure Build Settings

**Build Method:**
- ✅ **Docker** (CRITICAL: Must use Docker, not Buildpack!)
- Dockerfile path: `./Dockerfile` (or path to parakeet Dockerfile)
- Context: `.` (root directory)

**⚠️ IMPORTANT**: You MUST select "Docker" as the build method. The default buildpack does not include CUDA/GPU support and PyTorch dependencies required for this service.

### 4. Set Environment Variables

Add these environment variables in Koyeb:

#### Core Configuration (GPU Optimized)

| Variable | Value | Required | Description |
|----------|-------|----------|-------------|
| `MODEL_PRECISION` | `fp16` | ✅ Yes | Model precision (fp16/fp32) |
| `DEVICE` | `cuda` | ✅ Yes | Use GPU acceleration |
| `BATCH_SIZE` | `8` | ✅ Yes | Processing batch size (8 for A100 80GB) |

#### Audio Processing

| Variable | Value | Recommended | Description |
|----------|-------|-------------|-------------|
| `TARGET_SR` | `16000` | ✅ Yes | Audio sample rate (Hz) - model native |
| `MAX_AUDIO_DURATION` | `30` | ✅ Yes | Max audio chunk duration (seconds) |
| `VAD_THRESHOLD` | `0.5` | ✅ Yes | Voice Activity Detection sensitivity (0.0-1.0) |

#### Advanced Configuration

| Variable | Value | Optional | Description |
|----------|-------|----------|-------------|
| `PROCESSING_TIMEOUT` | `60` | Yes | Processing timeout (seconds) |
| `LOG_LEVEL` | `INFO` | Yes | Logging verbosity (DEBUG, INFO, WARNING, ERROR) |

**Copy-Paste Configuration for A100 80GB (Recommended):**
```bash
MODEL_PRECISION=fp16
DEVICE=cuda
BATCH_SIZE=8
TARGET_SR=16000
MAX_AUDIO_DURATION=30
VAD_THRESHOLD=0.5
PROCESSING_TIMEOUT=60
LOG_LEVEL=INFO
```

**For A100 40GB (if available):**
```bash
MODEL_PRECISION=fp16
DEVICE=cuda
BATCH_SIZE=4
TARGET_SR=16000
MAX_AUDIO_DURATION=30
VAD_THRESHOLD=0.5
LOG_LEVEL=INFO
```

**For L40S or Similar GPUs:**
```bash
MODEL_PRECISION=fp16
DEVICE=cuda
BATCH_SIZE=2
TARGET_SR=16000
MAX_AUDIO_DURATION=30
VAD_THRESHOLD=0.5
LOG_LEVEL=INFO
```

### 5. Select Instance Type

**Instance Configuration:**
- **Type**: GPU Instance
- **GPU**: NVIDIA A100 80GB (production deployment)
- **Memory**: 32GB RAM (recommended for optimal performance)
- **Storage**: 20GB+ (for model cache + working memory)
- **CUDA Version**: 12.8+ (automatically provided in Docker image)

**Parakeet Model Advantages:**
- ✅ **Lightweight**: Only 0.6B parameters (~2-3GB VRAM)
- ✅ **Fast Inference**: 30-50x real-time on A100
- ✅ **High Accuracy**: Comparable to larger models for English
- ✅ **NeMo Integration**: Built on NVIDIA's production toolkit
- ✅ **WebSocket Support**: Real-time streaming transcription

**GPU Comparison for Parakeet:**

| GPU | VRAM | Batch Size | Performance | Cost/Hour | Recommendation |
|-----|------|------------|-------------|-----------|----------------|
| **A100 80GB** | 80GB | 8-16 | 40-60x RT | ~$2.00 | ⭐⭐⭐⭐⭐ Best for high throughput |
| **A100 40GB** | 40GB | 4-8 | 35-50x RT | ~$1.50 | ⭐⭐⭐⭐ Great balance |
| **L40S** | 48GB | 4-8 | 30-45x RT | ~$1.55 | ⭐⭐⭐⭐ Cost-effective |
| **A10G** | 24GB | 2-4 | 20-30x RT | ~$0.80 | ⭐⭐⭐ Budget option |

**Recommended**: A100 80GB for production deployment. The extensive VRAM (80GB) allows for larger batch sizes (8-16), multiple concurrent transcriptions, and maximum throughput with Parakeet's efficient footprint.

### 6. Configure Networking

**Port Configuration:**
- **Port**: `8000` (must match Dockerfile EXPOSE)
- **Protocol**: HTTP
- **Health Check Path**: `/healthz`

**Health Check Settings:**
- Initial delay: 60 seconds (model download time)
- Interval: 30 seconds
- Timeout: 10 seconds
- Success threshold: 1
- Failure threshold: 3

### 7. Advanced Settings (Optional)

**Scaling:**
- Min instances: 1
- Max instances: 1 (or more for auto-scaling)

**Resources:**
- Auto-sleep: Disable (or model reloads each time)

### 8. Deploy!

1. Review all settings
2. Click **"Create Service"**
3. Monitor deployment logs

## Deployment Timeline

Expected deployment time: **5-8 minutes**

1. **Build phase** (~3-4 min)
   - Clone repository
   - Build Docker image with PyTorch + CUDA 12.8
   - Install NeMo toolkit and dependencies
   - Multi-stage build optimization

2. **Model download** (~2-3 min)
   - Download Parakeet-TDT 0.6B v3 model (~2-3GB)
   - Initialize NeMo ASR model
   - Load into GPU memory

3. **Service start** (~1 min)
   - Initialize FastAPI
   - Start WebSocket support
   - Run health checks
   - Service becomes available

## Monitoring Deployment

### View Logs

In Koyeb dashboard:
1. Click on your service
2. Go to **"Logs"** tab
3. Watch for:
   ```
   Loading Parakeet-TDT model: nvidia/parakeet-tdt-0.6b-v3
   Model loaded successfully on cuda
   Parakeet service ready
   Application startup complete
   ```

### Common Log Messages

**✅ Success:**
```
INFO: Parakeet service ready on cuda
INFO: Model: nvidia/parakeet-tdt-0.6b-v3, Device: cuda, Precision: fp16
INFO: Application startup complete
```

**⚠️ Issues:**
```
ERROR: CUDA out of memory
→ Reduce BATCH_SIZE from 4 to 2

ERROR: Model download failed
→ Check internet connectivity and disk space

WARNING: CUDA not available, using CPU
→ Verify GPU instance was selected (not CPU instance)

ERROR: torch not compiled with CUDA
→ Verify Docker build method was selected (not Buildpack)
```

## Testing the Deployment

### 1. Get Service URL

Once deployed, Koyeb provides a URL:
```
https://your-service-name-your-org.koyeb.app
```

### 2. Health Check

```bash
curl https://your-service-name.koyeb.app/healthz
```

Expected response:
```json
{"status": "ok"}
```

### 3. Test Basic Transcription

```bash
curl -X POST https://your-service-name.koyeb.app/transcribe \
  -F "file=@your_audio.mp3" \
  -F "include_timestamps=false"
```

Expected response:
```json
{
  "text": "Your transcribed text here",
  "timestamps": null,
  "timing": {
    "upload": 0.06,
    "preprocessing": 0.53,
    "vad_chunking": 0.04,
    "model_inference": 0.68,
    "total_server": 1.31
  }
}
```

### 4. Test with Timestamps

```bash
curl -X POST https://your-service-name.koyeb.app/transcribe \
  -F "file=@your_audio.mp3" \
  -F "include_timestamps=true" \
  -F "should_chunk=true"
```

Expected response:
```json
{
  "text": "Your transcribed text here",
  "timestamps": {
    "words": [
      {"text": "Hello", "start": 0.2, "end": 0.5},
      {"text": "world", "start": 0.6, "end": 0.9}
    ],
    "segments": [
      {"text": "Hello world", "start": 0.2, "end": 0.9}
    ]
  },
  "timing": {
    "upload": 0.06,
    "preprocessing": 0.53,
    "vad_chunking": 0.04,
    "model_inference": 1.24,
    "formatting": 0.012,
    "total_server": 1.85
  }
}
```

### 5. Test OpenAI-Compatible Endpoint

```bash
curl -X POST https://your-service-name.koyeb.app/audio/transcriptions \
  -F "file=@your_audio.mp3"
```

### 6. Test WebSocket Streaming (Optional)

WebSocket endpoint: `wss://your-service-name.koyeb.app/ws`

**JavaScript Example:**
```javascript
const ws = new WebSocket("wss://your-service-name.koyeb.app/ws");

ws.onopen = () => {
  console.log("WebSocket connected");
  // Send 16kHz mono PCM audio data
  ws.send(audioBuffer);
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log("Transcription:", data.text);
  console.log("Is final:", data.is_final);
};
```

## Updating the Service

### Method 1: Git Push (Auto-Deploy Enabled)

```bash
# Make changes locally
git add .
git commit -m "Update service"
git push origin main
```

Koyeb automatically rebuilds and deploys.

### Method 2: Manual Redeploy

1. Go to Koyeb dashboard
2. Click service → **"Redeploy"**
3. Optionally update environment variables

### Method 3: Update Environment Variables Only

1. Go to service settings
2. Update environment variables
3. Service restarts automatically

## Troubleshooting

### CUDA Out of Memory

**Problem:** `RuntimeError: CUDA out of memory`

**Solutions:**
1. **Reduce BATCH_SIZE**:
   - Try: 8 → 4 → 2 (for A100 80GB)
2. **Check model precision**:
   - Verify `MODEL_PRECISION=fp16` (not fp32)
3. **Verify GPU allocation**:
   - Ensure A100 80GB was selected
4. **Restart service**:
   - Clear any memory leaks

**Expected VRAM Usage on A100 80GB (Parakeet):**
- Model: ~2-3GB
- Batch processing (size 8): ~4-7GB
- Total: ~7-10GB (plenty of headroom on 80GB)
- Available for scaling: 70GB+ free for increased batch sizes or concurrent processing

### Slow Inference

**Problem:** Transcription takes too long

**Solutions:**
1. **Verify GPU usage**: Check logs for "cuda" device
2. **Optimize batch size**: 
   - Try increasing to `BATCH_SIZE=8` for A100
3. **Disable unnecessary features**: 
   - Set `should_chunk=false` for short audio
4. **Check audio length**: Very long audio takes proportionally longer
5. **Use WebSocket**: Real-time streaming for live audio

**Expected Performance on A100 40GB:**
- Short audio (<30s): 0.3-0.8 seconds
- Medium audio (1-5 min): 1-4 seconds
- Long audio (10+ min): 5-15 seconds
- Real-time factor: 30-50x

### Service Keeps Restarting

**Problem:** Service restarts repeatedly

**Solutions:**
1. **Increase health check timeout**: Set to 60 seconds (model load time)
2. **Check Docker build**: Verify "Docker" was selected, not Buildpack
3. **Verify port**: Ensure port 8000 is exposed
4. **Check logs**: Look for startup errors
5. **Ensure sufficient memory**: Allocate at least 16GB RAM
6. **Verify PyTorch installation**: Check for CUDA-enabled torch

### Model Download Fails

**Problem:** Model fails to download on startup

**Solutions:**
1. **Check internet connectivity**: Ensure Koyeb instance can reach Hugging Face
2. **Verify disk space**: Ensure at least 15GB storage allocated
3. **Check model name**: Verify `nvidia/parakeet-tdt-0.6b-v3`
4. **Review logs**: Look for specific error messages
5. **Retry deployment**: Temporary network issues may resolve

### Audio Format Issues

**Problem:** `Unsupported audio format` error

**Solutions:**
1. **Supported formats**: `.wav`, `.mp3`, `.flac`, `.ogg`, `.opus`, `.m4a`
2. **Check file encoding**: Ensure file is not corrupted
3. **Test with different format**: Try converting to `.wav` first
4. **Verify ffmpeg**: Service uses ffmpeg for conversion

**Note**: All audio is automatically converted to 16kHz mono internally.

### WebSocket Connection Issues

**Problem:** WebSocket fails to connect or drops

**Solutions:**
1. **Use WSS protocol**: `wss://` not `ws://` for production
2. **Send correct format**: 16kHz mono PCM int16 audio
3. **Check frame size**: Send audio in reasonable chunks (e.g., 1024 samples)
4. **Monitor connection**: Handle reconnection logic in client
5. **Check logs**: Review server-side WebSocket errors

## Performance Optimization

### GPU Tuning Tips

1. **Adjust batch size**: Balance throughput vs latency
   - Higher batch = better throughput but longer per-request latency
   - Lower batch = lower latency but reduced throughput
2. **Enable VAD chunking**: Reduces processing by skipping silence
3. **Use fp16 precision**: 2x faster than fp32 with minimal accuracy loss
4. **Monitor VRAM**: Parakeet uses minimal VRAM, can increase batch size
5. **Profile timing**: Use timing breakdown to identify bottlenecks

### Expected Throughput

**A100 80GB Performance (Production):**
- Real-time factor: 50-70x (e.g., 1 min audio → 0.8-1.2 sec)
- Concurrent requests: 4-8 (depending on batch size)
- Peak VRAM: ~10GB with batch size 8 (leaves 70GB free)
- Latency: 0.3-1.5 seconds for typical audio
- Maximum throughput: Can handle multiple simultaneous streams

**A100 40GB Performance:**
- Real-time factor: 30-50x (e.g., 1 min audio → 1.2-2 sec)
- Concurrent requests: 2-4 (depending on batch size)
- Peak VRAM: ~7GB (leaves 33GB free)
- Latency: 0.5-2 seconds for typical audio

**L40S Performance:**
- Real-time factor: 25-40x
- Concurrent requests: 2-3
- Peak VRAM: ~7GB (leaves 41GB free)
- Latency: 0.7-2.5 seconds

## Cost Optimization

### Reduce Costs

1. **Optimize batch size**: With A100 80GB, increase to 12-16 for maximum throughput
2. **Auto-sleep**: Enable for development (model reloads on wake)
3. **Monitor usage**: Track actual GPU utilization to justify A100 80GB
4. **Consider alternatives**: If low traffic, A100 40GB or L40S may suffice
5. **Implement queuing**: Batch multiple requests together efficiently

### Estimated Costs

**A100 80GB Pricing** (approximate):
- ~$2.00/hour
- ~$48/day (if running 24/7)
- ~$1,440/month (if running 24/7)

**A100 40GB Pricing** (approximate):
- ~$1.50/hour
- ~$36/day (if running 24/7)
- ~$1,080/month (if running 24/7)

**L40S Pricing** (approximate):
- ~$1.55/hour
- ~$37/day (if running 24/7)
- ~$1,116/month (if running 24/7)

**Cost per transcription on A100 80GB** (estimated):
- 1 minute audio: $0.0006-0.0008 (faster processing)
- 10 minute audio: $0.006-0.008
- 1 hour audio: $0.04-0.06

**Note**: A100 80GB provides better cost-per-transcription due to higher throughput and ability to handle more concurrent requests.

*Note: Actual costs vary by provider and usage patterns. Check Koyeb pricing.*

## Security Best Practices

### ✅ Current Security Measures

- ✅ No authentication tokens required
- ✅ HTTPS enabled by default on Koyeb
- ✅ No sensitive data in logs
- ✅ Public model (no access restrictions)
- ✅ WebSocket secure (WSS) support

### 🔒 Additional Recommendations

1. **Add API authentication**: Implement API key authentication if needed
2. **Rate limiting**: Prevent abuse with request rate limits
3. **Input validation**: Service validates file size and format
4. **Monitor access**: Check Koyeb logs for unusual activity
5. **Network security**: Use Koyeb's built-in DDoS protection
6. **WebSocket authentication**: Add token-based auth for streaming

## Monitoring & Metrics

### Built-in Metrics

The service provides detailed timing breakdowns:
- `upload`: File upload time
- `preprocessing`: Audio conversion time
- `vad_chunking`: Voice Activity Detection time
- `model_inference`: GPU processing time
- `formatting`: Response formatting time
- `total_server`: End-to-end server time

### Monitoring Tools

1. **Koyeb Dashboard**: View service health, logs, metrics
2. **Custom monitoring**: Add DataDog, Prometheus, etc.
3. **Log aggregation**: Export logs to external service
4. **WebSocket analytics**: Monitor streaming session metrics

## Features Comparison

### REST API vs WebSocket

**REST API (`/transcribe`):**
- ✅ Best for: File uploads, batch processing
- ✅ Supports: All audio formats via multipart upload
- ✅ Returns: Complete transcription with timestamps
- ❌ Latency: Higher (wait for full file)

**WebSocket (`/ws`):**
- ✅ Best for: Real-time streaming, live audio
- ✅ Supports: 16kHz mono PCM int16 audio
- ✅ Returns: Partial + final transcriptions
- ❌ Format: Raw audio bytes only

## Support

### Getting Help

- **Parakeet Model**: https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3
- **NeMo Toolkit**: https://github.com/NVIDIA/NeMo
- **Koyeb Platform**: https://www.koyeb.com/docs
- **Repository Issues**: [Your repository issues URL]

### Useful Commands

```bash
# Test locally with Docker
docker-compose up --build

# Check git status
git status

# View recent commits
git log --oneline -5

# Test REST API locally
curl -X POST http://localhost:8000/transcribe \
  -F "file=@test_audio.mp3" \
  -F "include_timestamps=true"

# Test WebSocket locally (requires wscat)
wscat -c ws://localhost:8000/ws
```

## Next Steps

After successful deployment:

1. ✅ Test all API endpoints (REST + WebSocket)
2. ✅ Configure custom domain (optional)
3. ✅ Set up monitoring/alerts
4. ✅ Document API for users
5. ✅ Optimize performance based on usage patterns
6. ✅ Consider implementing API authentication
7. ✅ Set up auto-scaling if needed
8. ✅ Benchmark WebSocket streaming performance

## Quick Reference

### Environment Variables Summary

**Minimal (Required):**
```bash
DEVICE=cuda
MODEL_PRECISION=fp16
BATCH_SIZE=4
```

**Recommended (Production on A100 80GB):**
```bash
DEVICE=cuda
MODEL_PRECISION=fp16
BATCH_SIZE=8
TARGET_SR=16000
MAX_AUDIO_DURATION=30
VAD_THRESHOLD=0.5
PROCESSING_TIMEOUT=60
LOG_LEVEL=INFO
```

### API Endpoints

- `GET /` - Service info
- `GET /healthz` - Health check
- `POST /transcribe` - Main transcription (REST)
- `POST /audio/transcriptions` - OpenAI-compatible
- `WS /ws` - WebSocket streaming
- `GET /debug/config` - Configuration

### Model Information

- **Name**: nvidia/parakeet-tdt-0.6b-v3
- **Parameters**: 0.6 billion
- **Language**: English (optimized)
- **Architecture**: Transformer-based ASR
- **Framework**: NeMo Toolkit + PyTorch
- **VRAM**: ~2-3GB base + batch overhead
- **Sample Rate**: 16kHz native

---

**Deployment Status**: Ready for Koyeb deployment
**Last Updated**: 2025-10-27
**Model**: Parakeet-TDT 0.6B v3 (nvidia/parakeet-tdt-0.6b-v3)
**Features**: REST API + WebSocket Streaming
