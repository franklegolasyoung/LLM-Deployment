# My Experience Deploying DeepSeek-R1 Locally with Ollama (RTX 3070 + i7-10700K)

Recently, I wanted to try running DeepSeek-R1 locally on my own machine —  
I have a RTX 3070 (8GB VRAM) and an Intel i7-10700K (16 threads).  
Originally, I thought it might be a hassle because big models usually require A100s or expensive cloud servers.  
But surprisingly, using Ollama, the whole setup turned out to be quite manageable.

Here’s my full experience — including the good, the bad, and a few small issues I encountered.

## My Machine Specs

- GPU: NVIDIA GeForce RTX 3070 8GB  
- CPU: Intel Core i7-10700K (3.8 GHz, 16 threads)  
- RAM: 32GB DDR4  
- OS: Ubuntu 22.04 LTS

## 1. Setting Up the Environment

### 1.1 Installing GPU Drivers

First, I double-checked my NVIDIA driver version:

```bash
nvidia-smi
```

It showed:

```
Driver Version: 535.113.01    CUDA Version: 12.2
```

As long as your driver is above 530 and CUDA is 12.x, you’re fine.  
If it's too old, you'll probably want to upgrade the driver first.

Tip: Ubuntu default drivers can sometimes be old. Manually installing from the NVIDIA website saved me some trouble.

### 1.2 Installing Ollama

Next, I installed Ollama.  
This tool honestly saved me a ton of setup pain — you don’t need to mess with HuggingFace manually, don’t need to quantize models yourself.

Installation was straightforward:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

After installing, I started the Ollama server:

```bash
ollama serve
```

It printed:

```
[INFO] Ollama server started at http://localhost:11434
```

At this point, Ollama just sits quietly in the background, ready to serve models.

## 2. Deploying DeepSeek-R1

### 2.1 Pulling the Model

To get the model, I ran:

```bash
ollama pull deepseek-r1
```

It took about 5–10 minutes because the model file is pretty big (~5–6GB even in quantized 4-bit format).

The output looked like this:

```
pulling manifest...
pulling layers...
verifying sha256...
successfully pulled deepseek-r1
```

Small thing to note:  
If you have slow internet, the pull process might timeout once or twice — I had to retry once.

### 2.2 Running DeepSeek-R1

After the pull finished, I simply launched it:

```bash
ollama run deepseek-r1
```

First launch took about 30 seconds, because it needs to load into GPU memory.

After loading, it said:

```
[INFO] deepseek-r1 ready for input
```

Now I could start chatting directly in the terminal.

Example conversation:

```bash
> What's the capital of Japan?

Tokyo is the capital city of Japan.
```

Pretty fast reply.  
Latency feels around 1 second per sentence — not instant like GPT-4, but very usable.

## 3. API and Automation

I also tested using curl to interact programmatically.

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "deepseek-r1",
  "prompt": "Explain the concept of overfitting in machine learning."
}'
```

Result:

```json
{
  "response": "Overfitting occurs when a machine learning model learns the training data too well and performs poorly on unseen data..."
}
```

Later I also installed the Python SDK:

```bash
pip install ollama
```

and used a quick script:

```python
import ollama

