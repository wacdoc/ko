# 나만의 SMTP 메일 발송 서버 구축

## 전문

SMTP는 다음과 같은 클라우드 공급업체로부터 서비스를 직접 구매할 수 있습니다.

* [아마존 SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [알리 클라우드 이메일 푸시](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

무제한 전송, 낮은 전체 비용 등 자체 메일 서버를 구축할 수도 있습니다.

아래에서는 자체 메일 서버를 구축하는 방법을 단계별로 보여줍니다.

## 서버 선택

자체 호스팅 SMTP 서버에는 포트 25, 456 및 587이 열려 있는 공용 IP가 필요합니다.

일반적으로 사용되는 퍼블릭 클라우드는 이러한 포트를 기본적으로 차단했으며 작업 지시서를 발행하여 열 수는 있지만 결국 매우 번거롭습니다.

이러한 포트가 열려 있고 역방향 도메인 이름 설정을 지원하는 호스트에서 구입하는 것이 좋습니다.

여기서 [콘타보를](https://contabo.com) 추천합니다.

Contabo는 2003년에 설립되어 매우 경쟁력 있는 가격으로 독일 뮌헨에 기반을 둔 호스팅 제공업체입니다.

구매 통화를 유로로 선택하면 가격이 더 저렴해집니다(메모리 8GB, CPU 4개 서버는 연간 약 529위안, 초기 설치비는 1년간 무료).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

주문할 때 `prefer AMD` 표시하면 AMD CPU가 장착된 서버의 성능이 더 좋아집니다.

다음에서는 Contabo의 VPS를 예로 들어 자신의 메일 서버를 구축하는 방법을 설명합니다.

## 우분투 시스템 구성

여기서 운영 체제는 Ubuntu 22.04입니다.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

ssh의 서버에 `Welcome to TinyCore 13!` 표시되면(아래 그림 참조) 시스템이 아직 설치되지 않았음을 의미하므로 ssh 연결을 끊고 몇 분 정도 기다린 후 다시 로그인하십시오.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

`Welcome to Ubuntu 22.04.1 LTS` 나타나면 초기화가 완료된 것이며 다음 단계를 계속 진행할 수 있습니다.

### [선택] 개발 환경 초기화

이 단계는 선택 사항입니다.

편의를 위해 우분투 소프트웨어의 설치 및 시스템 구성을 [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) 에 넣었습니다.

한 번의 클릭으로 설치하려면 다음 명령을 실행하십시오.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

중국 사용자는 다음 명령어를 대신 사용하시면 언어, 시간대 등이 자동으로 설정됩니다.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo는 IPV6를 지원합니다.

SMTP가 IPV6 주소로 이메일을 보낼 수도 있도록 IPV6를 활성화합니다.

`/etc/sysctl.conf` 편집

다음 줄을 수정하거나 추가하십시오.

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

[contabo 튜토리얼 후속 조치: 서버에 IPv6 연결 추가](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

`/etc/netplan/01-netcfg.yaml` 편집하고 아래 그림과 같이 몇 줄을 추가합니다(Contabo VPS 기본 구성 파일에는 이미 이러한 줄이 있으므로 주석을 제거하십시오).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

그런 다음 `netplan apply` 수정된 구성을 적용합니다.

구성이 성공하면 `curl 6.ipw.cn` 사용하여 외부 네트워크의 ipv6 주소를 볼 수 있습니다.

## 구성 저장소 작업 복제

```
git clone https://github.com/wactax/ops.soft.git
```

## 도메인 이름에 대한 무료 SSL 인증서 생성

메일을 보내려면 암호화 및 서명을 위한 SSL 인증서가 필요합니다.

우리는 [acme.sh를](https://github.com/acmesh-official/acme.sh) 사용하여 인증서를 생성합니다.

acme.sh는 오픈 소스 자동 인증서 서명 도구입니다.

구성 웨어하우스 ops.soft를 입력하고 `./ssl.sh` 실행하면 **상위 디렉토리** 에 `conf` 폴더가 생성됩니다.

[acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) 에서 DNS 공급자를 찾고 `conf/conf.sh` 편집하십시오.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

그런 다음 `./ssl.sh 123.com` 실행하여 도메인 이름에 대한 `123.com` 및 `*.123.com` 인증서를 생성합니다.

첫 번째 실행은 [acme.sh를](https://github.com/acmesh-official/acme.sh) 자동으로 설치하고 자동 갱신을 위해 예약된 작업을 추가합니다. `crontab -l` 볼 수 있습니다. 다음과 같은 줄이 있습니다.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

생성된 인증서의 경로는 `/mnt/www/.acme.sh/123.com_ecc。`

인증서 갱신은 `conf/reload/123.com.sh` 스크립트를 호출하고 이 스크립트를 편집하며 `nginx -s reload` 와 같은 명령을 추가하여 관련 응용 프로그램의 인증서 캐시를 새로 고칠 수 있습니다.

## chasquid로 SMTP 서버 구축

[chasquid](https://github.com/albertito/chasquid) 는 Go 언어로 작성된 오픈 소스 SMTP 서버입니다.

Postfix, Sendmail과 같은 고대 메일 서버 프로그램을 대체하기 위해 chasquid는 더 간단하고 사용하기 쉬우며 2차 개발도 더 쉽습니다.

`./chasquid/init.sh 123.com` 자동으로 설치됩니다(123.com을 보내는 도메인 이름으로 대체).

## 이메일 서명 DKIM 구성

DKIM은 편지가 스팸으로 취급되는 것을 방지하기 위해 이메일 서명을 보내는 데 사용됩니다.

명령이 성공적으로 실행되면 DKIM 레코드를 설정하라는 메시지가 표시됩니다(아래 참조).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

DNS에 TXT 레코드를 추가하기만 하면 됩니다(아래 참조).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## 서비스 상태 및 로그 보기

 `systemctl status chasquid` 서비스 상태를 봅니다.

정상 작동 상태는 아래 그림과 같습니다.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` 또는 `journalctl -xeu chasquid` 오류 로그를 볼 수 있습니다.

## 역방향 도메인 이름 구성

역방향 도메인 이름은 IP 주소를 해당 도메인 이름으로 해석할 수 있도록 하기 위한 것입니다.

역방향 도메인 이름을 설정하면 이메일이 스팸으로 식별되는 것을 방지할 수 있습니다.

메일이 수신되면 수신 서버는 발신 서버의 IP 주소에 대해 역 도메인 이름 분석을 수행하여 발신 서버에 유효한 역 도메인 이름이 있는지 확인합니다.

발신 서버에 역 도메인 이름이 없거나 역 도메인 이름이 발신 서버의 IP 주소와 일치하지 않는 경우 수신 서버에서 이메일을 스팸으로 인식하거나 거부할 수 있습니다.

[https://my.contabo.com/rdns를](https://my.contabo.com/rdns) 방문하여 아래와 같이 구성합니다.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

역방향 도메인 이름을 설정한 후 서버에 대한 도메인 이름 ipv4 및 ipv6의 정방향 확인을 구성해야 합니다.

## chasquid.conf의 호스트 이름 편집

`conf/chasquid/chasquid.conf` 역방향 도메인 이름 값으로 수정합니다.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

그런 다음 `systemctl restart chasquid` 실행하여 서비스를 다시 시작하십시오.

## git 저장소에 백업 conf

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

예를 들어 다음과 같이 conf 폴더를 내 github 프로세스에 백업합니다.

개인 창고 먼저 만들기

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

conf 디렉토리를 입력하고 창고에 제출

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## 발신자 추가

달리다

```
chasquid-util user-add i@wac.tax
```

발신자 추가 가능

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### 암호가 올바르게 설정되었는지 확인

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

사용자를 추가하면 `chasquid/domains/wac.tax/users` 업데이트되므로 창고에 제출하는 것을 잊지 마십시오.

## DNS 추가 SPF 레코드

SPF(Sender Policy Framework)는 이메일 사기를 방지하는 데 사용되는 이메일 확인 기술입니다.

발신자의 IP 주소가 주장하는 도메인 이름의 DNS 레코드와 일치하는지 확인하여 사기꾼이 가짜 이메일을 보내는 것을 방지하여 메일 발신자의 신원을 확인합니다.

SPF 레코드를 추가하면 이메일이 스팸으로 식별되는 것을 최대한 방지할 수 있습니다.

도메인 이름 서버가 SPF 유형을 지원하지 않는 경우 TXT 유형 레코드를 추가하기만 하면 됩니다.

예를 들어 `wac.tax` 의 SPF는 다음과 같습니다.

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

`_spf.wac.tax` 에 대한 SPF

`v=spf1 a:smtp.wac.tax ~all`

여기에는 `include:_spf.google.com` 있습니다. 나중에 Google 편지함의 발신 주소로 `i@wac.tax` 구성하기 때문입니다.

## DNS 구성 DMARC

DMARC는 (Domain-based Message Authentication, Reporting & Conformance)의 약자입니다.

SPF 바운스를 캡처하는 데 사용됩니다(구성 오류로 인해 발생하거나 다른 사람이 귀하를 사칭하여 스팸을 보내는 것일 수 있음).

TXT 레코드 `_dmarc` 추가,

내용은 다음과 같습니다

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

각 파라미터의 의미는 다음과 같습니다.

### 피(정책)

SPF(Sender Policy Framework) 또는 DKIM(DomainKeys Identified Mail) 확인에 실패한 이메일을 처리하는 방법을 나타냅니다. p 매개변수는 다음 세 값 중 하나로 설정할 수 있습니다.

* 없음: 아무런 조치도 취하지 않고 확인 결과만 이메일 보고 메커니즘을 통해 발신자에게 피드백됩니다.
* 검역소: 확인을 통과하지 못한 메일을 스팸 폴더에 넣지만 메일을 직접 거부하지는 않습니다.
* 거부: 확인에 실패한 이메일을 직접 거부합니다.

### fo(실패 옵션)

보고 메커니즘에서 반환되는 정보의 양을 지정합니다. 다음 값 중 하나로 설정할 수 있습니다.

* 0: 모든 메시지에 대한 유효성 검사 결과 보고
* 1: 인증에 실패한 메시지만 보고
* d: 도메인 이름 확인 실패만 보고
* s: SPF 확인 실패만 보고
* l: DKIM 확인 실패만 보고

### 루아앤루프

* rua(집계 보고서용 보고 URI) : 집계 보고서를 수신하기 위한 이메일 주소
* ruf(포렌식 보고서용 보고 URI): 자세한 보고서를 받을 이메일 주소

## MX 레코드를 추가하여 이메일을 Google Mail로 전달

범용 주소(Catch-All, 프리픽스 제한 없이 이 도메인 이름으로 전송된 모든 이메일 수신 가능)를 지원하는 무료 회사 사서함을 찾을 수 없었기 때문에 chasquid를 사용하여 모든 이메일을 내 Gmail 사서함으로 전달했습니다.

**유료 비즈니스 사서함이 있는 경우 MX를 수정하지 말고 이 단계를 건너뛰십시오.**

`conf/chasquid/domains/wac.tax/aliases` 편집, 전달 사서함 설정

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` 모든 이메일을 나타내며, `i` 는 위에서 생성한 발신 사용자의 이메일 주소 프리픽스입니다. 메일을 전달하려면 각 사용자가 회선을 추가해야 합니다.

그런 다음 MX 레코드를 추가합니다(아래 그림의 첫 번째 줄에 표시된 대로 여기에서 역방향 도메인 이름의 주소를 직접 가리킴).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

구성이 완료되면 다른 이메일 주소를 사용하여 `i@wac.tax` 및 `any123@wac.tax` 로 이메일을 보내 Gmail에서 이메일을 받을 수 있는지 확인할 수 있습니다.

그렇지 않은 경우 chasquid 로그( `grep chasquid /var/log/syslog` )를 확인하십시오.

## Google Mail로 i@wac.tax로 이메일 보내기

Google Mail이 메일을 받은 후 자연스럽게 i.wac.tax@gmail.com 대신 `i@wac.tax` 로 답장하기를 바랐습니다.

[https://mail.google.com/mail/u/1/#settings/accounts를](https://mail.google.com/mail/u/1/#settings/accounts) 방문하여 "다른 이메일 주소 추가"를 클릭합니다.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

그런 다음 전달된 이메일로 받은 인증 코드를 입력하십시오.

마지막으로 기본 발신자 주소로 설정할 수 있습니다(동일한 주소로 회신하는 옵션과 함께).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

이렇게 SMTP 메일 서버 구축을 완료함과 동시에 Google Mail을 이용하여 메일을 주고 받습니다.

## 구성이 성공했는지 확인하기 위해 테스트 이메일을 보냅니다.

`ops/chasquid` 입력

종속성을 설치하려면 `direnv allow` 실행하십시오(direnv는 이전 단일 키 초기화 프로세스에 설치되었으며 후크가 셸에 추가됨).

그런 다음 실행

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

매개변수의 의미는 다음과 같습니다.

* 사용자: SMTP 사용자 이름
* 패스: SMTP 비밀번호
* 받는 사람: 받는 사람

테스트 이메일을 보낼 수 있습니다.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

구성이 성공적인지 확인하기 위해 Gmail을 사용하여 테스트 이메일을 받는 것이 좋습니다.

### TLS 표준 암호화

아래 그림과 같이 SSL 인증서가 성공적으로 활성화되었음을 의미하는 작은 자물쇠가 있습니다.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

그런 다음 "원본 이메일 표시"를 클릭하십시오.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### 디킴

아래 그림과 같이 Gmail 원본 메일 페이지에 DKIM이 표시되며 이는 DKIM 구성이 성공했음을 의미합니다.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

원본 이메일의 헤더에 수신됨을 확인하면 발신자 주소가 IPV6임을 알 수 있으며 이는 IPV6도 성공적으로 구성되었음을 의미합니다.
