from flask import Flask, render_template_string, request, session, redirect, url_for, jsonify
import sqlite3
import os

app = Flask(__name__)
app.secret_key = "secret123"

DB_FILE = "ali_gram.db"

# Инициализация базы
def init_db():
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS users (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    username TEXT UNIQUE,
                    password TEXT
                )''')
    c.execute('''CREATE TABLE IF NOT EXISTS messages (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    sender TEXT,
                    receiver TEXT,
                    text TEXT
                )''')
    conn.commit()
    conn.close()

init_db()

# Главная страница / список пользователей
@app.route("/")
def index():
    if "username" not in session:
        return redirect(url_for("login"))

    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("SELECT username FROM users WHERE username != ?", (session["username"],))
    users = [u[0] for u in c.fetchall()]
    conn.close()

    return render_template_string('''
    <!doctype html>
    <html>
    <head>
        <title>Ali-gram</title>
        <script>
            var currentChat = "";

            function selectUser(user) {
                currentChat = user;
                document.getElementById("chatWith").innerText = "Чат с " + user;
                fetchMessages();
            }

            async function fetchMessages() {
                if(currentChat === "") return;
                let response = await fetch('/messages?user=' + currentChat);
                let data = await response.json();
                let chatDiv = document.getElementById('chat');
                chatDiv.innerHTML = "";
                data.forEach(msg => {
                    let p = document.createElement("p");
                    p.innerHTML = "<b>" + msg.sender + ":</b> " + msg.text;
                    chatDiv.appendChild(p);
                });
                chatDiv.scrollTop = chatDiv.scrollHeight;
            }

            async function sendMessage() {
                let input = document.getElementById('message');
                let msg = input.value;
                if(msg.trim() === "" || currentChat === "") return;
                await fetch('/send', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({receiver: currentChat, text: msg})
                });
                input.value = "";
                fetchMessages();
            }

            setInterval(fetchMessages, 1500);
            window.onload = fetchMessages;
        </script>
    </head>
    <body>
        <h2>Привет, {{username}}! <a href="{{ url_for('logout') }}">Выйти</a></h2>
        <h3>Пользователи:</h3>
        {% for user in users %}
            <button onclick="selectUser('{{user}}')">{{user}}</button>
        {% endfor %}
        <h3 id="chatWith">Выберите пользователя для чата</h3>
        <div id="chat" style="border:1px solid #000; height:300px; width:400px; overflow:auto; padding:5px;"></div>
        <input type="text" id="message" style="width:300px;">
        <button onclick="sendMessage()">Отправить</button>
    </body>
    </html>
    ''', username=session["username"], users=users)

@app.route("/messages")
def messages():
    if "username" not in session:
        return jsonify([])
    other = request.args.get("user")
    if not other:
        return jsonify([])
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute('''SELECT sender, text FROM messages
                 WHERE (sender=? AND receiver=?) OR (sender=? AND receiver=?)
                 ORDER BY id ASC''', (session["username"], other, other, session["username"]))
    msgs = c.fetchall()
    conn.close()
    return jsonify([{"sender": s, "text": t} for s, t in msgs])

@app.route("/send", methods=["POST"])
def send():
    if "username" not in session:
        return jsonify({"error": "not logged in"}), 403
    data = request.get_json()
    receiver = data.get("receiver")
    text = data.get("text", "")
    if not receiver or text.strip() == "":
        return jsonify({"error": "empty"}), 400
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("INSERT INTO messages (sender, receiver, text) VALUES (?, ?, ?)",
              (session["username"], receiver, text))
    conn.commit()
    conn.close()
    return jsonify({"success": True})

@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "POST":
        username = request.form.get("username")
        password = request.form.get("password")
        if not username or not password:
            return "Заполните все поля!"
        conn = sqlite3.connect(DB_FILE)
        c = conn.cursor()
        try:
            c.execute("INSERT INTO users (username, password) VALUES (?, ?)", (username, password))
            conn.commit()
        except sqlite3.IntegrityError:
            conn.close()
            return "Пользователь уже существует!"
        conn.close()
        return redirect(url_for("login"))
    return '''
    <h2>Регистрация</h2>
    <form method="post">
        Имя пользователя: <input type="text" name="username"><br>
        Пароль: <input type="password" name="password"><br>
        <button type="submit">Зарегистрироваться</button>
    </form>
    <a href="/login">Войти</a>
    '''

@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        username = request.form.get("username")
        password = request.form.get("password")
        conn = sqlite3.connect(DB_FILE)
        c = conn.cursor()
        c.execute("SELECT * FROM users WHERE username=? AND password=?", (username, password))
        user = c.fetchone()
        conn.close()
        if user:
            session["username"] = username
            return redirect(url_for("index"))
        return "Неверный логин или пароль!"
    return '''
    <h2>Вход</h2>
    <form method="post">
        Имя пользователя: <input type="text" name="username"><br>
        Пароль: <input type="password" name="password"><br>
        <button type="submit">Войти</button>
    </form>
    <a href="/register">Регистрация</a>
    '''

@app.route("/logout")
def logout():
    session.pop("username", None)
    return redirect(url_for("login"))

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    app.run(host="0.0.0.0", port=port)
