# Windows Computer Use Agent (WCUA) - Premium Specification v2.0
## Imagine Cup 2025 Submission

---

## Executive Summary

**Project Name:** Windows Computer Use Agent (WCUA) Premium v2.0  
**Submission Date:** January 5, 2025  
**Target Market:** Enterprise automation, accessibility, remote work  
**Team:** Deepen (Backend/API), Teja (Frontend/Mobile)  
**Duration:** 5 Days (Dec 31, 2024 - Jan 5, 2025)

### Core Value Proposition

WCUA enables users to control Windows computers through natural language commands via mobile apps, voice calls (IVR), or web interfaces. The agent performs clicks, keystrokes, and complex workflows without user intervention—perfect for accessibility, accessibility needs, and enterprise automation.

---

## 1. Architecture Overview

### 1.1 System Layers

```
┌─────────────────────────────────────────────────────┐
│         User Interfaces (Multi-Platform)            │
│  Mobile App (iOS/Android) | Web Dashboard | IVR     │
└────────────────┬──────────────────────────────────┘
                 │
┌────────────────▼──────────────────────────────────┐
│      API Gateway & Authentication Layer             │
│  Auth0/Azure AD | Rate Limiting | Request Router   │
└────────────────┬──────────────────────────────────┘
                 │
┌────────────────▼──────────────────────────────────┐
│      LLM Orchestration Service (Cloud/Local)       │
│  Azure OpenAI | OpenAI | Ollama (Local)           │
└────────────────┬──────────────────────────────────┘
                 │
┌────────────────▼──────────────────────────────────┐
│     Agent Runtime & Action Executor                │
│  Screen Analysis | Action Planning | Execution    │
└────────────────┬──────────────────────────────────┘
                 │
┌────────────────▼──────────────────────────────────┐
│      Windows Integration Layer                      │
│  PyAutoGUI | Win32API | Screenshot Capture        │
└─────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Backend API** | FastAPI (Python) | Async, easy deployment, excellent for agent workflows |
| **Mobile App** | React Native / Flutter | Cross-platform (iOS/Android) |
| **Web Dashboard** | React.js + Next.js | Real-time monitoring, responsive UI |
| **IVR System** | Twilio or Asterisk | Voice command integration |
| **LLM Integration** | Azure OpenAI / Ollama | Cloud scalability + local privacy option |
| **Database** | PostgreSQL | Reliable, JSONB support for complex workflows |
| **Cache** | Redis | Session management, rate limiting |
| **Agent Runtime** | Python + Async Tasks | Windows automation, lightweight |
| **Screen Capture** | Pillow + win32grab | Real-time visual feedback |
| **Desktop Automation** | PyAutoGUI + keyboard | Cross-app keyboard/mouse control |
| **Deployment** | Docker + Azure Container Instances | Scalable, enterprise-ready |
| **Monitoring** | Datadog / Azure Monitor | Production observability |

---

## 2. Detailed Module Specifications

### 2.1 Authentication & Authorization Module

**Responsibilities:**
- User registration and login
- API key generation for mobile/IVR
- Role-based access control (RBAC)
- Session management
- MFA support

**Key Endpoints:**
```
POST   /api/v1/auth/register          # User registration
POST   /api/v1/auth/login             # User login
POST   /api/v1/auth/mfa/verify        # MFA verification
POST   /api/v1/auth/refresh-token     # Token refresh
GET    /api/v1/auth/me                # Current user info
POST   /api/v1/auth/api-keys          # Generate API keys
DELETE /api/v1/auth/api-keys/{key_id} # Revoke API key
```

**Database Schema:**
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    full_name VARCHAR(255),
    phone_number VARCHAR(20),
    is_mfa_enabled BOOLEAN DEFAULT FALSE,
    mfa_secret VARCHAR(255),
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE api_keys (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    key_hash VARCHAR(255),
    name VARCHAR(255),
    scopes JSONB,
    last_used_at TIMESTAMP,
    created_at TIMESTAMP,
    expires_at TIMESTAMP
);

CREATE TABLE sessions (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    token VARCHAR(500),
    device_type VARCHAR(50),
    ip_address INET,
    created_at TIMESTAMP,
    expires_at TIMESTAMP
);
```

### 2.2 Command Processing Module

