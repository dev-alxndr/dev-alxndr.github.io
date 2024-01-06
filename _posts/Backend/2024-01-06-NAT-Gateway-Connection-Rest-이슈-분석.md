---
title: NAT Gateway Connection Rest 이슈 분석
tags:
  - AWS
  - backend
categories:
  - Backend
date: 2024-01-06 18:07
---
어느날 부터 개발, QA환경  구독결제 스케줄러에서 Connection Reset 에러가 발생했다.
스케줄러는 결제모듈에 HTTP를 통해 결제 요청을 하게 되는데 첫번 째 요청만 실패하고
그 다음 요청은 성공하는 현상을 보이고 있었다.

### 원인을 찾아서..
왜 갑자기 잘 되던 소스가 문제를 일으키는지 찾기 시작했고
인프라팀에서 인프라를 변경한 후 부터 문제가 발생했다는 것을 알았다.
인프라 변경점에 대해서 들었으나 크게 문제 될 건 찾지 못했다.

우선 HttpClient로 사용중인 Apache HttpClient 로그 레벨을 수정해서 확인을 해봤다.
![](/assets/img/d0a4a15500f1e3e5d8608db703c4d246.png)
- 화살표 방향이 시간순이다.

로그에서 알 수 있듯이 풀에서 커넥션을 할당받고 요청을 하려하지만 거의 1ms만에 Read Timeout을 받았다.
하지만 요청을 멈추지않고 시도를 했고 결국에 Connection Reset을 받고 커넥션을 종료시킨다.

현재 RestTemplate으로 요청하고 있으며 이번 이슈를 계기로 자세하게 살펴봤다.
우리가 사용중인 방식은 HTTP/1.1 Keep-alive를 활용해서 같은 호스트에 대해서는 매번 Handshake를 하지 않아도 되는 커넥션 풀을 사용하는 방식이다.

기존 설정되어있던..RestTemplate
```java
@Bean  
public RestTemplate restTemplate() {  
  final RestTemplate restTemplate = new RestTemplate();  
  final HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();  
  factory.setConnectTimeout(1_000);  
  factory.setReadTimeout(1_000 * 10);  
  factory.setConnectionRequestTimeout(1000);  
  
  restTemplate.setRequestFactory(factory);  
  
  return restTemplate;  
}
```
> HttpComponentsClientHttpRequestFactory에 따로 지정해주지 않으면 기본적으로 PoolingHttpClientConnectionManager를 사용하게 되고 keep-alive에 따라 pooling 한다.
> 또한, 기본적인 ReuseStrategy가 있는데 Keep-alive 시간을 서버로 부터 받으면 그 시간으로 timeout이 지정되지만 없는 경우 무제한이다.

##### Keep-alive가 문제인가..
이때까지만 해도 실패가 되는 규칙을 찾지 못했고 불확실하게 코드가 동작하는게 아니라 어떤 조건인 경우 안되기 때문에 그 조건을 찾기 위해 ....
결제모듈 Keep-alive를 조절한다..

결제모듈로 부터 keep-alive 값이 안오고 있었다. 

그렇다고 해도 커넥션은 계속 열려있는데 왜 안되는 것인가??
계속해서 첫번째 요청만 안됐다.

##### ALB IdleTimeout이 문제인가..
ALB 설정에는 IdleTimeout을 설정할 수 있다. 지정된 시간동안 요청이 없는 경우 커넥션을 닫게 된다.
인프라에서는 1800초(30분)을 설정해뒀다.
그리고 스케줄러는 테스트용으로 20분마다 한번씩 돌게 해뒀고 20분에 총 10건의 HTTP 요청을 하게 했다.
![](/assets/img/43552b03b3cdad3419d46f20de6faf49.png)
테스트를 위해서 20분 간격으로 10번을 요청하도록 했지만
ALB에는 9번만 요청이 들어왔다.

이 테스트도 실패했다.
같은 이슈로 에러가 발생했고 더 깊게 파보기 시작한다.

##### ALB Idle Timeout 의심
ALB IdleTimeout을 줄여봤다. 1800 -> 180초
의도한 결과는 아래와 같다.
1. 결제를 요청한다.
2. 요청 작업 이후 180초가 지나면 ALB에서 FIN 패킷이 와서 연결이 종료될 것이다.
3. 다음 스케줄 타임에 새로운 커넥션을 맺고 요청할 것 이다.

