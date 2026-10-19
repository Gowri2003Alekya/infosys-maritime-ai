🚢 **Agentic AI for Maritime Freight Pricing & Route Optimization**
A secure, multi-tier Streamlit authentication portal featuring JWT-based session state management, SQLite credential persistence, rate-limited email OTP verification, and role-based views (Administrator vs. Standard Analytics Dashboard).

📌 **Architecture & Authentication Specifications**
All three authentication views reside inside a unified single-file Streamlit application:


1. **Login Page**
--Inputs: Username/Email and Password (both required).

--Authentication: Validates hashed passwords using bcrypt.

--On Success: Issues a signed HS256 JWT session token and transitions directly to the Dashboard.

--On Failure: Displays a single, generic security notice (❌ Invalid credentials.) to prevent username/email enumeration attacks.

2. **Signup Page**
--**Inputs**: Username, Email Address, Password, Confirm Password, Security Question (dropdown), and Security Answer (all required).

--**Complexity Validation**:

    Minimum 8 characters.

    At least one uppercase letter ([A-Z]).

    At least one lowercase letter ([a-z]).

    At least one numeric digit ([0-9]).

    At least one special symbol ([!@#$%^&*(),.?":{}|<>]).

    Confirm Password must match Password identically.

    **Uniqueness**: Username and Email have UNIQUE constraints in SQLite; duplicate signups return a clear warning.


3. **Forgot Password Page**
Supports dual recovery mechanisms:

--**Via Security Question**: Verifies the user against their pre-registered security question and answer hash.

--**Via Email OTP**: Dispatches a cryptographically secure, randomized 6-digit verification code using Gmail SMTP (valid for 5 minutes).

--**Password History Protection**: Prevents users from resetting their password to the current/active password.

4. **Dashboards**
--**Admin Dashboard**: Displays a personalized greeting with the logged-in email and lists all registered users in an interactive table (User ID, Username, Email) without exposing password hashes.

--Standard User Dashboard: Displays session status cards, platform analytics metrics, and real-time health indicator gauges.


⚙️ **Environment Setup & Configuration**
**Step 1**: Set Up ngrok Authentication
ngrok establishes a secure reverse proxy from Google Colab to provide a publicly accessible URL:

1. Sign up for a free account at dashboard.ngrok.com.

2. Navigate to Getting Started > Your Authtoken and copy the personal token.

3. Save this value as NGROK_AUTHTOKEN in Google Colab Secrets (see table below).

**Step 2**: Generate a Gmail App Password
The portal uses Gmail SMTP over TLS (Port 587) to send one-time passcodes:

1. Log in to your Google Account.

2. Navigate to Security and ensure 2-Step Verification is turned ON.

3. Search for "App passwords" in the top search bar.

4. Enter an identifier (e.g., Infosys Portal) and click Create.

5. Copy the 16-character token immediately before dismissing the dialog (Google will not display this token again).

**Step 3**: Configure Google Colab Secrets
To avoid committing credentials in code, save environment parameters in the Secrets manager:

1. Open your notebook in Google Colab.

2. Click the Key icon (🔑 Secrets) in the left sidebar.

3. Add the following secrets and toggle "Notebook access" ON for each:


<img width="2970" height="809" alt="Code_Generated_Image" src="https://github.com/user-attachments/assets/9de254fc-3cd7-4963-adbd-db69492487b4" />

🚀 **Execution Instructions** (Google Colab)
**Cell 1**: Install Dependencies
Run the initial setup cell to download the required libraries:

```Bash
!pip install -q streamlit streamlit-option-menu pyngrok pyjwt bcrypt plotly
```

**Cell 2**: Write Application File (app.py)
Ensure your updated %%writefile app.py cell reads EMAIL_ADDRESS and other secrets dynamically from the runtime environment:

```Python
# SENDER_EMAIL pulls directly from Colab Secrets / Environment
SENDER_EMAIL = os.getenv("EMAIL_ADDRESS", "your-email@gmail.com")
EMAIL_PASSWORD = os.getenv("EMAIL_PASSWORD")
JWT_SECRET = os.getenv("JWT_SECRET", "super-secret-infosys-key-2026")
```
Execute the cell to compile and save app.py directly into /content/app.py.

**Cell 3**: Launch Streamlit & Establish ngrok Tunnel
Run the tunnel manager cell to start Streamlit as a background process and expose the port:

```Python
import os, time, subprocess
from pyngrok import ngrok
from google.colab import userdata

# 1. Safely retrieve secrets from Colab
try:
    os.environ['EMAIL_PASSWORD'] = userdata.get('EMAIL_PASSWORD')
    os.environ['EMAIL_ADDRESS'] = userdata.get('EMAIL_ADDRESS')
    os.environ['JWT_SECRET'] = userdata.get('JWT_SECRET')
    ngrok_token = userdata.get('NGROK_AUTHTOKEN')
    if ngrok_token:
        ngrok.set_auth_token(ngrok_token)
except Exception as e:
    print(f"⚠️ Secret Error: Make sure all keys are set in Colab Secrets: {e}")

# 2. Terminate existing Streamlit processes
!pkill -f streamlit
time.sleep(2)

# 3. Start Streamlit in background on port 8501
process = subprocess.Popen(
    ['streamlit', 'run', 'app.py', '--server.port', '8501'], 
    env=os.environ.copy(), 
    stdout=subprocess.DEVNULL, 
    stderr=subprocess.DEVNULL
)
time.sleep(3)

# 4. Connect / Reuse ngrok tunnel
try:
    tunnels = ngrok.get_tunnels()
    if tunnels:
        public_url = tunnels[0].public_url
        print("♻️ Reusing existing active ngrok tunnel...")
    else:
        public_url = ngrok.connect(8501).public_url
        print("🚀 Created new ngrok tunnel...")

    print("=" * 65)
    print(f"👉 Live Portal URL: {public_url}")
    print("=" * 65)
    print("⏳ Server running! Press [Ctrl + C] or click Colab Stop to terminate.")
    
    while True:
        time.sleep(1)

except KeyboardInterrupt:
    print("\n🛑 Shutting down server...")
    !pkill -f streamlit
    print("✅ Streamlit stopped.")
```
