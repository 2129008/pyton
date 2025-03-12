# pyton
pip install Flask
/music-streaming-platform
    /music
        song1.mp3
        song2.mp3
    /static
        /css
        /js
    app.py
    database.db

   from flask import Flask, render_template, send_from_directory, request, redirect, url_for, session
import os
import sqlite3
from werkzeug.utils import secure_filename

app = Flask(__name__)
app.secret_key = 'secret_key_here'

# Set up music folder and allowed file extensions
MUSIC_FOLDER = './music'
ALLOWED_EXTENSIONS = {'mp3'}

# Database setup
def init_db():
    conn = sqlite3.connect('database.db')
    cursor = conn.cursor()
    cursor.execute('''CREATE TABLE IF NOT EXISTS users (
                        id INTEGER PRIMARY KEY,
                        username TEXT,
                        password TEXT)''')
    cursor.execute('''CREATE TABLE IF NOT EXISTS songs (
                        id INTEGER PRIMARY KEY,
                        title TEXT,
                        artist TEXT,
                        file_name TEXT)''')
    conn.commit()
    conn.close()

# Initialize database
init_db()

# Function to check if the file is an allowed extension
def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

# Route for the home page
@app.route('/')
def index():
    conn = sqlite3.connect('database.db')
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM songs')
    songs = cursor.fetchall()
    conn.close()
    return render_template('index.html', songs=songs)

# Route to stream a song
@app.route('/stream/<filename>')
def stream_song(filename):
    return send_from_directory(MUSIC_FOLDER, filename)

# Route to register a user
@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        
        conn = sqlite3.connect('database.db')
        cursor = conn.cursor()
        cursor.execute('SELECT * FROM users WHERE username = ?', (username,))
        user = cursor.fetchone()
        
        if user:
            return "Username already exists!"
        
        cursor.execute('INSERT INTO users (username, password) VALUES (?, ?)', (username, password))
        conn.commit()
        conn.close()
        return redirect(url_for('login'))
    
    return render_template('register.html')

# Route to login
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        
        conn = sqlite3.connect('database.db')
        cursor = conn.cursor()
        cursor.execute('SELECT * FROM users WHERE username = ? AND password = ?', (username, password))
        user = cursor.fetchone()
        
        if user:
            session['user_id'] = user[0]
            return redirect(url_for('index'))
        else:
            return "Invalid credentials, please try again."
    
    return render_template('login.html')

# Route to upload a song (for admin purposes)
@app.route('/upload', methods=['GET', 'POST'])
def upload():
    if 'user_id' not in session:
        return redirect(url_for('login'))
    
    if request.method == 'POST':
        if 'file' not in request.files:
            return "No file part"
        
        file = request.files['file']
        if file and allowed_file(file.filename):
            filename = secure_filename(file.filename)
            file.save(os.path.join(MUSIC_FOLDER, filename))
            
            title = request.form['title']
            artist = request.form['artist']
            
            conn = sqlite3.connect('database.db')
            cursor = conn.cursor()
            cursor.execute('INSERT INTO songs (title, artist, file_name) VALUES (?, ?, ?)', (title, artist, filename))
            conn.commit()
            conn.close()
            
            return redirect(url_for('index'))
    
    return render_template('upload.html')

if __name__ == '__main__':
    app.run(debug=True)

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Music Streaming</title>
</head>
<body>
    <h1>Welcome to the Music Streaming Platform</h1>
    <ul>
        {% for song in songs %}
            <li>
                <strong>{{ song[1] }}</strong> by {{ song[2] }}
                <a href="{{ url_for('stream_song', filename=song[3]) }}">Play</a>
            </li>
        {% endfor %}
    </ul>
    <a href="{{ url_for('upload') }}">Upload a new song</a>
    <a href="{{ url_for('register') }}">Register</a>
    <a href="{{ url_for('login') }}">Login</a>
</body>
</html>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Register</title>
</head>
<body>
    <h1>Register</h1>
    <form action="{{ url_for('register') }}" method="POST">
        <label for="username">Username:</label>
        <input type="text" name="username" required><br>
        <label for="password">Password:</label>
        <input type="password" name="password" required><br>
        <button type="submit">Register</button>
    </form>
</body>
</html>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login</title>
</head>
<body>
    <h1>Login</h1>
    <form action="{{ url_for('login') }}" method="POST">
        <label for="username">Username:</label>
        <input type="text" name="username" required><br>
        <label for="password">Password:</label>
        <input type="password" name="password" required><br>
        <button type="submit">Login</button>
    </form>
</body>
</html>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Upload Song</title>
</head>
<body>
    <h1>Upload a New Song</h1>
    <form action="{{ url_for('upload') }}" method="POST" enctype="multipart/form-data">
        <label for="title">Song Title:</label>
        <input type="text" name="title" required><br>
        <label for="artist">Artist:</label>
        <input type="text" name="artist" required><br>
        <label for="file">Choose Song:</label>
        <input type="file" name="file" required><br>
        <button type="submit">Upload</button>
    </form>
</body>
</html>
python app.py