**Responsibilities:**
- Parse user commands from multiple sources
- Validate command syntax and constraints
- Route commands to appropriate handlers
- Maintain command history and logs

**Command Processing Pipeline:**

```
Raw Input (Mobile/IVR/Web)
         ↓
Command Parsing & Validation
         ↓
Safety Checks (Blocked Apps, Permissions)
         ↓
Rate Limiting Check
         ↓
Queue to Message Broker (Redis/RabbitMQ)
         ↓
Agent Runtime Picks Up
         ↓
LLM Analysis & Planning
         ↓
Action Execution
         ↓
Result Logging & User Notification
```

**Key Endpoints:**
```
POST   /api/v1/commands/execute       # Submit new command
GET    /api/v1/commands/{cmd_id}      # Get command status
GET    /api/v1/commands/history       # Command history
DELETE /api/v1/commands/{cmd_id}      # Cancel command
POST   /api/v1/commands/{cmd_id}/approve # Approve sensitive commands
```

**Database Schema:**
```sql
CREATE TABLE commands (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    command_text TEXT NOT NULL,
    source VARCHAR(50), -- 'mobile', 'ivr', 'web'
    status VARCHAR(50) DEFAULT 'pending', -- pending, approved, executing, completed, failed
    estimated_duration_seconds INTEGER,
    actual_duration_seconds INTEGER,
    result_summary TEXT,
    result_screenshots JSONB,
    created_at TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    requires_approval BOOLEAN DEFAULT FALSE,
    approved_by_user BOOLEAN,
    approved_at TIMESTAMP
);

CREATE TABLE command_logs (
    id UUID PRIMARY KEY,
    command_id UUID REFERENCES commands(id),
    log_level VARCHAR(50), -- info, warning, error
    message TEXT,
    timestamp TIMESTAMP
);

CREATE TABLE command_blacklist (
    id UUID PRIMARY KEY,
    pattern VARCHAR(255),
    reason TEXT,
    created_at TIMESTAMP
);
```

### 2.3 LLM Integration Module

**Responsibilities:**
- Connect to Azure OpenAI or Ollama
- Generate action plans from user commands
- Analyze screenshots for contextual understanding
- Handle model fallback and error recovery

**LLM Configuration:**
```json
{
  "providers": {
    "cloud": {
      "type": "azure_openai",
      "endpoint": "https://<resource>.openai.azure.com/",
      "api_key": "env:AZURE_OPENAI_API_KEY",
      "model": "gpt-4-vision",
      "deployment_name": "gpt-4-vision",
      "max_tokens": 2000
    },
    "local": {
      "type": "ollama",
      "base_url": "http://localhost:11434",
      "model": "llava-phi", // or "neural-chat"
      "timeout_seconds": 120
    }
  },
  "default_provider": "cloud",
  "fallback_provider": "local"
}
```

**Key Prompts:**

*System Prompt:*
```
You are WCUA - Windows Computer Use Agent. Your role is to:
1. Understand user intentions from natural language
2. Analyze Windows screen screenshots provided
3. Generate precise, step-by-step action sequences
4. Each action must be one of: click(x, y), type(text), key(key_name), wait(seconds), screenshot()
5. Consider UI elements, current application state, and user intent
6. Generate the SHORTEST sequence possible
7. Include error recovery if needed
8. Output ONLY valid JSON in specified format

Current Screen Context:
- Application: {app_name}
- Window Title: {window_title}
- Screen Resolution: {screen_width}x{screen_height}
- Active Foreground Window: {foreground_window}
```

*User Request Transformation:*
```json
{
  "user_command": "Send an email to john@example.com saying hello world",
  "context": {
    "current_app": "outlook",
    "current_screen": "<base64_screenshot>",
    "previous_actions": ["opened outlook", "clicked compose"],
    "user_email": "user@example.com"
  },
  "request": "Generate action sequence as JSON array"
}
```

### 2.4 Screen Analysis & OCR Module

**Responsibilities:**
- Capture current screen state
- Perform OCR on text elements
- Detect UI components (buttons, fields, etc.)
- Maintain screen history for context

