# WhisperWit Chrome Extension MVP Specification

## Extension Structure

```
extension/
├── manifest.json     # Extension configuration
├── popup/
│   ├── popup.html   # Simple UI
│   ├── popup.js     # UI logic
│   └── styles.css   # Basic styling
├── background.js    # Audio recording & API calls
└── content.js       # Text insertion
```

## Component Details

### 1. manifest.json
```json
{
  "manifest_version": 3,
  "name": "WhisperWit",
  "version": "1.0.0",
  "permissions": [
    "activeTab",
    "storage",
    "audioCapture"
  ],
  "action": {
    "default_popup": "popup/popup.html"
  },
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [{
    "matches": ["<all_urls>"],
    "js": ["content.js"]
  }]
}
```

### 2. popup.html
```html
<!DOCTYPE html>
<html>
<head>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <div class="container">
    <button id="recordButton">Record</button>
    <div id="status"></div>
    <div class="settings">
      <select id="language">
        <option value="en">English</option>
        <option value="da">Danish</option>
      </select>
      <input type="text" id="apiKey" placeholder="OpenAI API Key">
    </div>
  </div>
  <script src="popup.js"></script>
</body>
</html>
```

### 3. background.js
```javascript
// Audio recording and API handling
class WhisperWit {
  constructor() {
    this.isRecording = false;
    this.mediaRecorder = null;
    this.audioChunks = [];
  }

  async startRecording() {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    this.mediaRecorder = new MediaRecorder(stream);
    this.mediaRecorder.ondataavailable = e => this.audioChunks.push(e.data);
    this.mediaRecorder.start();
  }

  async stopRecording() {
    return new Promise(resolve => {
      this.mediaRecorder.onstop = async () => {
        const audioBlob = new Blob(this.audioChunks);
        const text = await this.transcribe(audioBlob);
        resolve(text);
      };
      this.mediaRecorder.stop();
    });
  }

  async transcribe(audioBlob) {
    const formData = new FormData();
    formData.append('file', audioBlob);
    formData.append('model', 'whisper-1');

    const response = await fetch('https://api.openai.com/v1/audio/transcriptions', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${await this.getApiKey()}`
      },
      body: formData
    });

    const result = await response.json();
    return result.text;
  }
}
```

### 4. content.js
```javascript
// Text insertion
chrome.runtime.onMessage.addListener((request, sender, sendResponse) => {
  if (request.action === 'insertText') {
    const activeElement = document.activeElement;
    if (activeElement.isContentEditable || 
        activeElement.tagName === 'INPUT' || 
        activeElement.tagName === 'TEXTAREA') {
      
      const start = activeElement.selectionStart;
      const end = activeElement.selectionEnd;
      const text = activeElement.value;
      const newText = text.substring(0, start) + 
                     request.text + 
                     text.substring(end);
      
      activeElement.value = newText;
      activeElement.selectionStart = 
      activeElement.selectionEnd = start + request.text.length;
    }
  }
});
```

### 5. popup.js
```javascript
document.getElementById('recordButton').addEventListener('click', async () => {
  const button = document.getElementById('recordButton');
  const status = document.getElementById('status');
  
  if (button.textContent === 'Record') {
    button.textContent = 'Stop';
    status.textContent = 'Recording...';
    chrome.runtime.sendMessage({ action: 'startRecording' });
  } else {
    button.textContent = 'Record';
    status.textContent = 'Processing...';
    const text = await chrome.runtime.sendMessage({ action: 'stopRecording' });
    chrome.tabs.query({active: true, currentWindow: true}, tabs => {
      chrome.tabs.sendMessage(tabs[0].id, { action: 'insertText', text });
    });
    status.textContent = 'Done!';
  }
});
```

### 6. styles.css
```css
.container {
  width: 300px;
  padding: 16px;
}

#recordButton {
  width: 100%;
  padding: 8px;
  margin-bottom: 8px;
  background: #4299E1;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

#status {
  text-align: center;
  margin: 8px 0;
  color: #666;
}

.settings {
  margin-top: 16px;
}

select, input {
  width: 100%;
  padding: 8px;
  margin: 4px 0;
  border: 1px solid #ddd;
  border-radius: 4px;
}
```

## Implementation Steps

1. Create extension directory structure
2. Implement basic UI
3. Add audio recording
4. Integrate Whisper API
5. Add text insertion
6. Test and debug
7. Package for store

Would you like me to start implementing any of these components?