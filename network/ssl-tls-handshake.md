# SSL/TLS HandShake 과정 정리

#### SSL(Secure Sockets Layer)/TLS(Transport Layer Security)란

* 이전에는 Netscape가 개발한 SSL을 사용하였으나 SSL 3.1부터는 TLS로 이름이 변경되었다.
* 그러나 보통 TLS과 SSL를 섞어서 많이 사용하므로 SSL/TLS로 통칭한다.

### SSL/TLS Handshake 과정

{% embed url="https://github.com/genier1/genier1.gitbook.io/assets/19471818/73aa3316-97d6-4d46-b5b0-5569014f3d3a" %}
출처: [https://www.cloudflare.com/ko-kr/learning/ssl/transport-layer-security-tls/](https://www.cloudflare.com/ko-kr/learning/ssl/transport-layer-security-tls/)
{% endembed %}

#### Client Hello

* 클라이언트가 서버에 연결을 시도하며 전송하는 패킷
* 클라이언트가 사용 가능한 Cipher Suite(암호화 알고리즘 목록), Session ID, TLS 버전, Random Byte를 전송

#### Server Hello

* 서버에서는 클라이언트가 제공한 Cipher Suite 중 하나를 선택한다.
* 또한, 서버의 SSL 인증서, Random Byte를 Client에게 전달한다.

#### Certificate

* 인증서 내부에는 서버에서 발행한 공개키가 들어있다.
* 클라이언트에서는 서버에서 보낸 SSL 인증서를 CA(Certificate Authority)의 공개키를 통해 인증서의 서명을 검증한다.

#### **Server Key Exchange / ServerHello Done**

* 서버의 공개키가 SSL 인증서 내부에 없는 경우 서버가 직접 공개키를 전달함. 공개키가 SSL 인증서 내부에 있는 경우 Server Key Exchange는 생략한다.

#### Client Key Exchange

* Client에서는 Pre-master Key를 생성 한 후 서버의 공개키를 통해 이를 암호화 하여 서버에 전송한다.
* 키교환 알고리즘을 RSA가 아닌 Diffie-Hellman(DH, DHE 등) 알고리즘과 타원곡선 암호인 ECDHE(Elliptic Curve DHE)을 사용하게 된다면 Client가 데이터를 암호화할 대칭키(비밀키)를 보내는 것이 아니라 대칭키(비밀키)를 생성할 재료를 Client와 Server가 교환하게 된다
* 서버에서는 비밀키로 암호화된 Pre-master Key를 복호화 한다.

#### **ChangeCipherSpec / Finished**

* Client, Server 모두가 서로에게 보내는 Packet으로 교환할 정보를 모두 교환한 뒤 통신할 준비가 다 되었음을 알리는 패킷입니다. 그리고 'Finished' Packet을 보내어 SSL Handshake를 종료한다.

#### 참고링크

* [https://aws-hyoh.tistory.com/39](https://aws-hyoh.tistory.com/39)
* [https://www.cloudflare.com/ko-kr/learning/ssl/what-happens-in-a-tls-handshake/](https://www.cloudflare.com/ko-kr/learning/ssl/what-happens-in-a-tls-handshake/)
* [https://howhttps.works/ko/](https://howhttps.works/ko/)
