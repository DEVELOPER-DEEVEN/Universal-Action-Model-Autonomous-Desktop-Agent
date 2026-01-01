# Windows Computer Use Agent (WCUA) - Code Templates & Boilerplate

---

## Table of Contents
1. FastAPI Base Templates
2. React Native Components
3. Database Query Helpers
4. LLM Prompt Templates
5. Error Handling Patterns
6. Testing Templates

---

## 1. FastAPI Base Templates

### 1.1 Async Route Handler Template

```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from pydantic import BaseModel, Field
from typing import Optional, List
import logging

logger = logging.getLogger(__name__)
router = APIRouter()

# Request/Response Models
class ItemRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=255)
    description: Optional[str] = None
    
    class Config:
        example = {
            "name": "Example Item",
            "description": "A sample item"
        }

class ItemResponse(BaseModel):
    id: str
    name: str
    description: Optional[str]
    created_at: str
    
    class Config:
        from_attributes = True

# Route Handler
@router.post("/items", response_model=ItemResponse, status_code=status.HTTP_201_CREATED)
async def create_item(
    item: ItemRequest,
    db: Session = Depends(get_db),
    current_user_id: str = Depends(get_current_user)
) -> ItemResponse:
    """
    Create a new item.
    
    - **name**: Item name (required)
    - **description**: Item description (optional)
    """
    try:
        # Validate business logic
        if not current_user_id:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Not authenticated"
            )
        
        # Create database object
        from backend.app.models import Item
        db_item = Item(
            name=item.name,
            description=item.description,
            user_id=current_user_id
        )
        
        db.add(db_item)
        db.commit()
        db.refresh(db_item)
        
        logger.info(f"Created item {db_item.id} for user {current_user_id}")
        
        return ItemResponse.from_orm(db_item)
        
    except Exception as e:
        db.rollback()
        logger.error(f"Error creating item: {str(e)}")
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Failed to create item"
        )

@router.get("/items/{item_id}", response_model=ItemResponse)
async def get_item(
    item_id: str,
    db: Session = Depends(get_db),
    current_user_id: str = Depends(get_current_user)
) -> ItemResponse:
    """Retrieve item by ID"""
    from backend.app.models import Item
    
    item = db.query(Item).filter(
        Item.id == item_id,
        Item.user_id == current_user_id
    ).first()
    
    if not item:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Item not found"
        )
    
    return ItemResponse.from_orm(item)

@router.get("/items", response_model=List[ItemResponse])
async def list_items(
    skip: int = 0,
    limit: int = 50,
    db: Session = Depends(get_db),
    current_user_id: str = Depends(get_current_user)
) -> List[ItemResponse]:
    """List all items for current user"""
    from backend.app.models import Item
    
    items = db.query(Item).filter(
        Item.user_id == current_user_id
    ).offset(skip).limit(limit).all()
    
    return [ItemResponse.from_orm(item) for item in items]
```

### 1.2 Dependency Injection Template

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthCredentials
from backend.app.services.auth_service import AuthService

security = HTTPBearer()
auth_service = AuthService()

async def get_current_user(credentials: HTTPAuthCredentials = Depends(security)) -> str:
    """Extract and validate JWT token, return user_id"""
    token = credentials.credentials
    token_data = auth_service.verify_access_token(token)
    
    if not token_data:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid token"
        )
    
    return token_data.user_id

async def get_admin_user(
    current_user_id: str = Depends(get_current_user),
    db: Session = Depends(get_db)
) -> str:
    """Verify user has admin role"""
    from backend.app.models import User, UserRole
    
    user = db.query(User).filter(User.id == current_user_id).first()
    if not user or user.role != UserRole.ADMIN:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin access required"
        )
    
    return current_user_id
```

### 1.3 Error Handling Template

```python
from fastapi import Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
import logging

logger = logging.getLogger(__name__)

class APIError(Exception):
    """Base API error"""
    def __init__(self, message: str, status_code: int = 400, error_code: str = None):
        self.message = message
        self.status_code = status_code
        self.error_code = error_code or "ERROR"

@app.exception_handler(APIError)
async def api_error_handler(request: Request, exc: APIError):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": exc.error_code,
            "message": exc.message,
            "path": str(request.url)
        }
    )