**Implementation:**
```python
from PIL import ImageGrab
import pytesseract
from ultralytics import YOLO

class ScreenAnalyzer:
    def __init__(self):
        self.yolo_model = YOLO('yolov8n.pt')
        
    def capture_screen(self):
        """Capture current screen"""
        screenshot = ImageGrab.grab()
        return screenshot
    
    def extract_text_ocr(self, image):
        """Extract text from screen using OCR"""
        text = pytesseract.image_to_string(image)
        return text
    
    def detect_ui_elements(self, image):
        """Detect buttons, fields, etc using YOLO"""
        results = self.yolo_model(image)
        return results
    
    def get_clickable_elements(self, image):
        """Identify and return coordinates of clickable elements"""
        # Returns: [{'type': 'button', 'text': 'Save', 'x': 100, 'y': 50, 'width': 80, 'height': 30}]
        pass
```

### 2.5 Action Execution Module

**Responsibilities:**
- Execute clicks, keystrokes, and other actions
- Handle error recovery
- Provide feedback on execution
- Maintain action history

**Supported Actions:**

```python
class ActionExecutor:
    async def execute_action(self, action: dict):
        """
        Execute a single action
        
        action format:
        {
            "type": "click|type|key|wait|screenshot|hotkey",
            "x": int,           # for click
            "y": int,           # for click
            "text": str,        # for type
            "key": str,         # for key (enter, tab, escape, etc)
            "keys": [str],      # for hotkey (e.g., ['ctrl', 'c'])
            "seconds": float    # for wait
        }
        """
        
    async def execute_sequence(self, actions: list):
        """Execute multiple actions in sequence with error recovery"""
        
    async def verify_action_result(self, action: dict, before_screen, after_screen):
        """Verify if action achieved expected result"""
```

**Key Implementation:**
```python
import pyautogui
import keyboard
from time import sleep

async def click(x: int, y: int, duration: float = 0.1):
    """Move mouse and click at coordinates"""
    pyautogui.moveTo(x, y, duration=0.2)
    pyautogui.click()
    await asyncio.sleep(0.5)

async def type_text(text: str):
    """Type text character by character"""
    pyautogui.typewrite(text, interval=0.05)
    await asyncio.sleep(0.3)

async def press_key(key: str):
    """Press a single key"""
    key_map = {
        'enter': 'return',
        'tab': 'tab',
        'escape': 'esc',
        'backspace': 'backspace',
        'delete': 'delete',
        'shift': 'shift',
        'ctrl': 'ctrl',
        'alt': 'alt'
    }
    keyboard.press(key_map.get(key.lower(), key))
    await asyncio.sleep(0.2)

async def hotkey(keys: list):
    """Press multiple keys simultaneously"""
    keyboard.press_and_release('+'.join(keys))
    await asyncio.sleep(0.3)
```

### 2.6 Mobile App Module (React Native)

**Key Screens:**

1. **Authentication Flow**
   - Login/Register
   - API Key Display
   - MFA Setup

2. **Command Interface**
   - Text Input (natural language)
   - Voice Input (speech-to-text)
   - Command History
   - Real-time Status Updates

3. **Desktop Preview**
   - Live screenshot from target computer
   - Command execution progress
   - Error notifications

4. **Settings**
   - Ollama URL configuration
   - Default LLM provider selection
   - Security settings

**Key Libraries:**
```json
{
  "react-native": "^0.71.0",
  "react-navigation": "^6.x",
  "@react-navigation/native": "^6.x",
  "axios": "^1.4.0",
  "react-native-voice": "^3.2.2",
  "react-native-webview": "^12.0.0",
  "socket.io-client": "^4.5.0"
}
```

### 2.7 IVR Integration Module

**Responsibilities:**
- Handle incoming voice calls
- Convert speech to text
- Execute commands via voice
- Provide voice feedback

**Twilio Integration:**
```python
from twilio.rest import Client
from twilio.twiml.voice_response import VoiceResponse

class IVRHandler:
    def __init__(self):
        self.client = Client(TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN)
    
    async def handle_incoming_call(self, call_sid: str, from_number: str):
        """Handle incoming call from user"""
        response = VoiceResponse()
        response.say("Welcome to Windows Computer Use Agent")
        response.gather(
            num_digits=1,
            timeout=30,
            speech_timeout='auto',
            hints="open browser, send email, create document"
        )
        return response
    
    async def process_speech_input(self, speech_text: str, user_id: str):
        """Process voice command and execute"""
        # Same as text command but with voice output
        pass
```

### 2.8 Monitoring & Logging Module

