# 질문: 하나의 nginx 서버 설정에 여러 개의 domain 을 연결해서 접속할 때 발생하는 redirection 오류

도메인정보:  
`web.mycom.co.kr: 10.10.10.1`  
`web-int.mycom.co.kr: 192.168.10.1`

springboot 기반 웹서비스(192.168.10.1:8080)를 nginx(192.168.10.1:443) 를 통해서 접속할 수 있습니다.

외부망 공유기(10.10.10.1)는 DHCP 서비스(192.168.10/24)를 제공합니다.  
공유기는 외부포트(10072)를  내부서버 192.168.10.1:443 에 포워딩을 합니다.

그래서 10.10.10/24 망의 서버(외부망 서버)에서 `https://web.mycom.co.kr:10072`를 통해서 웹서비스를 접속합니다.

nginx는  연결정보 관련해서 아래와 같은 설정을 합니다.
```
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port 10072;
    # '라우팅'용 prefix 를 연결된 서비스로 전달
    proxy_set_header X-Client-Port $remote_port;
```

그리고 192.168.10/24 망의 서버(내부망 서버)에서는 `https://web-int.mycom.co.kr` 을 통해서 웹서비스에 접속합니다.

그런데 내부망 서버에서 로그인에 실패했을 때, 웹서비스에서 리다이렉션 URL로 `https://web-int.mycom.co.kr:10072`를 반환합니다.


# 답변: 전형적인 리버스 프록시(Reverse Proxy) 환경에서의 '헤더 오염(Header Contamination)' 및 Spring Boot의 컨텍스트 해석 차이로 인해 발생

이 문제는 전형적인 **리버스 프록시(Reverse Proxy) 환경에서의 '헤더 오염(Header Contamination)' 및 Spring Boot의 컨텍스트 해석 차이**로 인해 발생합니다.

---

### 1. 원인 분석 (Root Cause Analysis)

문제의 핵심은 Nginx 설정 중 아래의 하드코딩된 항목에 있습니다.

```nginx
    proxy_set_header X-Forwarded-Port 10072;

```

외부망 사용자(`web.mycom.co.kr:10072`)와 내부망 사용자(`web-int.mycom.co.kr:443`)의 트래픽은 라우팅 계층을 거쳐 최종적으로 Nginx가 위치한 `192.168.10.1:443`으로 인입됩니다.

이때 Nginx가 모든 요청에 대해 무조건 `X-Forwarded-Port`를 `10072`로 강제 설정하여 Spring Boot(192.168.10.1:8080)로 전달하고 있습니다. Spring Boot 4.0.3은 인증 실패 시 리다이렉션 URL을 생성하기 위해 `X-Forwarded-*` 헤더를 참조(ForwardedHeaderFilter 등)하는데, 내부망에서 들어온 요청(`Host: web-int.mycom.co.kr`)임에도 불구하고 포트가 `10072`로 명시되어 있으므로 조합 결과가 `[https://web-int.mycom.co.kr:10072](https://web-int.mycom.co.kr:10072)`로 도출되는 것입니다.

---

### 2. 해결책 (Solutions)

이 문제를 해결하려면 Nginx가 클라이언트가 요청한 **도메인(Host)을 기준으로 포트를 동적으로 분기**하여 Spring Boot로 전달해야 합니다. 아키텍처 표준에 따라 2가지 방식을 제안합니다.

#### 해결책 A: Nginx `server` 블록 분리 (권장)

가장 안정적이고 명시적인 엔터프라이즈 표준 방식입니다. 도메인별로 `server` 블록을 나누어 각각에 맞는 포트를 헤더에 주입합니다.

```nginx
# 외부망(10.10.10.1:10072 &rarr; 192.168.10.1:443)을 통한 접근
server {
    listen 443 ssl;
    server_name web.mycom.co.kr;
    
    # ... ssl 설정 등 ...

    location / {
        proxy_pass http://192.168.10.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        
        # 외부망 접속이므로 라우터의 포트 주입
        proxy_set_header X-Forwarded-Port 10072; 
        proxy_set_header X-Client-Port $remote_port;
    }
}

# 내부망(192.168.10.1:443)을 통한 직접 접근
server {
    listen 443 ssl;
    server_name web-int.mycom.co.kr;
    
    # ... ssl 설정 등 ...

    location / {
        proxy_pass http://192.168.10.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        
        # 내부망 접속이므로 기본 HTTPS 포트 주입 (또는 $server_port 사용)
        proxy_set_header X-Forwarded-Port 443; 
        proxy_set_header X-Client-Port $remote_port;
    }
}

```

#### 해결책 B: Nginx `map` 지시어를 활용한 동적 할당

하나의 `server` 블록을 반드시 유지해야 하는 환경이라면, `http` 블록 수준에서 `map`을 사용하여 도메인(`$host`)에 따라 변수를 동적으로 매핑할 수 있습니다.

```nginx
# http 블록에 추가
map $host $proxy_x_forwarded_port {
    "web.mycom.co.kr"     10072;
    "web-int.mycom.co.kr" 443;
    default               443;
}

server {
    listen 443 ssl;
    server_name web.mycom.co.kr web-int.mycom.co.kr;

    location / {
        proxy_pass http://192.168.10.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        
        # map에서 정의한 변수를 주입
        proxy_set_header X-Forwarded-Port $proxy_x_forwarded_port;
        proxy_set_header X-Client-Port $remote_port;
    }
}

```

---

### 3. Spring Boot 추가 검증 포인트

Nginx 설정 변경 후, Spring Boot 4.0.3이 전달받은 프록시 헤더를 신뢰하고 리다이렉션 URL에 반영하도록 애플리케이션 프로퍼티 설정이 올바르게 되어 있는지 확인이 필요합니다. `application.yml` 파일에 다음 설정이 적용되어 있어야 합니다.

```yaml
server:
  forward-headers-strategy: framework # 또는 native

```

Nginx 레벨에서 도메인에 따른 포트 분리만 정확히 이루어지면, 내부망에서의 인증 실패 리다이렉트는 정상적으로 `[https://web-int.mycom.co.kr](https://web-int.mycom.co.kr)` (또는 `[https://web-int.mycom.co.kr:443](https://web-int.mycom.co.kr:443)`)로 라우팅될 것입니다.
