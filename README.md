Here’s a **step-by-step guide** to build the **Smart Autofill Chrome Extension** with local server (Flask/FastAPI) and Gemini/OpenAI APIs:

---

### **Step 1: Get API Keys**
#### **For OpenAI (GPT-4)**
1. Go to [OpenAI Platform](https://platform.openai.com/)
2. Sign up/log in → Click **"API Keys"** in left sidebar.
3. Click **"Create new secret key"** → Copy it (save securely, it won’t be shown again).

#### **For Google Gemini**
1. Go to [Google AI Studio](https://aistudio.google.com/)
2. Click **"Get API Key"** → Create a new key under **API Keys**.
3. Copy the key (enable the API if prompted).

---

### **Step 2: Set Up Local Server (Python)**
#### **Install Dependencies**
```bash
pip install flask google-generativeai openai python-dotenv
```

#### **Create `.env` File (Store API Keys)**
```plaintext
# .env
OPENAI_API_KEY="your_openai_key"
GEMINI_API_KEY="your_gemini_key"
```

#### **Flask Server Code (`server.py`)**
```python
from flask import Flask, request, jsonify
from dotenv import load_dotenv
import os
import google.generativeai as genai
import openai

load_dotenv()  # Load .env

app = Flask(__name__)

# Configure APIs
genai.configure(api_key=os.getenv("GEMINI_API_KEY"))
openai.api_key = os.getenv("OPENAI_API_KEY")

@app.route('/autofill', methods=['POST'])
def autofill():
    data = request.json
    field_name = data.get("field_name")
    user_context = data.get("context", "")

    # Choose either Gemini or OpenAI
    # Option 1: Gemini
    model = genai.GenerativeModel('gemini-pro')
    response = model.generate_content(
        f"Generate a professional 50-word response for {field_name}. Context: {user_context}"
    )
    result = response.text

    # Option 2: OpenAI
    # response = openai.ChatCompletion.create(
    #     model="gpt-4",
    #     messages=[{"role": "user", "content": f"Generate a professional 50-word response for {field_name}. Context: {user_context}"}]
    # )
    # result = response.choices[0].message.content

    return jsonify({"response": result})

if __name__ == '__main__':
    app.run(port=5000, debug=True)
```

#### **Run the Server**
```bash
python server.py
```
→ Server runs at `http://localhost:5000`.

---

### **Step 3: Build Chrome Extension**
#### **Folder Structure**
```
autofill-extension/
├── manifest.json
├── background.js
├── content.js
└── popup.html
```

#### **1. `manifest.json` (Manifest V3)**
```json
{
  "manifest_version": 3,
  "name": "Smart Autofill",
  "version": "1.0",
  "action": {
    "default_popup": "popup.html",
    "default_icon": "icon.png"
  },
  "permissions": ["activeTab", "contextMenus", "scripting"],
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [{
    "matches": ["<all_urls>"],
    "js": ["content.js"]
  }]
}
```

#### **2. `background.js` (Handles API Calls)**
```javascript
chrome.contextMenus.create({
  id: "autofill",
  title: "Smart Autofill",
  contexts: ["editable"]  // Works on text inputs/textarea
});

chrome.contextMenus.onClicked.addListener(async (info, tab) => {
  if (info.menuItemId === "autofill") {
    // Inject content script to get selected field name
    chrome.scripting.executeScript({
      target: { tabId: tab.id },
      files: ["content.js"]
    });
  }
});

// Handle responses from content script
chrome.runtime.onMessage.addListener((request, sender, sendResponse) => {
  if (request.action === "autofill") {
    fetch("http://localhost:5000/autofill", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        field_name: request.fieldName,
        context: request.context
      })
    })
    .then(res => res.json())
    .then(data => sendResponse(data))
    .catch(console.error);
    return true;  // Keep message channel open
  }
});
```

#### **3. `content.js` (Gets Field Info)**
```javascript
// Get the active input field name (e.g., "work_experience")
const fieldName = document.activeElement.placeholder || 
                  document.activeElement.name || 
                  "this field";

// Send to background.js
chrome.runtime.sendMessage({
  action: "autofill",
  fieldName: fieldName,
  context: window.getSelection().toString()  // Optional user context
}, (response) => {
  if (response?.response) {
    document.activeElement.value = response.response;  // Autofill!
  }
});
```

#### **4. `popup.html` (Optional UI)**
```html
<!DOCTYPE html>
<html>
<body>
  <button id="autofill-btn">Autofill Selected Field</button>
  <script src="background.js"></script>
</body>
</html>
```

---

### **Step 4: Load Extension in Chrome**
1. Go to `chrome://extensions/`.
2. Enable **Developer mode** (top-right toggle).
3. Click **"Load unpacked"** → Select your `autofill-extension` folder.

---

### **Step 5: Test It!**
1. Open any website with a text field (e.g., LinkedIn job application).
2. Right-click a text field → Select **"Smart Autofill"**.
3. The field auto-populates with a generated response!

---

### **🔧 Troubleshooting**
- **CORS Error**: Add Flask CORS support:
  ```bash
  pip install flask-cors
  ```
  Then in `server.py`:
  ```python
  from flask_cors import CORS
  CORS(app)  # Allow all origins (for local testing)
  ```
- **Extension Not Working**: Check Chrome’s **Inspect Views** (`chrome://extensions/` → click "Service Worker" under your extension).

---

### **🚀 Next Steps**
- Add **custom templates** (e.g., "formal", "casual").
- Support **multi-language** autofill.
- Use **local LLMs** (e.g., Ollama) for privacy.

Want me to refine any part? 😊
