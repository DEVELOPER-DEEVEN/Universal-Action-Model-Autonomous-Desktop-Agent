# Windows Computer Use Agent (WCUA) - Developer Implementation Guide
## Step-by-Step Development Instructions

---

## Project Context

**Submission Deadline:** January 5, 2025 (5 Days Total)  
**Team Members:** 
- **Deepen:** Backend Developer (API, Agent Runtime, LLM Integration)
- **Teja:** Frontend Developer (Mobile App, Web Dashboard, IVR Interface)

---

## Phase 1: Project Setup (Dec 31, 2024 - 6 hours)

### 1.1 Repository Structure (Deepen)

```bash
# Create root project directory
mkdir wcua-premium
cd wcua-premium

# Create subdirectories
mkdir -p backend mobile-app web-dashboard agent-runner config docs scripts

# Backend structure
mkdir -p backend/{app,models,services,utils,routes,middleware,tests}
mkdir -p backend/app/{auth,commands,llm,actions,database}

# Create Python files
touch backend/requirements.txt
touch backend/main.py
touch backend/.env.example
touch backend/Dockerfile

# Frontend structure
mkdir -p mobile-app/{src,components,screens,services,assets}
mkdir -p web-dashboard/{src,components,pages,services,utils}

# Documentation
touch docs/{SETUP.md,API_REFERENCE.md,ARCHITECTURE.md}
touch README.md
```

### 1.2 Initialize Git & Version Control (Deepen)

```bash
cd wcua-premium
git init
git config user.email "deepen@wcua.dev"
git config user.name "Deepen"

# Create .gitignore
cat > .gitignore << 'EOF'
# Python
__pycache__/
*.py[cod]
*$py.class
.Python
env/
venv/
*.egg-info/
.eggs/
dist/
build/

# Environment
.env
.env.local
.env.*.local

# IDEs
.vscode/
.idea/
*.swp
*.swo

# Node
node_modules/
npm-debug.log
yarn-error.log

# OS
.DS_Store
Thumbs.db

# Databases
*.db
*.sqlite3

# Logs
logs/
*.log
EOF

git add .
git commit -m "Initial project setup"
```

### 1.3 Environment Configuration (Deepen)

Create `backend/.env.example`:
```bash
# Database
DATABASE_URL=postgresql://wcua_user:password@localhost:5432/wcua_db
DB_HOST=localhost
DB_PORT=5432
DB_NAME=wcua_db
DB_USER=wcua_user
DB_PASSWORD=your_secure_password

# Redis
REDIS_URL=redis://localhost:6379

# Azure OpenAI (Cloud LLM)
AZURE_OPENAI_API_KEY=your_key_here
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_MODEL=gpt-4-vision
AZURE_OPENAI_DEPLOYMENT=gpt-4-vision

# OpenAI (Alternative)
OPENAI_API_KEY=your_key_here

# Ollama (Local LLM)
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=llava-phi

# Default LLM Provider
DEFAULT_LLM_PROVIDER=cloud  # or 'local'
FALLBACK_LLM_PROVIDER=local

# Application
APP_ENV=development
APP_DEBUG=True
APP_SECRET_KEY=your_secret_key_change_in_production
APP_TITLE=Windows Computer Use Agent
APP_VERSION=2.0

# Server
HOST=0.0.0.0
PORT=8000
WORKERS=4

# Security
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:3001,http://localhost:5173
JWT_SECRET=your_jwt_secret_key
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7

# Twilio (IVR)
TWILIO_ACCOUNT_SID=your_sid
TWILIO_AUTH_TOKEN=your_token
TWILIO_PHONE_NUMBER=+1234567890

# Logging
LOG_LEVEL=INFO
LOG_FORMAT=json

# Features
ENABLE_VOICE_COMMANDS=true
ENABLE_MOBILE_APP=true
ENABLE_WEB_DASHBOARD=true
REQUIRE_COMMAND_APPROVAL=false
```

Copy to `.env`:
```bash
cp backend/.env.example backend/.env
# Edit .env with actual values
```

### 1.4 Docker Setup (Deepen)

Create `docker-compose.yml`:
```yaml
version: '3.9'

services:
  postgres:
    image: postgres:15-alpine
    container_name: wcua_postgres
    environment:
      POSTGRES_DB: wcua_db
      POSTGRES_USER: wcua_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U wcua_user"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: wcua_redis
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    build: ./backend
    container_name: wcua_api
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://wcua_user:${DB_PASSWORD}@postgres:5432/wcua_db
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./backend:/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  agent:
    build: ./agent-runner
    container_name: wcua_agent
    environment:
      DATABASE_URL: postgresql://wcua_user:${DB_PASSWORD}@postgres:5432/wcua_db
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./agent-runner:/app

volumes:
  postgres_data:

networks:
  default:
    name: wcua_network
```

### 1.5 Backend Dependencies (Deepen)

