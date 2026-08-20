+++
title = "What is a VPN"
date = 2026-08-20T15:30:00+09:00
draft = true
summary = "VPN이란?"
tags = ["network", "vpn", "wireguard"]
topics = ["Writing", "vpn", "network", "wireguard"]
+++
![피터르 브뤼헐의 《바벨탑》](Pieter_Bruegel_the_Elder_-_The_Tower_of_Babel_(Vienna)_-_Google_Art_Project_-_edited.jpg)

인터넷은 매우 혼잡하다. 전 세계 수많은 컴퓨터를 잇고 있기 때문이다. 이 거대한 구조물에서 나만의 네트워크를 가질 순 없을까. 나의 집 컴퓨터와 내 폰만이라도 연결하는 네트워크를 가질 수는 없을까. 누군가 봐도 그 내용이 뭔지 절대 알 수 없도록, 비밀스런 정보는 나만 접근할 수 있도록. 나만의 공간을 가질 수는 없을까?

가능하다. 이 요구사항을 충족하기 위해 사용하는 기술이 VPN이다. VPN은 공용 네트워크 위에 암호화된 터널을 만들고, 인증된 단말이나 Peer들을 사설망처럼 연결해주는 기술이다.

어떻게 구현할까. 세상엔 이미 수많은 상용 VPN이 존재하며, 이를 공짜로 쉽게 구현할 수 있게 해주는 오픈소스들도 존재한다. WireGuard, OpenVPN등이다. 오늘은 WireGuard를 통해 VPN을 구성하는 방법을 알아보고자 한다.



![WireGuard](wireguard.webp)
빠르고 가볍다. 심지어 안전하기까지해서 차세대 오픈소스 VPN 프로토콜로 이미 널리 알려져있다. 기존 VPN의 복잡성을 줄이고 현대적인 암호화 기술을 사용하기 위해 만들어졌다.
WireGuard를 사용하면, 인터넷 속에서 여러 단말간 비밀 통로를 만들 수 있다.

WireGuard 프로토콜의 대표적인 구현체 WireGuard-go가 존재하는데, 기본적으로 client / server등으로 단말을 구분하지 않는다. 모든 단말은 자기 자신을 Device로 생각하며, 연결할 다른 단말은 Peer라는 개념으로 인식한다. 즉 나라는 Device는 다른 단말들을 Peer로 취급해 연결한다.

# 써보자.
VM을 사용해 다음처럼 실습 환경을 구성했다. 원래는 가볍게 docker network와 container로 실습해보려고 했으나, network 스택을 가지고 있는 커널의 격리까지 생각하면 VM을 사용하는게 더 나았다.
현재 MacOS를 사용중이므로 UTM을 설치 후 격리했다.

## 준비환경
- Ubuntu 26.04 LTS VM 2대
  - VM_WG_CLIENT
    - cpu 2 core
    - memory 2GiB
    - enp0s1: 192.168.64.2/24
  - VM_WG_SERVER
    - cpu 2 core
    - memory 2GiB
    - enp0s1: 192.168.64.4/24

이 두 VM은 기본적으로 UTM의 shared network를 모두 공유한다. 192.168.64.0/24 대역을 사용중이다.
사설대역이긴 하나, 본 실험에서는 192.168.64.0/24 네트워크를 공용 네트워크 역할을 하는 underlay라고 가정하려고 한다. 이 위에 WireGuard 터널을 만들기 때문에 VPN의 동작을 관찰할 수 있다.

## 실습의 요구사항
나에게는 사실 관리중인 http server가 하나 있다. 그런데 이건 아주 작고 너무 소중해서 나만 접속하게 하고 싶다.
그래서 애초에 public network에 노출시키지 않았다. 아주 사적인 private network에만 속하도록 띄워두었다. 이 서버에, 내 기기인 VM_WG_CLIENT만 접근하게 하고 싶다.

