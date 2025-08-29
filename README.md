<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>ChatBot</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      display: flex;
      flex-direction: column;
      height: 100vh;
      margin: 0;
    }
    header {
      background: #10a37f;
      color: #fff;
      padding: 15px;
      text-align: center;
      font-size: 18px;
      font-weight: bold;
    }
    #chat-container {
      flex: 1;
      padding: 15px;
      overflow-y: auto;
      background: #f2f2f2;
    }
    .message {
      max-width: 70%;
      margin: 8px 0;
      padding: 10px 14px;
      border-radius: 10px;
    }
    .user-message {
      background: #10a37f;
      color: white;
      align-self: flex-end;
      margin-left: auto;
    }
    .bot-message {
      background: #e0e0e0;
      color: black;
      align-self: flex-start;
      margin-right: auto;
    }
    .chat-input {
      display: flex;
      border-top: 1px solid #ddd;
      padding: 10px;
      background: white;
    }
    .chat-input input {
      flex: 1;
      padding: 10px;
      border: 1px solid #ccc;
      border-radius: 6px;
      outline: none;
    }
    .chat-input button {
      margin-left: 10px;
      padding: 10px 16px;
      background: #10a37f;
      border: none;
      border-radius: 6px;
      color: white;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <header>🤖 ChatBot</header>
  <div id="chat-container">
    <div class="message bot-message">Hello! I’m your AI assistant.</div>
  </div>

  <div class="chat-input">
    <input type="text" id="user-input" placeholder="Type your message...">
    <button onclick="sendMessage()">Send</button>
  </div>

  <script>
    const chatContainer = document.getElementById("chat-container");
    const userInput = document.getElementById("user-input");

    function sendMessage() {
      const text = userInput.value.trim();
      if (text === "") return;

      // User message
      const userMsg = document.createElement("div");
      userMsg.className = "message user-message";
      userMsg.innerText = text;
      chatContainer.appendChild(userMsg);

      chatContainer.scrollTop = chatContainer.scrollHeight;
      userInput.value = "";

      // Bot reply
      setTimeout(() => {
        const botMsg = document.createElement("div");
        botMsg.className = "message bot-message";
        botMsg.innerText = getBotReply(text);
        chatContainer.appendChild(botMsg);
        chatContainer.scrollTop = chatContainer.scrollHeight;
      }, 500);
    }

    function getBotReply(input) {
      input = input.toLowerCase();
      if (input.includes("hi") || input.includes("hello")) {
        return "Hello 👋 How are you?";
      } else if (input.includes("how are you")) {
        return "I am fine 😊 Thanks for asking!";
      } else if (input.includes("bye")) {
        return "Goodbye! 👋 See you soon.";
      } else {
        return "I didn’t understand that. Can you rephrase?";
      }
    }

    // Send on Enter
    userInput.addEventListener("keypress", (e) => {
      if (e.key === "Enter") sendMessage();
    });
  </script>
</body>
</html>
