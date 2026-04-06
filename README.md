<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Physics Support</title>
    <link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@500;700&display=swap" rel="stylesheet">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.min.js"></script>
    <style>
        :root {
            --bg-color: #0d0d0d;
            --panel-bg: #1a1a1a;
            --accent: #00ffcc;
            --alt-accent: #ff0055;
            --text-color: #e0e0e0;
        }
        body {
            background-color: var(--bg-color);
            color: var(--text-color);
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            margin: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
            height: 100vh;
            overflow: hidden;
        }
        .header {
            font-family: 'Orbitron', sans-serif;
            font-size: 2.5rem;
            margin: 20px 0;
            display: flex;
            align-items: center;
            gap: 15px;
            color: var(--accent);
            text-shadow: 0 0 10px rgba(0, 255, 204, 0.5);
        }
        .ai-logo { font-size: 3rem; }
        
        /* Views */
        #chat-view, #admin-view {
            width: 100%;
            max-width: 800px;
            display: none;
            flex-direction: column;
            align-items: center;
            flex-grow: 1;
        }

        /* Chat UI */
        .chat-box {
            width: 90%;
            flex-grow: 1;
            background: var(--panel-bg);
            padding: 20px;
            border-radius: 10px;
            border: 1px solid #333;
            overflow-y: auto;
            margin-bottom: 20px;
            box-shadow: inset 0 0 10px #000;
        }
        .message { margin-bottom: 15px; line-height: 1.5; }
        .user-msg { color: #fff; border-left: 3px solid #777; padding-left: 10px; }
        .ai-msg { color: var(--accent); border-left: 3px solid var(--accent); padding-left: 10px; }
        
        .input-area {
            display: flex;
            width: 90%;
            margin-bottom: 30px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.5);
        }
        input[type="text"] {
            flex-grow: 1;
            padding: 15px;
            background: #222;
            color: #fff;
            border: 1px solid #444;
            border-radius: 5px 0 0 5px;
            font-size: 1rem;
        }
        input[type="text"]:focus { outline: none; border-color: var(--accent); }
        button {
            padding: 15px 20px;
            border: none;
            cursor: pointer;
            font-weight: bold;
            font-size: 1rem;
            transition: 0.2s;
        }
        button:hover { filter: brightness(1.2); }
        .mic-btn { background: var(--alt-accent); color: white; }
        .send-btn { background: var(--accent); color: #000; border-radius: 0 5px 5px 0; }

        /* Alternate Answer Logic */
        .alt-arrow {
            cursor: pointer;
            color: var(--alt-accent);
            font-size: 1.2rem;
            margin-left: 10px;
            display: inline-block;
            transition: transform 0.3s ease;
        }
        .alt-answer {
            display: none;
            margin-top: 10px;
            padding: 15px;
            background: #252525;
            border-left: 3px solid var(--alt-accent);
            color: #bbb;
            font-size: 0.95em;
            border-radius: 0 5px 5px 0;
        }

        /* Admin UI */
        .admin-panel {
            width: 90%;
            background: var(--panel-bg);
            padding: 30px;
            border-radius: 10px;
            border: 1px dashed var(--alt-accent);
        }
        .admin-panel h2 { color: var(--alt-accent); font-family: 'Orbitron', sans-serif; }
        .admin-input-group { margin-bottom: 20px; }
        .admin-panel input[type="file"], .admin-panel input[type="url"] {
            width: 100%;
            padding: 10px;
            margin-top: 5px;
            background: #111;
            color: #fff;
            border: 1px solid #444;
            box-sizing: border-box;
        }
        .status { margin-top: 15px; color: #aaa; }
    </style>
</head>
<body>

    <div class="header">
        <span class="ai-logo">⚙️</span> PHYSICS SUPPORT
    </div>

    <div id="chat-view">
        <div class="chat-box" id="chat-box">
            <div class="message ai-msg">Systems online. Awaiting physics queries...</div>
        </div>
        <div class="input-area">
            <input type="text" id="user-input" placeholder="Type or speak your question...">
            <button class="mic-btn" onclick="startDictation()" title="Click to speak">🎤</button>
            <button class="send-btn" onclick="processQuestion()">Send</button>
        </div>
    </div>

    <div id="admin-view">
        <div class="admin-panel">
            <h2>Admin: Secret Vault</h2>
            <p>Upload a physics source PDF. The AI will extract and search this text.</p>
            
            <div class="admin-input-group">
                <label>Upload PDF File:</label>
                <input type="file" id="pdfFile" accept="application/pdf">
                <button class="send-btn" style="border-radius: 5px; margin-top: 10px;" onclick="loadPdfFromFile()">Extract from File</button>
            </div>

            <div class="admin-input-group">
                <label>Or Paste PDF URL (Note: Many servers block direct JS access due to CORS):</label>
                <input type="url" id="pdfUrl" placeholder="https://example.com/physics.pdf">
                <button class="send-btn" style="border-radius: 5px; margin-top: 10px;" onclick="loadPdfFromUrl()">Extract from URL</button>
            </div>

            <div class="status" id="admin-status">No PDF loaded currently.</div>
        </div>
    </div>

    <script>
        // Setup PDF.js worker
        pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.worker.min.js';

        let extractedKnowledge = ""; // This holds the PDF text

        // --- ROUTING LOGIC ---
        function checkRoute() {
            if (window.location.hash === '#/secretvault') {
                document.getElementById('chat-view').style.display = 'none';
                document.getElementById('admin-view').style.display = 'flex';
            } else {
                document.getElementById('admin-view').style.display = 'none';
                document.getElementById('chat-view').style.display = 'flex';
            }
        }
        window.addEventListener('hashchange', checkRoute);
        window.addEventListener('load', checkRoute);

        // --- PDF EXTRACTION LOGIC ---
        async function extractTextFromPDF(pdfDoc) {
            document.getElementById('admin-status').innerText = "Extracting text... please wait.";
            let fullText = "";
            for (let i = 1; i <= pdfDoc.numPages; i++) {
                const page = await pdfDoc.getPage(i);
                const textContent = await page.getTextContent();
                const pageText = textContent.items.map(item => item.str).join(' ');
                fullText += pageText + " ";
            }
            extractedKnowledge = fullText.replace(/\s+/g, ' '); // Clean up extra spaces
            
            // Save to localStorage so it persists if the user refreshes or switches to the chat view
            localStorage.setItem('physicsPdfText', extractedKnowledge);
            document.getElementById('admin-status').innerText = "Extraction complete! Knowledge base updated. Remove '#/secretvault' from URL to test.";
        }

        async function loadPdfFromFile() {
            const fileInput = document.getElementById('pdfFile').files[0];
            if (!fileInput) return alert("Please select a file.");
            
            const fileReader = new FileReader();
            fileReader.onload = async function() {
                const typedarray = new Uint8Array(this.result);
                const pdfDoc = await pdfjsLib.getDocument(typedarray).promise;
                extractTextFromPDF(pdfDoc);
            };
            fileReader.readAsArrayBuffer(fileInput);
        }

        async function loadPdfFromUrl() {
            const url = document.getElementById('pdfUrl').value;
            if (!url) return alert("Please enter a URL.");
            try {
                const pdfDoc = await pdfjsLib.getDocument(url).promise;
                extractTextFromPDF(pdfDoc);
            } catch (error) {
                document.getElementById('admin-status').innerText = "Error: Could not load URL. The server likely blocks cross-origin requests (CORS). Try downloading the file and uploading it instead.";
            }
        }

        // --- CHAT AND "AI" LOGIC ---
        // Load existing knowledge on startup
        window.onload = () => {
            checkRoute();
            const savedText = localStorage.getItem('physicsPdfText');
            if (savedText) extractedKnowledge = savedText;
        };

        const chatBox = document.getElementById('chat-box');
        const userInput = document.getElementById('user-input');

        // Allow pressing Enter to send
        userInput.addEventListener("keypress", function(event) {
            if (event.key === "Enter") {
                event.preventDefault();
                processQuestion();
            }
        });

        function startDictation() {
            if (window.hasOwnProperty('webkitSpeechRecognition') || window.hasOwnProperty('SpeechRecognition')) {
                const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
                const recognition = new SpeechRecognition();
                recognition.continuous = false;
                recognition.interimResults = false;
                recognition.lang = "en-US";
                
                userInput.placeholder = "Listening...";
                recognition.start();

                recognition.onresult = function(e) {
                    userInput.value = e.results[0][0].transcript;
                    userInput.placeholder = "Type or speak your question...";
                    recognition.stop();
                };
                recognition.onerror = function() {
                    userInput.placeholder = "Error listening. Type instead.";
                }
            } else {
                alert("Voice recognition is not supported in this browser. Please use Chrome or Edge.");
            }
        }

        function processQuestion() {
            const question = userInput.value.trim();
            if (!question) return;

            // Add user message to UI
            addMessageToUI('You', question, 'user-msg');
            userInput.value = '';

            // Simulated AI Search Logic
            if (!extractedKnowledge) {
                addMessageToUI('AI', "I am working on it. (Knowledge base is empty. Admin needs to upload a PDF).", 'ai-msg');
                return;
            }

            // Clean the question to find core keywords (removing common words)
            const stopWords = ['what', 'is', 'the', 'a', 'an', 'how', 'why', 'explain', 'define', 'of', 'in', 'to'];
            const keywords = question.toLowerCase().replace(/[?.,]/g, '').split(' ')
                                     .filter(word => !stopWords.includes(word) && word.length > 2);

            if (keywords.length === 0) {
                addMessageToUI('AI', "Could you be more specific?", 'ai-msg');
                return;
            }

            // Split the PDF text into sentences to search for context
            const sentences = extractedKnowledge.match(/[^.!?]+[.!?]+/g) || [];
            
            let matchedSentences = [];
            
            // Search for sentences containing the keywords
            sentences.forEach(sentence => {
                const lowerSentence = sentence.toLowerCase();
                // Check if the sentence contains at least one of the main keywords
                const hasMatch = keywords.some(kw => lowerSentence.includes(kw));
                if (hasMatch) {
                    matchedSentences.push(sentence.trim());
                }
            });

            if (matchedSentences.length > 0) {
                // Return the first match as the main answer
                const mainAnswer = matchedSentences[0];
                
                // If there are more matches, use the second one as an alternative
                const altAnswer = matchedSentences.length > 1 ? matchedSentences[1] : null;
                
                addAIMessageWithAlt(mainAnswer, altAnswer);
            } else {
                addMessageToUI('AI', "I am working on it.", 'ai-msg');
            }
        }

        function addMessageToUI(sender, text, className) {
            chatBox.innerHTML += `<div class="message ${className}"><b>${sender}:</b> ${text}</div>`;
            chatBox.scrollTop = chatBox.scrollHeight;
        }

        function addAIMessageWithAlt(main, alt) {
            let html = `<div class="message ai-msg"><b>AI:</b> ${main}`;
            
            if (alt) {
                const uniqueId = 'alt-' + Math.random().toString(36).substr(2, 9);
                html += ` <span class="alt-arrow" onclick="toggleAlt('${uniqueId}', this)" title="Show alternative answer">▼</span>`;
                html += `<div class="alt-answer" id="${uniqueId}"><b>Alternative perspective:</b> ${alt}</div>`;
            }
            
            html += `</div>`;
            chatBox.innerHTML += html;
            chatBox.scrollTop = chatBox.scrollHeight;
        }

        function toggleAlt(id, arrowElement) {
            const altDiv = document.getElementById(id);
            if (altDiv.style.display === "block") {
                altDiv.style.display = "none";
                arrowElement.style.transform = "rotate(0deg)";
            } else {
                altDiv.style.display = "block";
                arrowElement.style.transform = "rotate(180deg)";
            }
            chatBox.scrollTop = chatBox.scrollHeight;
        }
    </script>
</body>
</html>