@app.exception_handler(RequestValidationError)
async def validation_error_handler(request: Request, exc: RequestValidationError):
    errors = []
    for error in exc.errors():
        errors.append({
            "field": ".".join(str(x) for x in error["loc"][1:]),
            "message": error["msg"]
        })
    
    return JSONResponse(
        status_code=422,
        content={
            "error": "VALIDATION_ERROR",
            "message": "Invalid request data",
            "errors": errors
        }
    )
```

---

## 2. React Native Components

### 2.1 Custom Hook Template

```javascript
import { useState, useEffect, useCallback } from 'react';
import AsyncStorage from '@react-native-async-storage/async-storage';

// Hook for managing async data fetching
export const useFetch = (url, options = {}) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  const fetchData = useCallback(async () => {
    try {
      setLoading(true);
      setError(null);
      
      const response = await fetch(url, {
        headers: {
          'Content-Type': 'application/json',
          ...options.headers
        },
        ...options
      });

      if (!response.ok) {
        throw new Error(`API error: ${response.status}`);
      }

      const json = await response.json();
      setData(json);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, [url, options]);

  useEffect(() => {
    fetchData();
  }, [fetchData]);

  return { data, loading, error, refetch: fetchData };
};

// Hook for managing local auth state
export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const checkAuth = async () => {
      try {
        const token = await AsyncStorage.getItem('access_token');
        const userId = await AsyncStorage.getItem('user_id');
        
        if (token && userId) {
          setUser({ id: userId, token });
        }
      } catch (error) {
        console.error('Auth check error:', error);
      } finally {
        setLoading(false);
      }
    };

    checkAuth();
  }, []);

  const login = async (email, password) => {
    const response = await fetch('http://localhost:8000/api/v1/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });

    const data = await response.json();
    await AsyncStorage.setItem('access_token', data.access_token);
    await AsyncStorage.setItem('user_id', data.user_id);
    setUser({ id: data.user_id, token: data.access_token });

    return data;
  };

  const logout = async () => {
    await AsyncStorage.removeItem('access_token');
    await AsyncStorage.removeItem('user_id');
    setUser(null);
  };

  return { user, loading, login, logout };
};
```

### 2.2 Form Component Template

```javascript
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  ActivityIndicator,
  StyleSheet,
} from 'react-native';

const FormComponent = ({ onSubmit, fields, isLoading }) => {
  const [formData, setFormData] = useState(
    fields.reduce((acc, field) => ({ ...acc, [field.name]: '' }), {})
  );
  const [errors, setErrors] = useState({});

  const validateForm = () => {
    const newErrors = {};
    fields.forEach(field => {
      if (field.required && !formData[field.name]?.trim()) {
        newErrors[field.name] = `${field.label} is required`;
      }
      if (field.type === 'email' && formData[field.name]) {
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        if (!emailRegex.test(formData[field.name])) {
          newErrors[field.name] = 'Invalid email address';
        }
      }
    });
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleSubmit = async () => {
    if (validateForm()) {
      await onSubmit(formData);
    }
  };

  return (
    <View style={styles.container}>
      {fields.map((field) => (
        <View key={field.name} style={styles.fieldContainer}>
          <Text style={styles.label}>{field.label}</Text>
          <TextInput
            style={[styles.input, errors[field.name] && styles.inputError]}
            placeholder={field.placeholder}
            value={formData[field.name]}
            onChangeText={(text) =>
              setFormData({ ...formData, [field.name]: text })
            }
            secureTextEntry={field.type === 'password'}
            keyboardType={field.type === 'email' ? 'email-address' : 'default'}
            editable={!isLoading}
          />
          {errors[field.name] && (
            <Text style={styles.errorText}>{errors[field.name]}</Text>
          )}
        </View>
      ))}

      <TouchableOpacity
        style={[styles.button, isLoading && styles.buttonDisabled]}
        onPress={handleSubmit}
        disabled={isLoading}
      >
        {isLoading ? (
          <ActivityIndicator color="white" />
        ) : (
          <Text style={styles.buttonText}>Submit</Text>
        )}
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { padding: 16 },
  fieldContainer: { marginBottom: 16 },
  label: { fontSize: 14, fontWeight: '600', marginBottom: 8 },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    padding: 12,
    borderRadius: 8,
    fontSize: 14,
  },
  inputError: { borderColor: '#f44336' },
  errorText: { color: '#f44336', fontSize: 12, marginTop: 4 },
  button: {
    backgroundColor: '#2196F3',
    padding: 16,
    borderRadius: 8,
    alignItems: 'center',
  },
  buttonDisabled: { backgroundColor: '#ccc' },
  buttonText: { color: 'white', fontWeight: '600', fontSize: 16 },
});