Create `backend/requirements.txt`:
```
# Web Framework
fastapi==0.104.1
uvicorn[standard]==0.24.0
python-multipart==0.0.6

# Database
sqlalchemy==2.0.23
psycopg2-binary==2.9.9
alembic==1.12.1

# Authentication & Security
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-dotenv==1.0.0
PyJWT==2.8.1
pydantic==2.5.0

# Async & Concurrency
aiohttp==3.9.1
asyncio-contextmanager==1.0.0

# Caching
redis==5.0.1
aioredis==2.0.1

# LLM Integration
openai==1.3.6
azure-identity==1.14.0
azure-keyvault-secrets==4.4.0
requests==2.31.0

# Screen Capture & Image Processing
Pillow==10.1.0
pytesseract==0.3.10
pyautogui==0.9.53
keyboard==0.13.5
python-xlib==0.33
PyGetWindow==0.0.9

# Data Validation
pydantic-settings==2.1.0

# Logging & Monitoring
python-json-logger==2.0.7
sentry-sdk==1.38.0

# Testing
pytest==7.4.3
pytest-asyncio==0.21.1
httpx==0.25.2

# Code Quality
black==23.12.0
flake8==6.1.0
isort==5.13.2
mypy==1.7.1

# Utilities
python-dateutil==2.8.2
click==8.1.7
```

---

## Phase 2: Backend Core Development (Jan 1, 2024 - 24 hours)

### 2.1 Database Models (Deepen)

Create `backend/app/models/__init__.py`:
```python
# Export all models for easy import
from .user import User
from .command import Command, CommandLog
from .api_key import APIKey
from .session import Session

__all__ = ['User', 'Command', 'CommandLog', 'APIKey', 'Session']
```

Create `backend/app/models/user.py`:
```python
from sqlalchemy import Column, String, Boolean, DateTime, Enum
from sqlalchemy.dialects.postgresql import UUID
from datetime import datetime
import uuid
import enum

from backend.app.database import Base

class UserRole(str, enum.Enum):
    ADMIN = "admin"
    USER = "user"
    READONLY = "readonly"

class User(Base):
    __tablename__ = "users"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    email = Column(String(255), unique=True, nullable=False, index=True)
    password_hash = Column(String(255), nullable=False)
    full_name = Column(String(255))
    phone_number = Column(String(20))
    role = Column(Enum(UserRole), default=UserRole.USER)
    
    is_mfa_enabled = Column(Boolean, default=False)
    mfa_secret = Column(String(255))
    
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow, index=True)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    def __repr__(self):
        return f"<User {self.email}>"
```

Create `backend/app/models/command.py`:
```python
from sqlalchemy import Column, String, Integer, Text, DateTime, Boolean, ForeignKey, Enum, JSON
from sqlalchemy.dialects.postgresql import UUID
from datetime import datetime
import uuid
import enum

from backend.app.database import Base

class CommandStatus(str, enum.Enum):
    PENDING = "pending"
    APPROVED = "approved"
    EXECUTING = "executing"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"

class CommandSource(str, enum.Enum):
    MOBILE = "mobile"
    WEB = "web"
    IVR = "ivr"

class Command(Base):
    __tablename__ = "commands"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False, index=True)
    
    command_text = Column(Text, nullable=False)
    source = Column(Enum(CommandSource), default=CommandSource.MOBILE)
    status = Column(Enum(CommandStatus), default=CommandStatus.PENDING, index=True)
    
    estimated_duration_seconds = Column(Integer)
    actual_duration_seconds = Column(Integer)
    
    result_summary = Column(Text)
    result_data = Column(JSON)  # Store complex results
    
    requires_approval = Column(Boolean, default=False)
    approved_by_user = Column(Boolean, default=False)
    approved_at = Column(DateTime)
    
    created_at = Column(DateTime, default=datetime.utcnow, index=True)
    started_at = Column(DateTime)
    completed_at = Column(DateTime)

class CommandLog(Base):
    __tablename__ = "command_logs"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    command_id = Column(UUID(as_uuid=True), ForeignKey("commands.id"), nullable=False, index=True)
    
    log_level = Column(String(50))  # info, warning, error, debug
    message = Column(Text)
    timestamp = Column(DateTime, default=datetime.utcnow, index=True)
```

Create `backend/app/models/api_key.py`:
```python
from sqlalchemy import Column, String, DateTime, ForeignKey, JSON
from sqlalchemy.dialects.postgresql import UUID
from datetime import datetime, timedelta
import uuid

from backend.app.database import Base

class APIKey(Base):
    __tablename__ = "api_keys"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False, index=True)
    
    key_hash = Column(String(255), unique=True, nullable=False, index=True)
    name = Column(String(255), nullable=False)
    scopes = Column(JSON, default={"commands": ["read", "write"]})
    
    last_used_at = Column(DateTime)
    created_at = Column(DateTime, default=datetime.utcnow)
    expires_at = Column(DateTime, default=lambda: datetime.utcnow() + timedelta(days=365))
```

### 2.2 Database Configuration (Deepen)

