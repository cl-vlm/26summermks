## 익스플로잇 코드 탐색 및 활용

### CVE-2017-1000486

- PBE 암호화
    - 패스워드 기반 암호
    - 패스워드를 기초로 해서 만든 키로 암호화를 수행하는 방법
- JSF
    - javaServer faces
    - 자바 기반의 웹 애플리케이션의 사용자 인터페이스(UI)를 개발하기 위한 컴포넌트 기반 프레임워크
    - 요즘은 React, Vue.js와 같은 다른 프론트엔드 기술이 인기를 얻고있어 사용이 다소 줄어듦
- el 표현식
    - jsp 2.0의 새롭게 추가된 스크립트 언어
    - $(…)와 같은 표현식으로 표현
    - jsp 스크립트의 표현식을 대신해 속성 값을 쉽게 출력하도록 고안된 언어

---

## 최근 CVE 취약점 분석과 이해

### CVE-2023-23524

→ 시크릿 키가 디폴트 값이어서 RCE까지 가능한 내용

이를 이해하기 위한 세 가지 지식

1. Flask 서버가 쿠키를 어떻게 만드는지
2. Flask-Unsign에 대한 지식
3. PostgreSQL에서 어떤 정상적인 기능이 RCE를 가능하게 하는지

**실습 환경 구축**

```python
$ git clone https://github.com/apache/superset.git

$ cd superset
$ sed -i 's/lastest-dev/1.4.0/g' docker-compose-non-dev.yml

$ docker-compose -f docker-compose-non-dev.yml pull
$ docker-compose -f docker-compose-non-dev.yml up
```

### Super FabriXss

공격방법

1. 공격자가 iframe 태그로 구성된 reflected XSS 페이로드가 담긴 URL을 Azure Service Family 사용자에게 전달
2. 사용자가 악성 URL을 클릭하여 XSS 공격에 노출
3. XSS 페이로드인 iframe 태그의 src 속성에 정의된 공격자 서버의 주소에 접근에 Fetch.html 파일을 내기 (Fetch.html 파일 내용에는 악성 스크립트 태그 내용(CSRF)이 담김)
4. 공격자는 docker file을 작성하고 빌드해 docker 부분의 public 이미지로 배포
5. 공격자가 작성한 이 임의의 docker 파일에는 최종적으로 리버스 시야를 여는 악성 페이로드가 담겨있다.

→ 단순한 XSS 공격을 시작으로 인스턴스를 업그레이드하는 CSRF 공격과 연계하여 사용된 인스턴스에 원격 능력 코드를 실행할 수 있는 시나리오가 만들어진다는 것을 알 수 있다.

### 버그헌팅 주의사항

실제로 서비스에 피해를 주는 공격을 하는 행위는 지양해야함

---

## CVE 취약점 버그헌팅 사례

### hackerone - CVE-2017-1000486

→ Metasploit을 이용해 알아본 prime faces의 framework의 원격 코드 실행이 가능한 취약점

### hackerone - CVE-2022-22954

→ VMWare에 Workspace One Access의 Identity Access Major, IAM 플랫폼을 대상으로 원격 코드 실행이 가능한 취약점   

#### ReadOnePNGImage 함수를 통해 PNG 이미지에 text chunk 정보를 읽어내는 과정

- 이미지를 업로드했고, 서버에서 내부적으로 convert 명령어가 실행되었다는 가정
- ReadOnePNGImage 함수(coders/png.c:2164)가 실행됨
- 아래 코드는 ReadOnePNGImage 함수의 일부로 PNG 파일의 tEXt chunk를 읽는 동작을 수행함

#### text chunk 정보 활용 과정

- SetImageProperty 함수(MagickCore/property.c:4360)가 실행됨
- 키워드 중에 “profile”이라는 문자열이 있는지 확인함
- profile에 정의된 문자열을 filename으로 복사함 (4720-4722번째 줄)

#### SetImageProperty 함수를 통한 file name변수에 새로운 값을 할당 → 새로운 값이 뭐고, 어떻게 활용?

- FileToStringInfo 함수는 filename으로 들어온 파일의 내용을 읽고 반환함
- FileToBlob함수는 내부적으로 open 함수로 파일의 내용을 읽음

#### 읽어낸 파일의 내용 활용 과정

- FileToStringInfo 함수의 반환 값이 profile 변수에 할당됨
- SetImageProfile 함수가 호출되고 파일내용이 새롭게 생성될 이미지 파일에 쓰여짐
- 이제 새로 생성된 이미지를 다운로드하여 페이로드를 확인하면됨

#### 이 정보가 어떤 것이고, 어떻게 확인하는지

- 만약 처음에 사용자가 본인의 프로필 사진을 업로드했을 때 profile에 /etc/passwd 문자열을 PNG의 profile에 넣고 업로드했다고 가정

#### 이 정보를 어떻게 확인하는지 그리고 그 결과는 어떤 것인지

- 서버에서 변환된 파일을 다운로드 받고, ImageMagick의 identify 명령을 통해 Raw profile type 키 값을 확인

#### 16진수 값 해석

- Raw Profile Type에 있던 16진수 전부 복사
- 개행 제거 후 16진수를 ASCII값으로 변환함
- /etc/passwd 파일 내용 확인 가능