export default FormComponent;
```

### 2.3 Modal Component Template

```javascript
import React, { useState } from 'react';
import {
  Modal,
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  Pressable,
} from 'react-native';

const CustomModal = ({ visible, title, content, buttons, onClose }) => {
  return (
    <Modal
      visible={visible}
      transparent={true}
      animationType="fade"
      onRequestClose={onClose}
    >
      <Pressable
        style={styles.overlay}
        onPress={onClose}
      >
        <View style={styles.centeredView}>
          <Pressable
            style={styles.modalView}
            onPress={(e) => e.stopPropagation()}
          >
            <Text style={styles.title}>{title}</Text>
            <View style={styles.content}>
              {typeof content === 'string' ? (
                <Text style={styles.text}>{content}</Text>
              ) : (
                content
              )}
            </View>
            <View style={styles.buttonContainer}>
              {buttons.map((button, index) => (
                <TouchableOpacity
                  key={index}
                  style={[
                    styles.button,
                    button.style === 'cancel' && styles.cancelButton,
                  ]}
                  onPress={() => {
                    button.onPress?.();
                    onClose();
                  }}
                >
                  <Text
                    style={[
                      styles.buttonText,
                      button.style === 'cancel' && styles.cancelButtonText,
                    ]}
                  >
                    {button.text}
                  </Text>
                </TouchableOpacity>
              ))}
            </View>
          </Pressable>
        </View>
      </Pressable>
    </Modal>
  );
};

const styles = StyleSheet.create({
  overlay: { flex: 1, backgroundColor: 'rgba(0, 0, 0, 0.5)' },
  centeredView: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  modalView: {
    margin: 20,
    backgroundColor: 'white',
    borderRadius: 8,
    padding: 24,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.25,
    shadowRadius: 3.84,
    elevation: 5,
  },
  title: { fontSize: 20, fontWeight: 'bold', marginBottom: 16 },
  content: { marginBottom: 24 },
  text: { fontSize: 14, color: '#666' },
  buttonContainer: { flexDirection: 'row', justifyContent: 'flex-end', gap: 12 },
  button: { backgroundColor: '#2196F3', paddingVertical: 8, paddingHorizontal: 16, borderRadius: 4 },
  cancelButton: { backgroundColor: '#f44336' },
  buttonText: { color: 'white', fontWeight: '600' },
  cancelButtonText: { color: 'white' },
});

export default CustomModal;
```

---

## 3. Database Query Helpers

### 3.1 SQLAlchemy Query Builder Template

```python
from sqlalchemy import and_, or_, desc
from sqlalchemy.orm import Session
from typing import List, Dict, Any, Optional
import logging

logger = logging.getLogger(__name__)

class QueryHelper:
    """Helper class for building complex queries"""
    
    @staticmethod
    def filter_by_conditions(
        db: Session,
        model,
        filters: Dict[str, Any],
        skip: int = 0,
        limit: int = 100,
        order_by: Optional[str] = None,
        descending: bool = False
    ) -> List:
        """
        Build query with dynamic filters
        
        filters: {"user_id": "123", "status": "active"}
        """
        query = db.query(model)
        
        # Apply filters
        for key, value in filters.items():
            if value is not None and hasattr(model, key):
                column = getattr(model, key)
                if isinstance(value, list):
                    query = query.filter(column.in_(value))
                elif isinstance(value, str) and '%' in value:
                    query = query.filter(column.like(value))
                else:
                    query = query.filter(column == value)
        
        # Apply ordering
        if order_by and hasattr(model, order_by):
            order_column = getattr(model, order_by)
            query = query.order_by(desc(order_column) if descending else order_column)
        
        # Apply pagination
        return query.offset(skip).limit(limit).all()
    
    @staticmethod
    def find_by_id(db: Session, model, id: Any):
        """Get single record by ID"""
        return db.query(model).filter(model.id == id).first()
    
    @staticmethod
    def find_by_multiple_fields(db: Session, model, **kwargs):
        """Find record by multiple fields"""
        conditions = [getattr(model, k) == v for k, v in kwargs.items()]
        return db.query(model).filter(and_(*conditions)).first()
    
    @staticmethod
    def find_all_for_user(
        db: Session,
        model,
        user_id: str,
        skip: int = 0,
        limit: int = 100
    ) -> List:
        """Get all records for a user"""
        return db.query(model).filter(
            model.user_id == user_id
        ).offset(skip).limit(limit).all()
    
    @staticmethod
    def count_by_filter(db: Session, model, **kwargs) -> int:
        """Count records matching filter"""
        conditions = [getattr(model, k) == v for k, v in kwargs.items()]
        return db.query(model).filter(and_(*conditions)).count()
