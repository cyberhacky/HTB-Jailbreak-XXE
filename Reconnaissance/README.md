---

## Reconnaissance

Before interacting with the application, I wanted to identify the exposed services and gather basic information about the target. Although the challenge could be accessed through the browser, performing a quick service enumeration helped confirm the underlying technology stack and identify any additional services that might be available.

I began by running an Nmap scan against the challenge instance.

```bash
nmap -Pn -sV -sC -p31025 <TARGET_IP>
```

### Scan Results

```
PORT      STATE SERVICE VERSION
31025/tcp open  http
Server: Werkzeug/3.0.3 Python/3.12.3
```

The scan confirmed that the challenge was running a web application served by **Werkzeug**, the development server commonly used with **Flask** applications written in Python.

Since only a single HTTP service was exposed, the remainder of the assessment focused on analyzing the web application and identifying any functionality that accepted user-controlled input.
