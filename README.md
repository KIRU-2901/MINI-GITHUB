from flask import Flask, render_template, request, redirect, session, url_for
import sqlite3
import os

app = Flask(__name__)
app.secret_key = 'your_secret_key'  # Change this to a secure random key for production

# Database connection helper
def get_db_connection():
    conn = sqlite3.connect('mini_github.db')
    conn.row_factory = sqlite3.Row
    return conn

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username'].strip()
        email = request.form['email'].strip()
        password = request.form['password'].strip()

        # Basic validation can be added here

        try:
            conn = get_db_connection()
            conn.execute('INSERT INTO users (username, email, password) VALUES (?, ?, ?)',
                         (username, email, password))
            conn.commit()
        except sqlite3.IntegrityError:
            # This happens if email already exists (due to UNIQUE constraint)
            return "Email already registered. Please use a different email."
        finally:
            conn.close()

        return redirect(url_for('login'))

    return render_template('register.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        email = request.form['email'].strip()
        password = request.form['password'].strip()

        conn = get_db_connection()
        user = conn.execute('SELECT * FROM users WHERE email = ? AND password = ?',
                            (email, password)).fetchone()
        conn.close()

        if user:
            session['user_id'] = user['user_id']
            session['username'] = user['username']
            return redirect(url_for('dashboard'))
        else:
            return "Invalid credentials. Please try again."

    return render_template('login.html')

@app.route('/dashboard')
def dashboard():
    if 'user_id' not in session:
        return redirect(url_for('login'))

    conn = get_db_connection()
    repos = conn.execute('SELECT * FROM repositories WHERE user_id = ?', (session['user_id'],)).fetchall()
    conn.close()

    return render_template('dashboard.html', repos=repos)

@app.route('/create_repo', methods=['GET', 'POST'])
def create_repo():
    if 'user_id' not in session:
        return redirect(url_for('login'))

    if request.method == 'POST':
        repo_name = request.form['repo_name'].strip()
        description = request.form['description'].strip()

        conn = get_db_connection()
        conn.execute('INSERT INTO repositories (repo_name, description, user_id) VALUES (?, ?, ?)',
                     (repo_name, description, session['user_id']))
        conn.commit()
        conn.close()

        return redirect(url_for('dashboard'))

    return render_template('create_repo.html')

@app.route('/upload_file/<int:repo_id>', methods=['GET', 'POST'])
def upload_file(repo_id):
    if 'user_id' not in session:
        return redirect(url_for('login'))

    if request.method == 'POST':
        file_name = request.form['file_name'].strip()
        content = request.form['content']

        # Optionally check if repo_id belongs to this user here for extra security

        conn = get_db_connection()
        conn.execute('INSERT INTO files (file_name, content, repo_id) VALUES (?, ?, ?)',
                     (file_name, content, repo_id))
        conn.commit()
        conn.close()

        return redirect(url_for('view_repo', repo_id=repo_id))

    return render_template('upload_file.html', repo_id=repo_id)

@app.route('/view_repo/<int:repo_id>')
def view_repo(repo_id):
    if 'user_id' not in session:
        return redirect(url_for('login'))

    conn = get_db_connection()
    repo = conn.execute('SELECT * FROM repositories WHERE repo_id = ? AND user_id = ?',
                        (repo_id, session['user_id'])).fetchone()
    files = conn.execute('SELECT * FROM files WHERE repo_id = ?', (repo_id,)).fetchall()
    conn.close()

    if repo is None:
        return "Repository not found or you don't have access to it."

    return render_template('view_repo.html', repo=repo, files=files)

@app.route('/view_file/<int:file_id>')
def view_file(file_id):
    if 'user_id' not in session:
        return redirect(url_for('login'))

    conn = get_db_connection()
    file = conn.execute('''
        SELECT * FROM files 
        WHERE file_id = ? AND repo_id IN (
            SELECT repo_id FROM repositories WHERE user_id = ?
        )
    ''', (file_id, session['user_id'])).fetchone()
    conn.close()

    print("DEBUG: file fetched:", file)  # Debug print to terminal

    if file is None:
        return "File not found or you don't have access to it."

    return render_template('view_file.html', file=file)


@app.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('index'))

if __name__ == '__main__':
    if not os.path.exists('mini_github.db'):
        conn = get_db_connection()
        with open('schema.sql') as f:
            conn.executescript(f.read())
        conn.commit()
        conn.close()

    app.run(debug=True)