```

### 3.2 Bulk Operations Template

```python
class BulkOperations:
    """Bulk database operations"""
    
    @staticmethod
    def bulk_insert(db: Session, model, records: List[Dict]):
        """Insert multiple records"""
        db.bulk_insert_mappings(model, records)
        db.commit()
        return len(records)
    
    @staticmethod
    def bulk_update(db: Session, model, records: List[Dict], id_field: str = 'id'):
        """Update multiple records"""
        for record in records:
            record_id = record.pop(id_field)
            db.query(model).filter(getattr(model, id_field) == record_id).update(record)
        db.commit()
    
    @staticmethod
    def bulk_delete(db: Session, model, ids: List):
        """Delete multiple records by IDs"""
        db.query(model).filter(model.id.in_(ids)).delete()
        db.commit()
        return len(ids)
```

---

## 4. LLM Prompt Templates

### 4.1 System Prompts

```python
SYSTEM_PROMPTS = {
    "action_planning": """You are WCUA - Windows Computer Use Agent.
Your role is to understand user intentions and generate precise action sequences.

Each action must be one of:
- click(x, y)
- type(text)
- key(key_name)
- hotkey([keys])
- wait(seconds)
- screenshot()

Output ONLY valid JSON array. No explanations.""",

    "screen_analysis": """Analyze the provided screenshot and identify:
1. Current application
2. Visible UI elements (buttons, fields, menus)
3. Current state of application
4. Potential next steps

Provide analysis in JSON format.""",

    "command_clarification": """The user command might be ambiguous. 
Generate clarifying questions in JSON format:
{
  "questions": [
    {"id": 1, "text": "question", "options": ["option1", "option2"]}
  ]
}"""
}

# Usage
system_prompt = SYSTEM_PROMPTS["action_planning"]
response = await llm_service.generate(
    system_prompt=system_prompt,
    user_message=f"Take screenshot of {app_name}",
    image=screenshot_base64
)
```

### 4.2 Few-Shot Prompt Examples

```python
FEW_SHOT_EXAMPLES = [
    {
        "command": "Open Notepad and type hello world",
        "screenshot": "notepad_closed",
        "actions": [
            {"type": "key", "key": "win"},
            {"type": "type", "text": "notepad"},
            {"type": "key", "key": "enter"},
            {"type": "wait", "seconds": 2},
            {"type": "type", "text": "hello world"}
        ]
    },
    {
        "command": "Send email to john@example.com saying thank you",
        "screenshot": "outlook_closed",
        "actions": [
            {"type": "key", "key": "win"},
            {"type": "type", "text": "outlook"},
            {"type": "key", "key": "enter"},
            {"type": "wait", "seconds": 3},
            {"type": "click", "x": 150, "y": 200},  # New email button
            {"type": "type", "text": "john@example.com"},
            {"type": "key", "key": "tab"},
            {"type": "key", "key": "tab"},
            {"type": "type", "text": "Thank you"},
            {"type": "key", "key": "ctrl+enter"}  # Send
        ]
    }
]

def get_few_shot_prompt(user_command: str):
    """Generate few-shot prompt with relevant examples"""
    prompt = "Examples of similar commands:\n\n"
    for example in FEW_SHOT_EXAMPLES:
        prompt += f"Command: {example['command']}\n"
        prompt += f"Actions: {json.dumps(example['actions'])}\n\n"
    
    prompt += f"\nNow generate actions for: {user_command}"
    return prompt
```

---

## 5. Error Handling Patterns

### 5.1 Async Error Wrapper

```python
from functools import wraps
from typing import Callable
import asyncio