response = ollama.generate(model='deepseek-r1', prompt='What is gradient descent?')
print(response['response'])
```

Python integration is really clean. No messy setup.

## 4. GPU and Performance Observations

I checked GPU usage with `nvidia-smi`:

```
GPU Memory Usage: 7155MiB
```

- Memory: ~7.2GB (safe margin under 8GB VRAM)  
- Inference Speed: about 10–15 tokens/sec (depends on prompt size)  
- CPU: only moderate usage (~20–30%)

Honestly, for a 3070, this is very decent.  
The model uses 4-bit quantization (Q4) automatically, so it fits into smaller VRAM.

## 5. Small Issues / Things to Know

| Issue                     | My Experience                                                                 |
|---------------------------|--------------------------------------------------------------------------------|
| Model Load Time           | First launch takes ~30s. Later launches are faster.                           |
| Internet Drop during Pull | If pull fails midway, just rerun `ollama pull deepseek-r1`. It resumes.       |
| Token Limit               | DeepSeek-R1 in Ollama is configured with a ~4k context window.                |
| Memory Management         | If you open other heavy GPU apps (like Chrome with video playing), Ollama might crash. Restart it after freeing memory. |

### Final Thoughts

- If you just want to run DeepSeek-R1 easily without docker / huggingface complexity, Ollama is really a great choice.  
- RTX 3070 handles it surprisingly well if you don't expect huge batch generation.  
- DeepSeek-R1’s responses are decent for reasoning tasks — definitely usable for building local projects.

Overall, setting up took me about 20 minutes from scratch.  
Very satisfied with how easy it is now to run LLMs at home without spending hundreds on cloud GPUs.

## 6. Access DeepSeek-R1 Through a Web UI (So Much Better Than CLI)

Although terminal chatting works,  
after a few conversations, I realized I really needed a visual page —  
like ChatGPT style, where I could see previous chats properly.

So I set up a Web UI on top of Ollama, and honestly it makes the whole experience 10x better.

## 7. Setting Up Web UI for DeepSeek-R1

### 7.1 Install Open WebUI

I found a project called Open WebUI, which works perfectly with Ollama.

First, I pulled the Docker image:

```bash
docker pull ghcr.io/open-webui/open-webui:main
```

Then I ran it with:

```bash
docker run -d --name open-webui \
  -p 3000:3000 \
  -e OLLAMA_API_BASE_URL=http://localhost:11434 \
  ghcr.io/open-webui/open-webui:main
```

Note: If you're on Windows WSL2 or macOS, you might need to use `host.docker.internal` instead of `localhost`.

After a few seconds, I could visit:

```
http://localhost:3000
```

and got a beautiful ChatGPT-like interface.  
On the left side, you can create new chats, and at the top, you can choose models.

### 7.2 Connect to DeepSeek-R1

Once the page opened,  
I clicked the model selection dropdown at the top,  
and DeepSeek-R1 was already available because it was loaded into Ollama.

I selected `deepseek-r1` from the list and started chatting immediately.

## 8. Making It Automatic: Startup Ollama + WebUI Together

Typing two commands every time was annoying. So I decided to automate it.

### 8.1 Write a Simple Startup Script

I created a bash script called `start_ollama_webui.sh`:

```bash
#!/bin/bash

# Start Ollama server
ollama serve &

# Give Ollama a few seconds to start
sleep 5

# Start Open WebUI container
docker start open-webui
```

Then made it executable:

```bash
chmod +x start_ollama_webui.sh
```

Now every time I reboot, I just run:

```bash
./start_ollama_webui.sh
```

You can also add this script to your OS startup apps if you want it even more automatic.

### 8.2 Docker Auto-Restart

To make sure Open WebUI auto-starts if Docker restarts:

```bash
docker update --restart unless-stopped open-webui
```

Now, even if the machine reboots, Docker will restart WebUI automatically.

## 9. How to Switch Models Easily in WebUI

One thing I really like:  
If I pull multiple models with Ollama, I can easily switch between them in WebUI.

Example:

```bash
ollama pull llama3
ollama pull codellama
```

After pulling, they show up in the model list automatically.  
No extra configuration needed.

Switching is instantaneous — just select from the dropdown.

This is very useful if you want to compare DeepSeek-R1's performance with other models like LLaMA-3 or CodeLLaMA.

## Final Conclusion

After a few days of using this setup,  
I can honestly say — running DeepSeek-R1 locally feels almost as good as using ChatGPT,  
but everything is under my control, no API cost, no privacy issues.

My current daily workflow:

- Start PC  
- Ollama + WebUI auto-start  
- Open browser  
- Chat locally with DeepSeek-R1 instantly

RTX 3070 + i7-10700K is fully enough for running the 7B version with reasonable speed (~10–15 tokens/sec).  
The WebUI interface makes it super friendly for casual chatting, writing, even brainstorming ideas.

If you have a similar hardware setup,  
I highly recommend trying this — it feels amazing to have your own private LLM server running at home.

## References

- [Ollama - Official Site](https://ollama.com/)
- [Open WebUI GitHub](https://github.com/open-webui/open-webui)
- [DeepSeek-R1 Ollama Library](https://ollama.com/library/deepseek-r1)
- [DeepSeek AI on Huggingface](https://huggingface.co/deepseek-ai)