확인해본 결과 정확히 예측처럼 동작했다.
FIN 패킷이 ALB로 부터 왔고 새로운 커넥션을 맺어 요청했다.
![](/assets/img/4bd80df886236715ceeff9e89b1efd48.png)
그럼 ALB는 잘못이 없는건가..

#### 계속된 테스트..

다시 1800초로 바꾸고 확인을 해봤다.
##### Tcpdump로 확인한다.
스케줄러 서버에 tcpdump를 통해서 ALB와 TCP 주고 받는 내역을 확인하기 시작했다.
동시에 netstat으로 현재 연결 상태를 확인했는데
요청 직전까지 `ESTABLISHED`로 정상적인 상태였기 때문에 PUSH 패킷으로 데이터를 보냈지만 RST 패킷을 받고 새롭게 커넥션이 열리고 있었다.
![](/assets/img/0f5e3da865e0a995c11907857a55890b.png)

음... 이해할 수 없었다.

또한 ALB 로그도 확인을 했다. Reset된 요청은 ALB에도 로그가 없었다.
ALB까지 닿지도 못한 것이였다.

어느 수준까지 되는지 확인을 해봤다.
300초도 FIN패킷이 왔고 제대로 동작했다.

#### 실마리를 찾았다.
그러다 어떤 글을 봤다.
[Connection reset by peer with AWS NAT Gateway and Keep Alive · Issue #3808 · psf/requests · GitHub](https://github.com/psf/requests/issues/3808)
캡처된 이미지에 NAT Gateway가 Timeout되었을 때 행동이 적혀있었다.

![](/assets/img/476e43d331f30e8869ee3113239bc00d.png)
[NAT 게이트웨이 및 NAT 인스턴스 비교 - Amazon Virtual Private Cloud](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/vpc-nat-comparison.html)

뒤 리소스로 RST (Reset 패킷)을 반환한다고 한다. 그리고 결정적으로 FIN 패킷을 보내지 않는다고 되어있다. 
현재 내가 처한 상황과 정확히 똑같다.

ESTABLISHED 상태일 때 요청을 했지만 RST 패킷을 받고 연결이 종료되는 현상.
그러면 NAT의 제한시간은 얼마인가?

![](/assets/img/e0085001016ad7f7b6beb141a43f5a1c.png)
[NAT 게이트웨이 문제 해결 - Amazon Virtual Private Cloud](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/nat-gateway-troubleshooting.html#nat-gateway-troubleshooting-timeout)
350초가 최대 연결 시간이다.

350초이기 때문에 180초는 정확하게 FIN을 받았고
350초가 지난 연결에 대해서는 ALB에서 FIN을 줘도 받지 못했던 것이다.

현재 Keep-alive 타임이 무제한으로 되기 때문에 (결제 서버에서 Keep-alive 시간 헤더를 주지 않음)
350초 후에는 NAT Gateway와 연결이 끊어진 것이다.

문제의 모든 것이 설명될 때 도파민이 폭발했다. (끄악)
![](/assets/img/5377ecbd49b68fa8a86b68d10cf9d807.png)

#### 왜 NAT를 용의자에서 뺏는가?
NAT를 의심하려했으나 이미 운영까지 적용되어 있다고 했다.
NAT는 운영까지 되어있으면 운영도 발생해야하는 문제일텐데 발생하지 않아 용의선상에서 제외 되었다.

NAT 생성 옵션을 봤을 때 타임아웃을 지정하는 옵션이 없어서 
타임아웃과는 무관하다고 생각을 했고 엉뚱한 부분을 계속 보고 있던 것이다.
저 문서를 보기 전까지는...

### 개선 방법
- NAT 뒤에 있는 리소스는 최대 Keep-alive 타임을 350초 미만으로 설정해야한다.
	- 기본적으로 시간을 지정하는 것도 좋아보인다.
	- [Apache HttpClient Connection Management | Baeldung](https://www.baeldung.com/httpclient-connection-management#keep-alive)
-  RestTemplate PoolingManager를 쓰지않고 SimpleManager를 사용한다.
	- 매번 새롭게 handshake를 맺는다. (요청하는 서버가 고정이기 때문에 이 방법은 우리에게는 좋지 않아보인다.)
- 또는 퍼블릭 망으로 빼도 된다면 서버를 퍼블릭으로 이동한다.

[Connection Reset](Connection%20Reset.md)