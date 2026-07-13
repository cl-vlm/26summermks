**문제 설명**

**Description
php 7.4로 작성된 페이지입니다.
알맞은 Input 값을 입력하고 플래그를 획득하세요.
플래그 형식은 `DH{}` 입니다.**

- check.php
    
```php
    <html>
    <head>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap.min.css">
    <title>php7cmp4re</title>
    </head>
    <body>
        <!-- Fixed navbar -->
        <nav class="navbar navbar-default navbar-fixed-top">
          <div class="container">
            <div class="navbar-header">
              <a class="navbar-brand" href="/">php7cmp4re</a>
            </div>
            <div id="navbar">
              <ul class="nav navbar-nav">
                <li><a href="/">Index page</a></li>
              </ul>
            </div><!--/.nav-collapse -->
          </div>
        </nav>
        <div class="container">
        <?php
        require_once('flag.php');
        error_reporting(0);
        // POST request
        if ($_SERVER["REQUEST_METHOD"] == "POST") {
          $input_1 = $_POST["input1"] ? $_POST["input1"] : "";
          $input_2 = $_POST["input2"] ? $_POST["input2"] : "";
          sleep(1);
    
          if($input_1 != "" && $input_2 != ""){
            if(strlen($input_1) < 4){
              if($input_1 < "8" && $input_1 < "7.A" && $input_1 > "7.9"){
                if(strlen($input_2) < 3 && strlen($input_2) > 1){
                  if($input_2 < 74 && $input_2 > "74"){
                    echo "</br></br></br><pre>FLAG\n";
                    echo $flag;
                    echo "</pre>";
                  } else echo "<br><br><br><h4>Good try.</h4>";
                } else echo "<br><br><br><h4>Good try.</h4><br>";
              } else echo "<br><br><br><h4>Try again.</h4><br>";
            } else echo "<br><br><br><h4>Try again.</h4><br>";
          } else{
            echo '<br><br><br><h4>Fill the input box.</h4>';
          }
        } else echo "<br><br><br><h3>WHat??!</h3>";
        ?> 
        </div> 
    </body>
    </html>
```
    
- index.php
    
```php
    <html>
    <head>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap.min.css">
    <title>php7cmp4re</title>
    </head>
    <body>
        <!-- Fixed navbar -->
        <nav class="navbar navbar-default navbar-fixed-stop">
          <div class="container">
            <div class="navbar-header">
              <a class="navbar-brand" href="/">php7cmp4re</a>
            </div>
            <div id="navbar">
              <ul class="nav navbar-nav">
                <li><a href="/">index page</a></li>
              </ul>
            </div><!--/.nav-collapse -->
          </div>
        </nav>
        <div class="container">
          <div class="box">
          <h4>Enter the correct Input.</h4>
            <p>
              <form method="post" action="/check.php">
                  <input type="text" placeholder="input1" name="input1">
                  <input type="text" placeholder="input2" name="input2">
                  <input type="submit" value="제출">
              </form>
            </p>
          </div>
    
        <?php
            require_once('flag.php');
            error_reporting(0);
        ?> 
        </div> 
    </body>
    </html>
```
    
- flag.php
    
```php
    <?php
        $flag = 'flag{**Sample**}'
    ?>
```
    

<img width="840" height="358" alt="image (16)" src="https://github.com/user-attachments/assets/0dfe127f-fbc9-49a1-8913-4c67391dacb1" />


문제 화면은 이렇다.

일단 코드를 살펴보면

input1은

```php
strlen($input_1) < 4
$input_1 < "8" && $input_1 < "7.A" && $input_1 > "7.9"
```

7.A는 문자로 끝나서 숫자 문자열이 아니다. PHP는 문자열 vs 문자열 비교에서 둘 다 숫자 문자열일 때만 숫자로 비교하고, 아니면 byte 단위 사전식 비교를 한다.

$input_1을 숫자로 인식되지 않는 문자열로 만들면, 세 비교 전부 문자열 비교가 된다.

7.9를 아스키로 표현하면 `0x37 0x2E 0x39`, 7.A를 아스키로 표현하면 `0x37 0x2E 0x41`

세 번째 글자가 `0x39`보다 크고, `0x41`보다 작으면 조건을 만족시킬 수 있다.

그 범위는 `: ; < = > ? @` (0x3A ~ 0x40)

→ `input1 = “7 :”` (길이 3, `strlen<4` 통과)

input2는

```php
strlen($input_2) < 3 && strlen($input_2) > 1   // 길이 정확히 2
$input_2 < 74 && $input_2 > "74"
```

- `$input_2 < 74` → 상대가 **정수 74** → PHP7 구식 규칙상 문자열을 숫자로 강제 변환(선행 숫자 추출, 숫자로 시작 안 하면 0)해서 비교
- `$input_2 > "74"` → 상대가 **문자열 "74"** → 문자열 vs 문자열 비교. `input_2`가 숫자 문자열이 아니면 사전식 비교

즉 같은 변수인데 비교 상대의 타입(int vs string) 때문에 서로 다른 비교 방식이 적용됩니다.

`input_2`를 숫자로 시작하지 않는 문자열(예: 알파벳)로 만들면:

- `$input_2 < 74`: 숫자 추출 실패 → 0으로 캐스팅 → `0 < 74` → true
- `$input_2 > "74"`: 둘 다 숫자문자열 아니므로 사전식 비교, 알파벳(`A`=0x41)이 숫자(`7`=0x37)보다 크므로 `"AA" > "74"` → true

→ `input2 = "AA"` (길이 2, 조건 통과)

각 input을 집어넣으면 

<img width="967" height="306" alt="image (17)" src="https://github.com/user-attachments/assets/542dd838-3c24-46c6-b3e5-d8061af99fd4" />


flag 발견입니당