## 실습 구성
아주 작고 소중한 http server는 사실 다른 VM위에 띄워두진 않았다. 사실 VM_WG_SERVER 머신에 새로운 network 격리환경인 `internal-resources` network namespace를 만들어뒀고, 처음에는 loopback interface만 가지고 있어서 외부로 트래픽을 보낼 수도, 외부에서 받을 수도 없다.
- namespace: internal-resources
  - interface: lo

이 `internal-resources` namespace와 VM_WG_SERVER는 자기들끼리만 연결된 단 하나의 랜선을 열어뒀다. 그래서 VM_WG_SERVER는 `internal-resources`에 접속할 수 있다. 이건 veth이라는 가상 랜선 연결로 가볍게 연결해두었다.
내 작고 소중한 http 서버는 10.20.0.0/24이라는 나만의 private network에 두었다. 10.20.0.2:8080에 바인딩해두었다.

- namespace: (보통 default namespace는 이름이 없음)
  - interface: veth-host: 10.20.0.1/24

- namespace: internal-resources
  - interface: veth-resource: 10.20.0.2/24

## 실습
VM_WG_CLIENT는 여전히 내 http서버에 접근할 수가 없다. UTM shared network를 통해 VM_WG_SERVER에는 접근할 수 있고, VM_WG_SERVER가 10.20.0.0/24 네트워크에 속해있다고 하더라도, VM_WG_SERVER가 외부 트래픽을 내 사설망에 라우팅해주지 않기 때문이다.

어떻게 해주면 될까?
사실 VM_WG_SERVER의 라우팅 테이블에서 라우트 규칙만 잘 수정해주면 되는 일 아닐까? 사실 맞는 것 같다. 근데 그렇게 간단하지가 않다. 다음과 같은 문제가 존재하기 때문이다.

- 문제
  - 누구의 트래픽을 사설망으로 라우트해줄 것인가?, 동일하게, 누구만 허용할 것인가?
  - 내 사설서버에 들어올 트래픽이 public network를 거쳐올 것인데, 암호화된 내용으로 요청받아야하지 않을까?
    - 아무도 몰랐으면 하니까.. 내 서버에 들어온 query와 응답 보고 서버 동작을 유추하면 어떡하지?? 그럼 절대안된다..

`허용한 사용자`만 `암호화된 채`로 VM_WG_SERVER에 요청을 보내야 한다. 그렇게 들어온 트래픽은 internal http server에 넘겨줘도 될 것 같다. 이 때 WireGuard를 사용하면 문제가 깔끔하게 해결된다.

## WireGuard
WireGuard는 미리 등록한 Peer과만 통신하게 설정할 수 있다. 상대 Peer의 static Public Key는 Peer의 신원을 식별하고, `allowed-ips`는 해당 Peer에게 어떤 IP 대역을 라우팅할지와 어떤 source IP를 허용할지를 결정한다. Endpoint IP는 상대를 인증하는 정보라기보다 암호화된 UDP 패킷을 전달할 위치에 해당한다.
Handshake가 진행되면 각 Peer는 자신의 static key와 새로 생성한 ephemeral key를 사용해 Diffie-Hellman 연산을 수행한다. 그 결과로 세션 키를 만들고, 이 키로 transport data를 암호화한다. 이후 handshake와 rekey가 진행되면서 새로운 세션 키가 만들어지므로, 나중에 static private key가 유출되더라도 과거의 모든 세션을 복호화할 수 없도록 설계되어 있다. 이를 Perfect Forward Secrecy(PFS)라고 한다.
아무튼 짱짱 좋다는 의미다.

WireGuard의 구현체인 WireGuard-go를 사용하여 다음처럼 각 VM에 라우팅 테이블을 수정한다.
- VM_WG_CLIENT
  - wg0: 10.10.0.2/24
  - 10.20.0.0/24 dev wg0
- VM_WG_SERVER
  - wg0: 10.10.0.1/24
  - 10.20.0.0/24 dev veth-host


자! 이렇게 되었으니 VM_WG_CLIENT에서 10.20.0.2:8080에서 호스팅 중인 내 HTTP server에 접근할 수 있게됐다.
한번 `curl http://10.20.0.2:8080`을 날려보면...

