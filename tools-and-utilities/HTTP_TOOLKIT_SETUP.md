# HTTP Toolkit Setup - Capture HTTP Requests/Responses

## Installation

1. **Download and install HTTP Toolkit**
   - Visit the official page: https://httptoolkit.com/
   - Follow the installation instructions for your operating system

2. **Install Python dependencies** (if using Python)
   ```bash
   pip install httpx
   ```

## Capturing Traffic

### Method 1: Fresh Terminal (Recommended)

1. **Open HTTP Toolkit**

2. **Click "Intercept"** in the top menu

3. **Select "Fresh Terminal"**
   - HTTP Toolkit will open a new terminal window
   - This terminal has proxy settings pre-configured

4. **Run your Python script** from that terminal
   ```bash
   python your_script.py
   ```

5. **View captured traffic** in HTTP Toolkit's main window
   - See full request/response details
   - Headers, body, timing, etc.

### Method 2: Code-Based Proxy (Alternative)

If you prefer to configure the proxy in code:

```python
import httpx

# Configure your HTTP client with the proxy
http_client = httpx.Client(
    proxy="http://127.0.0.1:8000",
    verify=False  # Disable SSL verification for HTTP Toolkit's certificate
)

# Use this client for your HTTP requests
response = http_client.get("https://api.example.com/endpoint")
```

Then run normally:
```bash
python your_script.py
```

## Troubleshooting

- **No traffic showing?** Make sure HTTP Toolkit is running and "Intercept" is active
- **Certificate errors?** Use `verify=False` in the httpx.Client (Method 2) or trust the certificate in HTTP Toolkit
- **Wrong port?** Check HTTP Toolkit's interface for the actual proxy port (usually 8000 or 8001)