def async_error_handler(func: Callable):
    """Decorator for async functions with error handling"""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        try:
            return await func(*args, **kwargs)
        except asyncio.TimeoutError:
            logger.error(f"{func.__name__} timed out")
            raise APIError("Operation timed out", status_code=408)
        except ValueError as e:
            logger.warning(f"{func.__name__} validation error: {str(e)}")
            raise APIError(str(e), status_code=400)
        except Exception as e:
            logger.exception(f"{func.__name__} unexpected error: {str(e)}")
            raise APIError("Internal server error", status_code=500)
    
    return wrapper

# Usage
@async_error_handler
async def some_operation():
    pass
```

### 5.2 Retry with Exponential Backoff

```python
import time
from typing import TypeVar, Callable

T = TypeVar('T')

def retry_with_backoff(
    max_attempts: int = 3,
    initial_delay: float = 1.0,
    backoff_factor: float = 2.0
):
    """Decorator for retrying failed operations"""
    def decorator(func: Callable[..., T]) -> Callable[..., T]:
        @wraps(func)
        async def wrapper(*args, **kwargs):
            delay = initial_delay
            last_exception = None
            
            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    if attempt < max_attempts - 1:
                        logger.warning(
                            f"Attempt {attempt + 1} failed for {func.__name__}, "
                            f"retrying in {delay}s: {str(e)}"
                        )
                        await asyncio.sleep(delay)
                        delay *= backoff_factor
            
            raise last_exception
        
        return wrapper
    return decorator

# Usage
@retry_with_backoff(max_attempts=3, initial_delay=0.5)
async def call_llm_with_retry():
    pass
```

---

## 6. Testing Templates

### 6.1 Pytest Fixtures

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from fastapi.testclient import TestClient
from backend.app.main import app
from backend.app.database import Base, get_db

@pytest.fixture(scope="session")
def test_db_engine():
    """Create test database"""
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(bind=engine)
    yield engine
    Base.metadata.drop_all(bind=engine)

@pytest.fixture
def test_db_session(test_db_engine):
    """Create test database session"""
    connection = test_db_engine.connect()
    transaction = connection.begin()
    session = sessionmaker(autocommit=False, autoflush=False, bind=connection)()
    
    yield session
    
    session.close()
    transaction.rollback()
    connection.close()

@pytest.fixture
def client(test_db_session):
    """Create test client with test database"""
    def override_get_db():
        yield test_db_session
    
    app.dependency_overrides[get_db] = override_get_db
    yield TestClient(app)
    app.dependency_overrides.clear()

@pytest.fixture
def test_user(test_db_session):
    """Create test user"""
    from backend.app.models import User
    from backend.app.services.auth_service import AuthService
    
    auth_service = AuthService()
    user = User(
        email="test@example.com",
        password_hash=auth_service.hash_password("password123"),
        full_name="Test User"
    )
    test_db_session.add(user)
    test_db_session.commit()
    return user

@pytest.fixture
def auth_headers(client, test_user):
    """Get authenticated headers"""
    response = client.post("/api/v1/auth/login", json={
        "email": test_user.email,
        "password": "password123"
    })
    token = response.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}
```

### 6.2 Test Cases Template

```python
def test_create_command(client, auth_headers, test_user):
    """Test command creation"""
    response = client.post(
        "/api/v1/commands/execute",
        json={"command_text": "open notepad"},
        headers=auth_headers
    )
    
    assert response.status_code == 200
    data = response.json()
    assert "command_id" in data
    assert data["status"] in ["pending", "approved"]

def test_get_command(client, auth_headers, test_user):
    """Test getting command status"""
    # Create command first
    create_response = client.post(
        "/api/v1/commands/execute",
        json={"command_text": "open notepad"},
        headers=auth_headers
    )
    command_id = create_response.json()["command_id"]
    
    # Get status
    response = client.get(
        f"/api/v1/commands/{command_id}",
        headers=auth_headers
    )
    
    assert response.status_code == 200
    assert response.json()["id"] == command_id

def test_unauthorized_access(client):
    """Test unauthorized requests"""
    response = client.get("/api/v1/commands/some-id")
    assert response.status_code == 403

def test_invalid_command(client, auth_headers):
    """Test command validation"""
    response = client.post(
        "/api/v1/commands/execute",
        json={"command_text": ""},
        headers=auth_headers
    )
    assert response.status_code == 422
```

---

**Document Version:** 1.0  
**Last Updated:** December 31, 2024  
**Status:** Ready for Implementation