```html
<!DOCTYPE HTML>
<html lang="en">
<head>
<meta charset="utf-8">
<style type="text/css">
:root {
color-scheme: light dark;
}
</style>
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href=".font-unix/">.font-unix/</a></li>
<li><a href=".ICE-unix/">.ICE-unix/</a></li>
<li><a href=".X11-unix/">.X11-unix/</a></li>
<li><a href=".XIM-unix/">.XIM-unix/</a></li>
<li><a href="code-80a4d0b0-73b6-4570-a98a-f3f88cb33deb">code-80a4d0b0-73b6-4570-a98a-f3f88cb33deb</a></li>
<li><a href="fresh-handshake.pcap">fresh-handshake.pcap</a></li>
<li><a href="resource-arp.pcap">resource-arp.pcap</a></li>
<li><a href="snap-private-tmp/">snap-private-tmp/</a></li>
<li><a href="systemd-private-ab0e4d1760504c7d8ff478de310a3f0b-chrony.service-qd6Iyx/">systemd-private-ab0e4d1760504c7d8ff478de310a3f0b-chrony.service-qd6Iyx/</a></li>
<li><a href="systemd-private-ab0e4d1760504c7d8ff478de310a3f0b-ModemManager.service-8szTEk/">systemd-private-ab0e4d1760504c7d8ff478de310a3f0b-ModemManager.service-8szTEk/</a></li>
<li><a href="systemd-private-ab0e4d1760504c7d8ff478de310a3f0b-polkit.service-adiRmM/">systemd-private-ab0e4d1760504c7d8ff478de310a3f0b-polkit.service-adiRmM/</a></li>
<li><a href="systemd-private-ab0e4d1760504c7d8ff478de310a3f0b-systemd-logind.service-Ee2g3r/">systemd-private-ab0e4d1760504c7d8ff478de310a3f0b-systemd-logind.service-Ee2g3r/</a></li>
<li><a href="underlay.pcap">underlay.pcap</a></li>
<li><a href="wg0.pcap">wg0.pcap</a></li>
</ul>
<hr>
</body>
</html>
```

내 소중한 html을 잘 가져올 수 있었다.

## 토폴로지
![wireguard 실습 토폴로지](<wireguard 실습 토폴로지.png>)
1번, 2번, 3번 구간에서 packet dump를 떠봤다.

## 1번 구간(wireguard tunnel) packet dump

![1번 구간의 패킷 캡처](<1번 구간 packet dump.png>)

1번 구간은 VM_WG_CLIENT와 VM_WG_SERVER의 `enp0s1` 사이를 지나는 underlay 패킷을 캡처한 것이다. WireGuard는 이 구간에서 UDP를 사용한다. public network 상에서 발견되는 패킷이라 생각해도 무방하다.

캡처에는 `192.168.64.2`와 `192.168.64.4` 사이의 WireGuard 패킷만 보인다. 초반에는 Keepalive와 Transport Data가 오가고, 중간에는 다음과 같은 handshake 흐름도 확인할 수 있다.

```text
192.168.64.2 -> 192.168.64.4  Handshake Initiation
192.168.64.4 -> 192.168.64.2  Handshake Response
```

HTTP 요청을 발생시켰지만, 이 구간에서는 `10.10.0.2`, `10.20.0.2`, TCP 포트 `8080`, HTTP method 같은 내부 정보가 보이지 않는다. 외부에서 관찰할 수 있는 것은 터널링 양 끝단 주소 사이의 UDP 패킷과 패킷 크기, 방향, 시점 정도다. 실제 payload는 WireGuard에 의해 암호화된 상태로 지나간다.

## 2번 구간(wg0) packet dump

![2번 구간의 패킷 캡처](<2번 구간 packet dump.png>)

2번 캡처는 화면의 파일명이 `wg0.pcap`인 것으로 보아 VM_WG_SERVER의 `wg0`에서 캡처한 내부 패킷이다. WireGuard가 UDP payload를 복호화한 뒤 커널 네트워크 스택으로 넘긴 결과를 볼 수 있다.

먼저 `10.10.0.2`에서 `10.20.0.2:8080`으로 TCP 연결을 시작한다.

