# Project Intercept: Autonomous Visual Agent for Windows

**A multimodal AI experiment to bridge the gap between natural language and legacy GUI interfaces.**

---

## 1. The Problem: The Fragility of Traditional Automation
Robotic Process Automation (RPA) has traditionally relied on "selectors"—DOM IDs, XPaths, or window handles. This makes automation brittle; a single CSS change or UI update breaks the entire workflow. Furthermore, millions of legacy Windows applications lack accessible APIs, making them "black boxes" to modern software.

For users with motor impairments, this disconnect is even more critical. Existing accessibility tools often rely on application developers implementing specific accessibility APIs. If an app doesn't support them, the user is locked out.

## 2. The Solution: Visual Action Models (VAM)
**Intercept** bypasses the underlying code entirely. Instead of hooking into the OS's accessibility tree or the browser's DOM, it uses **Google Gemini 1.5 Pro** to "see" the screen exactly as a human does.

By treating the UI as a visual input, Intercept converts natural language intents (e.g., "Find the cheapest shoes on Amazon") into coordinate-based mouse clicks and keystrokes. It is resilient to code changes because it relies on visual semantic understanding, not rigid code selectors.

## 3. Target Audience
*   **AI Researchers**: Exploring the limits of Multimodal LLMs in dynamic environments.
*   **Accessibility Engineers**: Building next-generation "Switch Access" tools that understand context.
*   **DevOps & QA**: Creating self-healing end-to-end tests that verify visual correctness, not just code correctness.

## 4. Interesting Use Cases

### ♿ Context-Aware Accessibility
*   **Scenario**: A user with limited mobility uses a single switch device.
*   **Action**: "Play some jazz."
*   **Intercept**: Visually identifies the Spotify icon, clicks it, waits for load, finds the search bar, types "Jazz", and clicks the first playlist. No custom integration required.

### � Legacy Enterprise Automation
*   **Scenario**: An insurance company uses a VB6 application from 1998 to process claims. It has no API.
*   **Intercept**: Can visually navigate the archaic menus and input fields just like a human operator, automating data entry tasks that were previously impossible to script.

### 🧪 Visual QA Testing
*   **Scenario**: Verifying a frontend deployment.
*   **Action**: "Check if the 'Buy Now' button is visible and red."
*   **Intercept**: Uses its vision capabilities to assert visual states that code-based tests might miss (e.g., a button covered by a z-index overlay).

---

## 5. Technical Implementation

### Architecture
The system uses a **Planner-Actor** loop:
1.  **Observation**: The Client (`intercept/client`) captures a screenshot (`mss`) and sends it to the Server.
2.  **Reasoning**: The Server (`intercept/server`) constructs a multimodal prompt for **Gemini 1.5 Pro**, including the screenshot and the user's high-level goal.
3.  **Memory**:
    *   **Vector Store (Firestore)**: Retrieves similar past actions to "few-shot" prompt the model, improving reliability over time.
    *   **Transactional Log (Spanner)**: Records every decision for auditability and rollback capabilities.
4.  **Guardrails**: A `GuardrailService` intercepts the prompt and response to block dangerous commands (e.g., `rm -rf`) and redact PII before logging.
5.  **Execution**: The Client receives the coordinate action (`click(x,y)`) and executes it via `pyautogui`.

### Tech Stack
*   **Model**: Google Gemini 1.5 Pro (Vision + Text)
*   **Backend**: FastAPI (Python 3.10+)
*   **State Management**: Google Cloud Spanner (Transactional Consistency)
*   **Knowledge Retrieval**: Google Cloud Firestore (Vector Search)
*   **Client Automation**: PyAutoGUI, MSS

---

## 6. Getting Started

### Prerequisites
*   Python 3.10+
*   Google Cloud Project with Gemini, Spanner, and Firestore APIs enabled.

### Installation

1.  **Clone & Setup**
    ```bash
    git clone https://github.com/DEVELOPER-DEEVEN/operator.git
    cd operator
    python -m venv venv
    source venv/bin/activate  # or .\venv\Scripts\activate on Windows
    pip install -r requirements.txt
    ```

2.  **Environment Variables**
    ```bash
    export GOOGLE_API_KEY="your_gemini_key"
    export GOOGLE_CLOUD_PROJECT="your_project_id"
    ```

3.  **Run Server**
    ```bash
    uvicorn intercept.server.main:app --reload
    ```

4.  **Run Client**
    ```bash
    python -m intercept.client.agent "open notepad and type hello world"
    ```

---

## 7. Future Scope
*   **Continuous Learning**: Automatically adding successful execution traces to the Vector Store to improve autonomy.
*   **Edge Deployment**: Distilling the reasoning model to run locally for lower latency.
*   **Multi-Agent Collaboration**: Multiple Intercept agents coordinating on different machines to perform complex workflows.
