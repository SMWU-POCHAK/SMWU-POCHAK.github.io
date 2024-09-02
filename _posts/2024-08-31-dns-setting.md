---
title: GCP Cloud DNS를 통한 도메인 설정과 SSL 인증서 설치
date: 2024-08-31 17:00:00 +0900
categories: [Infra, DNS]
tags: [backend]     # TAG names should always be lowercase
author: lucy-oh
---

# 새로 도메인을 설정한 이유
POCHAK의 기존 도메인이 만료됨에 따라 기존 도메인 연장 혹은 새로운 도메인으로의 변경이 필요한 상태였습니다. <br>
팀원들과의 논의 결과, 무엇보다도 기존 도메인의 연장비용이.. 굉장히 부담스러웠기에 비교적으로 저렴한 `pochak.app` 이라는 새로운 도메인을 구매하게 됩니다.

> 창업지원단의 지원금을 받고 있기에 대부분의 서비스를 AWS에서 GCP로 옮기고 있었고, 따라서 AWS의 Route53이 아닌 GCP의 Cloud DNS라는 서비스를 통해 도메인을 설정하였습니다.
{: .prompt-info }

# 환경

## 호스트 환경
- GCP
- Ubuntu 20.04.6 LTS
- nginx/1.18.0 (Ubuntu)

# 도메인 설정 with GCP
> 가비아 구매 과정은 생략하겠습니다!

## Cloud DNS 영역 만들기
![img](/assets/img/2024-08-31-dns-setting/1-img.png)
_사진에서는 pochak--zone으로 캡처하였지만, 실제로는 pochak-zone으로 설정함_

다음과 같이 영역 이름을 `pochak-zone`, DNS 이름을 `pochak.app`으로 설정해주었습니다.

## 새로운 레코드 추가하기
![img](/assets/img/2024-08-31-dns-setting/2-img.png)

`표준 추가` 버튼을 눌러 새로운 레코드를 추가해줍니다.

<br>

![img](/assets/img/2024-08-31-dns-setting/3-img.png)

위와 같이 `A` 유형, 데이터엔 연결될 서버의 `IP주소`를 넣어줍니다.

## 가비아에서 GCP 네임 서버 설정

이제 다음과 같이 도메인 구매처의 관리 페이지에 들어가, GCP 네임 서버를 설정해주면 됩니다.

![img](/assets/img/2024-08-31-dns-setting/4-img.png)

네임 서버 정보는 콘솔에서 `NS` 레코드에서 확인할 수 있습니다.

![img](/assets/img/2024-08-31-dns-setting/5-img.png)

# SSL 인증서 설치
> .app 도메인은 SSL 인증서 설치가 필수입니다!
{: .prompt-info }

## certbot 설치

```bash
$ sudo apt install certbot
$ sudo apt install python3-certbot-nginx
$ sudo certbot --nginx
```

여기서 `sudo certbot --nginx` 명령어는 certbot이 nginx 설정을 자동으로 조정하게 하는 명령어입니다. 

![img](/assets/img/2024-08-31-dns-setting/6-img.png)

1. 먼저 도메인 `pochak.app`을 입력하였고,
2. 2번 `Redirect`로 설정을 해주었습니다.

<br>

`/etc/nginx/sites-enabled/default` 파일을 확인해보면
certbot이 다음과 같이 자동으로 세팅을 해둔 것을 확인할 수 있습니다.

![img](/assets/img/2024-08-31-dns-setting/7-img.png)

## Ubuntu 방화벽 확인

- 80 (http)
- 443 (https)
- 22 (ssh)

포트가 모두 열려있는지 확인합니다.

```bash
$ sudo ufw status
```

![img](/assets/img/2024-08-31-dns-setting/8-img.png)

## SSL 자동 갱신 확인

```bash
$ sudo certbot renew --dry-run
```

위 명령어를 통해 SSL 인증서 갱신 시뮬레이션을 실행할 수 있습니다.

![img](/assets/img/2024-08-31-dns-setting/9-img.png)

위와 같이 `Congratulations ~~` 이 뜬다면 성공!

## 사이트 보안 등급 확인
https://www.ssllabs.com/ssltest/analyze.html

위 사이트로 접속하여 도메인을 입력하면 사이트 보안 등급을 확인할 수 있습니다.

![img](/assets/img/2024-08-31-dns-setting/10-img.png)


## 설정 완료

https://pochak.app 에 정상적으로 서버로 접속할 수 있는 걸 확인할 수 있습니다!

![img](/assets/img/2024-08-31-dns-setting/11-img.png)

# 트러블슈팅

## 자동 갱신 과정에서 발생한 에러

```bash
$ sudo certbot certonly -d "*.pochak.app" --manual --preferred-challenges dns
```

> 위와 같이 --manual 을 사용하여 SSL 인증서를 발급받았다면, <br>
> 중간에 TXT 레코드 편집이 필요합니다.<br>
> 하지만, certbot이 레코드 편집까지 자동으로 수행해주는 것은 불가능하기 때문에, 에러가 발생하는 것입니다. <br>
> [참고: GitHub Issue](https://github.com/certbot/certbot/issues/6280#issuecomment-451361955)
{: .prompt-warning }

위 --manual 사용은 *(와일드 카드) 사용에 유리한 면이 있지만, 아직 포착은 도메인을 다양하게 사용하고 있지 않기에 위와 같은 방법을 사용하였습니다.