Create `backend/app/database.py`:
```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import NullPool
import os
from typing import Generator

DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://wcua_user:password@localhost:5432/wcua_db")

engine = create_engine(
    DATABASE_URL,
    poolclass=NullPool,
    echo=os.getenv("APP_DEBUG") == "True"
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

def get_db() -> Generator:
    """Dependency for getting database session"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

async def create_tables():
    """Create all tables"""
    Base.metadata.create_all(bind=engine)
```

Create `backend/alembic/env.py` for migrations:
```bash
# Initialize Alembic
cd backend
alembic init alembic

# Create initial migration
alembic revision --autogenerate -m "Initial models"

# Apply migrations
alembic upgrade head
```

### 2.3 Authentication Service (Deepen)

Create `backend/app/services/auth_service.py`:
```python
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from pydantic import BaseModel, EmailStr
import os

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class TokenData(BaseModel):
    user_id: str
    email: str
    exp: datetime

class AuthService:
    def __init__(self):
        self.secret_key = os.getenv("JWT_SECRET", "your_secret_key")
        self.algorithm = os.getenv("JWT_ALGORITHM", "HS256")
        self.access_token_expire = int(os.getenv("ACCESS_TOKEN_EXPIRE_MINUTES", 30))
    
    def hash_password(self, password: str) -> str:
        """Hash password using bcrypt"""
        return pwd_context.hash(password)
    
    def verify_password(self, plain_password: str, hashed_password: str) -> bool:
        """Verify password"""
        return pwd_context.verify(plain_password, hashed_password)
    
    def create_access_token(self, user_id: str, email: str) -> tuple[str, datetime]:
        """Create JWT access token"""
        expires = datetime.utcnow() + timedelta(minutes=self.access_token_expire)
        payload = {
            "user_id": str(user_id),
            "email": email,
            "exp": expires,
            "iat": datetime.utcnow()
        }
        token = jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
        return token, expires
    
    def verify_access_token(self, token: str) -> Optional[TokenData]:
        """Verify JWT token"""
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
            user_id = payload.get("user_id")
            email = payload.get("email")
            exp = datetime.fromtimestamp(payload.get("exp"))
            
            if not user_id or not email:
                return None
            
            return TokenData(user_id=user_id, email=email, exp=exp)
        except JWTError:
            return None
```

### 2.4 Command Processing Service (Deepen)

Create `backend/app/services/command_service.py`:
```python
from uuid import UUID
from sqlalchemy.orm import Session
from datetime import datetime
from typing import Optional, List
import logging

from backend.app.models import Command, CommandLog, CommandStatus, CommandSource
from backend.app.services.llm_service import LLMService
from backend.app.services.action_executor import ActionExecutor

logger = logging.getLogger(__name__)

class CommandService:
    def __init__(self, db: Session):
        self.db = db
        self.llm_service = LLMService()
        self.executor = ActionExecutor()
    
    async def create_command(
        self,
        user_id: UUID,
        command_text: str,
        source: CommandSource = CommandSource.MOBILE,
        requires_approval: bool = False
    ) -> Command:
        """Create new command"""
        command = Command(
            user_id=user_id,
            command_text=command_text,
            source=source,
            requires_approval=requires_approval,
            status=CommandStatus.PENDING if requires_approval else CommandStatus.APPROVED
        )
        self.db.add(command)
        self.db.commit()
        self.db.refresh(command)
        return command
    
    async def execute_command(self, command_id: UUID) -> dict:
        """Execute a single command"""
        command = self.db.query(Command).filter(Command.id == command_id).first()
        if not command:
            raise ValueError(f"Command {command_id} not found")
        
        if command.status != CommandStatus.APPROVED:
            raise ValueError(f"Command not approved for execution")
        
        command.status = CommandStatus.EXECUTING
        command.started_at = datetime.utcnow()
        self.db.commit()
        
        try:
            # 1. Capture current screen
            screenshot = self.executor.capture_screen()
            
            # 2. Get LLM to generate action plan
            actions = await self.llm_service.plan_actions(
                command_text=command.command_text,
                current_screen=screenshot
            )
            
            # 3. Execute actions
            result = await self.executor.execute_actions(actions)
            
            # 4. Capture result screenshot
            result_screenshot = self.executor.capture_screen()
            
            command.status = CommandStatus.COMPLETED
            command.completed_at = datetime.utcnow()
            command.actual_duration_seconds = int((datetime.utcnow() - command.started_at).total_seconds())
            command.result_summary = result.get("summary", "Success")
            command.result_data = {
                "actions_executed": len(actions),
                "screenshot_before": screenshot.tobytes().hex()[:100],  # Store partial
                "screenshot_after": result_screenshot.tobytes().hex()[:100]
            }
            
            self.db.commit()
            return result
            
        except Exception as e:
            command.status = CommandStatus.FAILED
            command.completed_at = datetime.utcnow()
            command.result_summary = f"Error: {str(e)}"
            self.db.commit()
            
            await self.log_command(command.id, "error", f"Execution failed: {str(e)}")
            raise
    
    async def log_command(self, command_id: UUID, level: str, message: str):
        """Log command activity"""
        log = CommandLog(
            command_id=command_id,
            log_level=level,
            message=message
        )
        self.db.add(log)
        self.db.commit()
    
    def approve_command(self, command_id: UUID) -> Command:
        """Approve a pending command"""
        command = self.db.query(Command).filter(Command.id == command_id).first()
        if not command:
            raise ValueError(f"Command {command_id} not found")
        
        command.status = CommandStatus.APPROVED
        command.approved_by_user = True
        command.approved_at = datetime.utcnow()
        self.db.commit()
        self.db.refresh(command)
        return command
    
    def get_command_history(self, user_id: UUID, limit: int = 50) -> List[Command]:
        """Get user's command history"""
        return self.db.query(Command)\
            .filter(Command.user_id == user_id)\
            .order_by(Command.created_at.desc())\
            .limit(limit)\
            .all()
```