```text
10.10.0.2 -> 10.20.0.2  SYN
10.20.0.2 -> 10.10.0.2  SYN, ACK
10.10.0.2 -> 10.20.0.2  ACK
```

TCP 연결이 성립한 뒤에는 `GET / HTTP/1.1` 요청이 전달되고, 내부 HTTP 서버가 `HTTP/1.0 200 OK` 응답을 반환한다. 응답 데이터가 여러 TCP segment로 나뉘어 전송되기 때문에 Wireshark에는 TCP 재조립 정보도 표시된다. 마지막의 `FIN, ACK`는 HTTP 요청 처리가 끝난 뒤 TCP 연결을 정리하는 과정이다.

이 구간에서는 1번 구간과 달리 내부 터널 주소와 목적지 주소, TCP 포트, HTTP 요청과 응답을 모두 확인할 수 있다. 따라서 `wg0`은 복호화 이후의 네트워크 상태를 확인하는 관찰 지점이다.

## 3번 구간(veth-resource) packet dump

![3번 구간의 패킷 캡처](<3번 구간 packet dump.png>)

3번 캡처는 화면의 파일명이 `resource-arp.pcap`인 것으로 보아 `internal-resources` namespace 안의 `veth-resource`에서 캡처한 것이다. 따라서 WireGuard 패킷이 아니라 내부 Ethernet 구간의 패킷을 확인할 수 있다.

가장 먼저 `10.20.0.1`인 `veth-host`가 `10.20.0.2`의 MAC 주소를 알아내기 위해 ARP request를 broadcast로 보낸다. `10.20.0.2`는 자신의 MAC 주소를 담은 ARP reply를 반환한다.

```text
Who has 10.20.0.2? Tell 10.20.0.1
10.20.0.2 is at <resource MAC address>
```

ARP가 끝난 뒤에는 `10.10.0.2`와 `10.20.0.2` 사이에 TCP handshake가 이어진다. 이후 HTTP `GET / HTTP/1.1` 요청과 `HTTP/1.0 200 OK` 응답이 같은 veth 구간을 통과한다.

캡처 마지막에는 반대 방향의 ARP도 확인할 수 있다. `10.20.0.2`가 응답 패킷을 보낼 때 다음 홉인 `10.20.0.1`의 MAC 주소를 다시 확인한 것이다. 양쪽 network namespace는 ARP 캐시를 별도로 가지므로, 한쪽의 ARP 캐시를 비웠다고 해서 다른 쪽의 캐시까지 비워지는 것은 아니다.

즉 `veth-resource`에서는 WireGuard의 암호화된 UDP가 보이지 않는다. 이미 복호화되어 전달된 내부 IP 패킷과, 그 패킷을 Ethernet으로 전달하기 위한 ARP 및 MAC 주소를 볼 수 있다.

## 구간별 packet dump를 떠봄으로써 알 수 있는것

세 캡처를 비교하면 WireGuard가 패킷을 어떤 경계에서 암호화하고 복호화하는지 확인할 수 있다.

| 캡처 지점 | 확인할 수 있는 정보 |
| --- | --- |
| `enp0s1` | underlay IP, UDP `51820`, handshake, keepalive, 암호화된 transport data |
| `wg0` | 복호화된 `10.10.0.2 ↔ 10.20.0.2` 패킷, TCP, HTTP |
| `veth-resource` | ARP, Ethernet MAC, TCP, HTTP |

따라서 연결 문제가 발생했을 때는 특정 로그 하나만 보는 것이 아니라, 문제가 어느 구간에서 끊겼는지부터 나눠서 확인해야 한다. `enp0s1`에서 handshake가 보이지 않으면 WireGuard peer나 underlay를 확인해야 하고, handshake는 보이지만 `wg0`에 내부 패킷이 없다면 `allowed-ips`나 라우팅을 확인해야 한다. `wg0`까지 패킷이 도착했는데 HTTP 응답이 없다면 veth 연결, namespace 내부 라우팅, 또는 HTTP 서버 로그를 확인해야 한다.