**Responsibilities:**
- Log all actions for audit
- Monitor system health
- Track performance metrics
- Alert on failures

**Key Metrics:**
- Command execution success rate
- Average execution time
- API response time
- LLM token usage
- System resource utilization

---

## 3. Security & Compliance

### 3.1 Security Layers

1. **Authentication**
   - OAuth2 / OpenID Connect
   - API key with rate limiting
   - MFA (TOTP/SMS)

2. **Authorization**
   - RBAC (Role-Based Access Control)
   - Command approval workflows for sensitive operations
   - IP whitelisting

3. **Data Protection**
   - End-to-end encryption for sensitive data
   - Screenshots stored securely with TTL
   - GDPR compliance (data retention policies)

4. **Action Whitelisting**
   - Blacklist dangerous applications
   - Restrict commands affecting system security
   - Require explicit approval for financial apps

### 3.2 Blocked Operations

```json
{
  "blocked_applications": [
    "C:\\Windows\\System32\\cmd.exe",
    "C:\\Windows\\System32\\powershell.exe",
    "C:\\Windows\\System32\\regedit.exe"
  ],
  "blocked_patterns": [
    "delete.*system32",
    "format.*drive",
    "uninstall.*antivirus"
  ],
  "requires_approval": [
    "financial.*transfer",
    "password.*reset",
    "admin.*command"
  ]
}
```

---

## 4. Deployment & Infrastructure

### 4.1 Azure Deployment (Primary)

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/wcua
      - REDIS_URL=redis://redis:6379
      - AZURE_OPENAI_API_KEY=${AZURE_OPENAI_API_KEY}
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=wcua
      - POSTGRES_USER=wcua_user
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  agent-runner:
    build: ./agent
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/wcua
      - REDIS_URL=redis://redis:6379
      - OLLAMA_BASE_URL=${OLLAMA_BASE_URL}
    depends_on:
      - postgres
      - redis

volumes:
  postgres_data:
```

### 4.2 Local Ollama Setup

```bash
# Installation
curl https://ollama.ai/install.sh | sh

# Start Ollama service
ollama serve

# Pull model in another terminal
ollama pull llava-phi
# Or
ollama pull neural-chat

# Verify
curl http://localhost:11434/api/tags
```

---

## 5. Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| **Command Processing Time** | < 5 seconds | From submission to action execution |
| **Screenshot Capture** | < 500ms | Real-time visual feedback |
| **Action Execution** | < 100ms per action | Responsive UI interaction |
| **API Response Time** | < 500ms | 95th percentile |
| **Command Success Rate** | > 95% | Acceptable error recovery |
| **System Availability** | 99.5% | Production SLA |
| **Concurrent Commands** | 100+ | Per instance |

---

## 6. Success Metrics for Imagine Cup

1. **Technical Excellence**
   - Multi-platform accessibility (mobile, web, voice)
   - Hybrid cloud/local LLM support
   - Robust error handling and recovery

2. **User Experience**
   - Intuitive natural language interface
   - Real-time visual feedback
   - Fast command execution

3. **Innovation**
   - Vision-based screen understanding
   - Multi-modal input (text, voice, UI)
   - Accessibility focus

4. **Business Impact**
   - Reduced manual work by 70%+
   - Enterprise automation use cases
   - Clear monetization path

5. **Code Quality**
   - Clean architecture
   - Comprehensive error handling
   - Well-documented codebase

---

## 7. Post-Launch Roadmap

**Phase 2 (Month 2-3):**
- Advanced workflow automation
- Integration with popular enterprise apps (SAP, Salesforce, etc.)
- Analytics dashboard

**Phase 3 (Month 4-6):**
- Team collaboration features
- API for third-party integrations
- Advanced security features (blockchain audit log)

---

## 8. References & Resources

- Microsoft Imagine Cup Guidelines: https://imaginecup.microsoft.com/
- FastAPI Documentation: https://fastapi.tiangolo.com/
- Ollama GitHub: https://github.com/ollama/ollama
- Azure OpenAI: https://learn.microsoft.com/en-us/azure/cognitive-services/openai/
- PyAutoGUI Documentation: https://pyautogui.readthedocs.io/

---

**Document Version:** 2.0  
**Last Updated:** December 31, 2024  
**Status:** Ready for Development