### 2.5 LLM Service (Deepen)

Create `backend/app/services/llm_service.py`:
```python
import os
import json
import base64
from typing import List, Optional
from PIL import Image
import io
import logging

logger = logging.getLogger(__name__)

class LLMService:
    def __init__(self):
        self.provider = os.getenv("DEFAULT_LLM_PROVIDER", "cloud")
        self.fallback_provider = os.getenv("FALLBACK_LLM_PROVIDER", "local")
        self._init_clients()
    
    def _init_clients(self):
        """Initialize LLM clients based on provider"""
        if self.provider == "cloud":
            try:
                from openai import AzureOpenAI
                self.client = AzureOpenAI(
                    api_key=os.getenv("AZURE_OPENAI_API_KEY"),
                    api_version="2024-02-15-preview",
                    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT")
                )
                self.model = os.getenv("AZURE_OPENAI_MODEL", "gpt-4-vision")
                self.deployment = os.getenv("AZURE_OPENAI_DEPLOYMENT", "gpt-4-vision")
            except Exception as e:
                logger.warning(f"Failed to init Azure OpenAI: {e}, falling back to local")
                self.provider = self.fallback_provider
        
        if self.provider == "local":
            import ollama
            self.client = ollama.Client(base_url=os.getenv("OLLAMA_BASE_URL", "http://localhost:11434"))
            self.model = os.getenv("OLLAMA_MODEL", "llava-phi")
    
    async def plan_actions(self, command_text: str, current_screen: Image) -> List[dict]:
        """Generate action plan from command and screen"""
        
        # Convert image to base64
        img_byte_arr = io.BytesIO()
        current_screen.save(img_byte_arr, format='PNG')
        img_byte_arr.seek(0)
        image_base64 = base64.b64encode(img_byte_arr.getvalue()).decode()
        
        system_prompt = """You are WCUA - Windows Computer Use Agent.
Your role is to:
1. Understand user intentions from natural language
2. Analyze Windows screen screenshots provided
3. Generate precise, step-by-step action sequences

Each action must be one of these types:
- click(x, y): Click at coordinates
- type(text): Type text
- key(key_name): Press key (enter, tab, escape, backspace, etc)
- wait(seconds): Wait for specified seconds
- screenshot(): Capture current screen
- hotkey(keys): Press multiple keys (e.g., ['ctrl', 'c'])

Output ONLY valid JSON array of actions.

Example:
[
  {"type": "click", "x": 100, "y": 50},
  {"type": "type", "text": "hello@example.com"},
  {"type": "key", "key": "enter"},
  {"type": "wait", "seconds": 2},
  {"type": "screenshot"}
]"""

        user_prompt = f"""User Command: {command_text}

Current screen is attached. Analyze it and generate the minimal sequence of actions to accomplish the user's command.

Requirements:
- Keep sequence as short as possible
- Include error recovery if needed
- Be specific with coordinates
- Output ONLY JSON array, no other text"""

        try:
            if self.provider == "cloud":
                response = self.client.chat.completions.create(
                    model=self.deployment,
                    messages=[
                        {"role": "system", "content": system_prompt},
                        {
                            "role": "user",
                            "content": [
                                {"type": "text", "text": user_prompt},
                                {
                                    "type": "image_url",
                                    "image_url": {
                                        "url": f"data:image/png;base64,{image_base64}"
                                    }
                                }
                            ]
                        }
                    ],
                    max_tokens=2000
                )
                response_text = response.choices[0].message.content
            else:
                response = self.client.generate(
                    model=self.model,
                    prompt=f"{system_prompt}\n\n{user_prompt}",
                    images=[image_base64],
                    stream=False
                )
                response_text = response['response']
            
            # Parse JSON from response
            actions = json.loads(response_text)
            return actions
            
        except Exception as e:
            logger.error(f"LLM action planning failed: {e}")
            # Return default actions on failure
            return [{"type": "screenshot"}]
```

### 2.6 Action Executor (Deepen)

Create `backend/app/services/action_executor.py`:
```python
import pyautogui
import keyboard
import asyncio
from PIL import ImageGrab
from typing import List
import logging
import time

logger = logging.getLogger(__name__)

class ActionExecutor:
    def __init__(self):
        pyautogui.FAILSAFE = True
        pyautogui.PAUSE = 0.1
    
    def capture_screen(self):
        """Capture current screen"""
        try:
            screenshot = ImageGrab.grab()
            return screenshot
        except Exception as e:
            logger.error(f"Failed to capture screen: {e}")
            raise
    
    async def execute_actions(self, actions: List[dict]) -> dict:
        """Execute sequence of actions"""
        results = {
            "actions_executed": 0,
            "summary": "Completed successfully",
            "errors": []
        }
        
        for i, action in enumerate(actions):
            try:
                action_type = action.get("type")
                
                if action_type == "click":
                    await self._execute_click(action.get("x"), action.get("y"))
                
                elif action_type == "type":
                    await self._execute_type(action.get("text"))
                
                elif action_type == "key":
                    await self._execute_key(action.get("key"))
                
                elif action_type == "hotkey":
                    await self._execute_hotkey(action.get("keys", []))
                
                elif action_type == "wait":
                    await self._execute_wait(action.get("seconds", 1))
                
                elif action_type == "screenshot":
                    await self._execute_screenshot()
                
                results["actions_executed"] += 1
                
            except Exception as e:
                error_msg = f"Action {i} ({action.get('type')}) failed: {str(e)}"
                logger.error(error_msg)
                results["errors"].append(error_msg)
                # Continue with next action
        
        return results
    
    async def _execute_click(self, x: int, y: int):
        """Click at coordinates"""
        pyautogui.moveTo(x, y, duration=0.2)
        pyautogui.click()
        await asyncio.sleep(0.3)
    
    async def _execute_type(self, text: str):
        """Type text"""
        for char in text:
            pyautogui.typewrite(char, interval=0.05)
        await asyncio.sleep(0.2)
    
    async def _execute_key(self, key: str):
        """Press single key"""
        key_map = {
            'enter': 'return',
            'tab': 'tab',
            'escape': 'esc',
            'backspace': 'backspace',
            'delete': 'delete',
            'shift': 'shift',
            'ctrl': 'ctrl',
            'alt': 'alt',
            'home': 'home',
            'end': 'end',
            'pageup': 'pageup',
            'pagedown': 'pagedown'
        }
        keyboard.press(key_map.get(key.lower(), key))
        await asyncio.sleep(0.2)
    
    async def _execute_hotkey(self, keys: List[str]):
        """Press multiple keys"""
        keyboard.press_and_release('+'.join(keys))
        await asyncio.sleep(0.3)
    
    async def _execute_wait(self, seconds: float):
        """Wait for specified seconds"""
        await asyncio.sleep(seconds)
    
    async def _execute_screenshot(self):
        """Capture screenshot"""
        self.capture_screen()
        await asyncio.sleep(0.2)
```

---

## Phase 3: API Endpoints (Deepen) (Jan 2, 2024 - 18 hours)

### 3.1 Main Application Entry Point

Create `backend/app/main.py`:
```python
from fastapi import FastAPI, Depends
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from contextlib import asynccontextmanager
import logging
import os

# Import routes
from backend.app.routes import auth, commands, users

# Configure logging
logging.basicConfig(level=os.getenv("LOG_LEVEL", "INFO"))
logger = logging.getLogger(__name__)

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    logger.info("WCUA API Starting up")
    yield
    # Shutdown
    logger.info("WCUA API Shutting down")

app = FastAPI(
    title="Windows Computer Use Agent API",
    description="Control Windows computers via natural language",
    version="2.0",
    lifespan=lifespan
)

# CORS Middleware
allowed_origins = os.getenv("ALLOWED_ORIGINS", "http://localhost:3000").split(",")
app.add_middleware(
    CORSMiddleware,
    allow_origins=allowed_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Trusted Host Middleware
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["localhost", "127.0.0.1"])

# Include routers
app.include_router(auth.router, prefix="/api/v1/auth", tags=["auth"])
app.include_router(commands.router, prefix="/api/v1/commands", tags=["commands"])
app.include_router(users.router, prefix="/api/v1/users", tags=["users"])

@app.get("/health")
async def health_check():
    return {"status": "healthy", "version": "2.0"}

@app.get("/")
async def root():
    return {
        "name": "Windows Computer Use Agent API",
        "version": "2.0",
        "docs_url": "/docs",
        "health_url": "/health"
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### 3.2 Authentication Routes

Create `backend/app/routes/auth.py`:
```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from pydantic import BaseModel, EmailStr

from backend.app.database import get_db
from backend.app.models import User, APIKey
from backend.app.services.auth_service import AuthService

router = APIRouter()
auth_service = AuthService()

class RegisterRequest(BaseModel):
    email: EmailStr
    password: str
    full_name: str

class LoginRequest(BaseModel):
    email: EmailStr
    password: str

class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    expires_in: int

@router.post("/register", response_model=dict)
async def register(req: RegisterRequest, db: Session = Depends(get_db)):
    """Register new user"""
    # Check if user exists
    existing = db.query(User).filter(User.email == req.email).first()
    if existing:
        raise HTTPException(status_code=400, detail="Email already registered")
    
    # Create user
    user = User(
        email=req.email,
        password_hash=auth_service.hash_password(req.password),
        full_name=req.full_name
    )
    db.add(user)
    db.commit()
    db.refresh(user)
    
    # Create access token
    token, expires = auth_service.create_access_token(str(user.id), user.email)
    
    return {
        "user_id": str(user.id),
        "email": user.email,
        "access_token": token,
        "token_type": "bearer"
    }

@router.post("/login", response_model=dict)
async def login(req: LoginRequest, db: Session = Depends(get_db)):
    """Login user"""
    user = db.query(User).filter(User.email == req.email).first()
    if not user or not auth_service.verify_password(req.password, user.password_hash):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    token, expires = auth_service.create_access_token(str(user.id), user.email)
    
    return {
        "access_token": token,
        "token_type": "bearer",
        "expires_in": int((expires - __import__('datetime').datetime.utcnow()).total_seconds())
    }

@router.post("/api-keys", response_model=dict)
async def create_api_key(
    name: str,
    current_user_id: str = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Create API key for current user"""
    import secrets
    import hashlib
    
    key = secrets.token_urlsafe(32)
    key_hash = hashlib.sha256(key.encode()).hexdigest()
    
    api_key = APIKey(
        user_id=current_user_id,
        key_hash=key_hash,
        name=name
    )
    db.add(api_key)
    db.commit()
    
    return {"key": key, "name": name, "created_at": api_key.created_at}

def get_current_user(token: str) -> str:
    """Extract user from JWT token"""
    # Implementation in auth middleware
    pass
```

### 3.3 Commands Routes

Create `backend/app/routes/commands.py`:
```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from pydantic import BaseModel
from uuid import UUID

from backend.app.database import get_db
from backend.app.services.command_service import CommandService
from backend.app.models import CommandSource

router = APIRouter()

class CommandRequest(BaseModel):
    command_text: str
    source: CommandSource = CommandSource.MOBILE
    requires_approval: bool = False

class CommandResponse(BaseModel):
    id: str
    command_text: str
    status: str
    created_at: str

@router.post("/execute", response_model=dict)
async def execute_command(
    req: CommandRequest,
    current_user_id: str = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Submit command for execution"""
    service = CommandService(db)
    command = await service.create_command(
        user_id=current_user_id,
        command_text=req.command_text,
        source=req.source,
        requires_approval=req.requires_approval
    )
    
    # Queue for async execution
    # In production: send to message queue
    
    return {
        "command_id": str(command.id),
        "status": command.status,
        "requires_approval": command.requires_approval
    }

@router.get("/{command_id}", response_model=dict)
async def get_command(
    command_id: UUID,
    current_user_id: str = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Get command status and results"""
    from backend.app.models import Command
    
    command = db.query(Command).filter(
        Command.id == command_id,
        Command.user_id == current_user_id
    ).first()
    
    if not command:
        raise HTTPException(status_code=404, detail="Command not found")
    
    return {
        "id": str(command.id),
        "command_text": command.command_text,
        "status": command.status,
        "result_summary": command.result_summary,
        "created_at": command.created_at,
        "completed_at": command.completed_at
    }

@router.post("/{command_id}/approve", response_model=dict)
async def approve_command(
    command_id: UUID,
    current_user_id: str = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Approve a pending command"""
    service = CommandService(db)
    command = service.approve_command(command_id)
    
    return {
        "command_id": str(command.id),
        "status": command.status,
        "approved_at": command.approved_at
    }

def get_current_user(token: str) -> str:
    """Extract user from JWT token"""
    pass
```

---

## Phase 4: Frontend Development (Teja) (Jan 2-4, 2024 - 48 hours)

### 4.1 React Native Mobile App Setup

```bash
# Create React Native app
npx create-expo-app wcua-mobile
cd wcua-mobile

# Install dependencies
npm install
npm install @react-navigation/native @react-navigation/bottom-tabs
npm install @react-navigation/stack
npm install axios
npm install react-native-voice
npm install expo-speech
npm install expo-camera
npm install socket.io-client
npm install @react-native-async-storage/async-storage

# Create folder structure
mkdir -p src/{screens,components,services,utils,assets}
```

### 4.2 Mobile App Main Structure (Teja)

Create `mobile-app/src/services/api.js`:
```javascript
import axios from 'axios';
import AsyncStorage from '@react-native-async-storage/async-storage';

const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:8000/api/v1';

const api = axios.create({
  baseURL: API_BASE_URL,
  timeout: 30000,
});

// Add auth token to requests
api.interceptors.request.use(async (config) => {
  const token = await AsyncStorage.getItem('access_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authService = {
  login: (email, password) => api.post('/auth/login', { email, password }),
  register: (email, password, fullName) =>
    api.post('/auth/register', { email, password, full_name: fullName }),
  logout: async () => {
    await AsyncStorage.removeItem('access_token');
  },
};

export const commandService = {
  execute: (commandText, source = 'mobile') =>
    api.post('/commands/execute', { command_text: commandText, source }),
  getStatus: (commandId) => api.get(`/commands/${commandId}`),
  getHistory: () => api.get('/commands/history'),
  approve: (commandId) => api.post(`/commands/${commandId}/approve`),
};

export default api;
```

Create `mobile-app/src/screens/CommandScreen.jsx`:
```javascript
import React, { useState, useEffect } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  ScrollView,
  ActivityIndicator,
  Image,
} from 'react-native';
import Voice from 'react-native-voice';
import { commandService } from '../services/api';

export default function CommandScreen() {
  const [command, setCommand] = useState('');
  const [isListening, setIsListening] = useState(false);
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState(null);
  const [screenshot, setScreenshot] = useState(null);

  useEffect(() => {
    Voice.onSpeechStart = onSpeechStart;
    Voice.onSpeechEnd = onSpeechEnd;
    Voice.onSpeechResults = onSpeechResults;
    return () => {
      Voice.destroy().catch(() => {});
    };
  }, []);

  const onSpeechStart = () => {
    setIsListening(true);
  };

  const onSpeechEnd = () => {
    setIsListening(false);
  };

  const onSpeechResults = (event) => {
    setCommand(event.value[0]);
  };

  const handleVoiceInput = async () => {
    try {
      if (isListening) {
        await Voice.stop();
        setIsListening(false);
      } else {
        await Voice.start('en-US');
      }
    } catch (error) {
      console.error('Voice input error:', error);
    }
  };

  const handleExecuteCommand = async () => {
    if (!command.trim()) {
      alert('Please enter a command');
      return;
    }

    setLoading(true);
    try {
      const response = await commandService.execute(command.trim());
      setResult({
        commandId: response.data.command_id,
        status: response.data.status,
        timestamp: new Date().toLocaleTimeString(),
      });
      setCommand('');
    } catch (error) {
      alert('Error executing command: ' + error.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <ScrollView style={{ flex: 1, backgroundColor: '#f5f5f5', padding: 16 }}>
      <Text style={{ fontSize: 24, fontWeight: 'bold', marginBottom: 20 }}>
        Windows Computer Use Agent
      </Text>

      {/* Screenshot Preview */}
      {screenshot && (
        <Image
          source={{ uri: screenshot }}
          style={{ width: '100%', height: 300, marginBottom: 20, borderRadius: 8 }}
        />
      )}

      {/* Command Input */}
      <TextInput
        placeholder="Describe what you want to do..."
        value={command}
        onChangeText={setCommand}
        multiline
        style={{
          borderWidth: 1,
          borderColor: '#ddd',
          padding: 12,
          marginBottom: 12,
          borderRadius: 8,
          minHeight: 80,
        }}
      />

      {/* Voice Input Button */}
      <TouchableOpacity
        onPress={handleVoiceInput}
        style={{
          backgroundColor: isListening ? '#ff6b6b' : '#4CAF50',
          padding: 12,
          borderRadius: 8,
          marginBottom: 12,
          alignItems: 'center',
        }}
      >
        <Text style={{ color: 'white', fontWeight: 'bold', fontSize: 16 }}>
          {isListening ? '🎤 Stop Listening' : '🎤 Voice Input'}
        </Text>
      </TouchableOpacity>

      {/* Execute Button */}
      <TouchableOpacity
        onPress={handleExecuteCommand}
        disabled={loading}
        style={{
          backgroundColor: loading ? '#ccc' : '#2196F3',
          padding: 16,
          borderRadius: 8,
          alignItems: 'center',
          marginBottom: 20,
        }}
      >
        {loading ? (
          <ActivityIndicator color="white" />
        ) : (
          <Text style={{ color: 'white', fontWeight: 'bold', fontSize: 16 }}>
            Execute Command
          </Text>
        )}
      </TouchableOpacity>

      {/* Result Display */}
      {result && (
        <View style={{ backgroundColor: '#e8f5e9', padding: 16, borderRadius: 8 }}>
          <Text style={{ fontWeight: 'bold', fontSize: 16, marginBottom: 8 }}>
            Command Submitted
          </Text>
          <Text>ID: {result.commandId}</Text>
          <Text>Status: {result.status}</Text>
          <Text>Time: {result.timestamp}</Text>
        </View>
      )}
    </ScrollView>
  );
}
```

### 4.3 Web Dashboard (Teja - React)

```bash
cd web-dashboard
npx create-react-app .
npm install axios socket.io-client chart.js react-chartjs-2
```

Create `web-dashboard/src/components/Dashboard.jsx`:
```javascript
import React, { useState, useEffect } from 'react';
import api from '../services/api';
import './Dashboard.css';

function Dashboard() {
  const [commands, setCommands] = useState([]);
  const [ollamaUrl, setOllamaUrl] = useState('http://localhost:11434');
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    fetchCommands();
    // Poll every 5 seconds
    const interval = setInterval(fetchCommands, 5000);
    return () => clearInterval(interval);
  }, []);

  const fetchCommands = async () => {
    try {
      const response = await api.get('/commands/history');
      setCommands(response.data);
    } catch (error) {
      console.error('Error fetching commands:', error);
    }
  };

  const handleOllamaUrlSave = async () => {
    try {
      await api.post('/settings/ollama-url', { url: ollamaUrl });
      alert('Ollama URL saved successfully');
    } catch (error) {
      alert('Error saving Ollama URL');
    }
  };

  return (
    <div className="dashboard-container">
      <header className="dashboard-header">
        <h1>🖥️ Windows Computer Use Agent</h1>
        <p>Control your Windows PC from anywhere</p>
      </header>

      <div className="dashboard-content">
        {/* Settings Panel */}
        <div className="settings-panel">
          <h2>Settings</h2>
          <div className="setting-group">
            <label>Ollama Base URL:</label>
            <input
              type="text"
              value={ollamaUrl}
              onChange={(e) => setOllamaUrl(e.target.value)}
              placeholder="http://localhost:11434"
            />
            <button onClick={handleOllamaUrlSave}>Save</button>
          </div>
        </div>

        {/* Commands History */}
        <div className="commands-panel">
          <h2>Command History</h2>
          <div className="commands-list">
            {commands.map((cmd) => (
              <div key={cmd.id} className="command-card">
                <div className="command-text">{cmd.command_text}</div>
                <div className="command-status">
                  <span className={`status-badge ${cmd.status}`}>
                    {cmd.status.toUpperCase()}
                  </span>
                </div>
                <div className="command-time">
                  {new Date(cmd.created_at).toLocaleString()}
                </div>
              </div>
            ))}
          </div>
        </div>

        {/* Real-time Monitoring */}
        <div className="monitoring-panel">
          <h2>Live Monitoring</h2>
          <div id="screenshot-preview" className="screenshot-preview">
            <p>Screenshot will appear here during execution...</p>
          </div>
        </div>
      </div>
    </div>
  );
}

export default Dashboard;
```

---

## Phase 5: Deployment & Testing (Both) (Jan 4-5, 2024 - 24 hours)

### 5.1 Docker Build (Deepen)

Create `backend/Dockerfile`:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    libpq-dev \
    gcc \
    python3-dev \
    libx11-dev \
    libxext-dev \
    libxrandr-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Run migrations
RUN alembic upgrade head

# Start application
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 5.2 Testing Suite (Both)

Create `backend/tests/test_commands.py`:
```python
import pytest
from fastapi.testclient import TestClient
from backend.app.main import app

client = TestClient(app)

@pytest.fixture
def auth_headers():
    response = client.post("/api/v1/auth/register", json={
        "email": "test@example.com",
        "password": "password123",
        "full_name": "Test User"
    })
    token = response.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}

def test_execute_command(auth_headers):
    response = client.post(
        "/api/v1/commands/execute",
        json={"command_text": "open notepad"},
        headers=auth_headers
    )
    assert response.status_code == 200
    assert "command_id" in response.json()

def test_get_command_status(auth_headers):
    # First, create a command
    create_response = client.post(
        "/api/v1/commands/execute",
        json={"command_text": "open notepad"},
        headers=auth_headers
    )
    command_id = create_response.json()["command_id"]
    
    # Then check its status
    response = client.get(
        f"/api/v1/commands/{command_id}",
        headers=auth_headers
    )
    assert response.status_code == 200
    assert response.json()["status"] in ["pending", "approved", "executing", "completed"]
```

### 5.3 Deployment Steps (Deepen)

```bash
# 1. Start Docker containers
docker-compose up -d

# 2. Verify services
docker ps

# 3. Check API health
curl http://localhost:8000/health

# 4. Run migrations
docker-compose exec api alembic upgrade head

# 5. Test endpoints
curl -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"test123","full_name":"Test User"}'

# 6. Deploy to Azure
az login
az containerregistry create --resource-group wcua-rg --name wcuaregistry --sku Basic
az acr build --registry wcuaregistry --image wcua-api:latest ./backend
```

---

## Daily Cheat Sheet

### Dec 31 (Deepen): 6 Hours
- ✅ Repository setup & Docker compose
- ✅ Database models & migrations
- ✅ .env configuration

### Jan 1 (Both): 24 Hours
**Deepen:**
- ✅ Auth service complete
- ✅ Command service complete
- ✅ LLM service scaffolding

**Teja:**
- ✅ Mobile app screens (2-3)
- ✅ API service layer
- ✅ Web dashboard layout

### Jan 2 (Both): 24 Hours
**Deepen:**
- ✅ Action executor fully implemented
- ✅ API endpoints (auth + commands)
- ✅ Database seeding with test data

**Teja:**
- ✅ Mobile voice input integration
- ✅ Web dashboard real-time updates
- ✅ Settings screen for Ollama URL

### Jan 3-4 (Both): 24 Hours
**Deepen:**
- ✅ Integration testing
- ✅ Error handling polish
- ✅ Docker deployment

**Teja:**
- ✅ UI polish & styling
- ✅ Mobile app builds (iOS/Android)
- ✅ Web dashboard optimization

### Jan 5: Final
- ✅ Code review
- ✅ Documentation finalization
- ✅ Submission package creation

---

**Document Version:** 1.0  
**Last Updated:** December 31, 2024
