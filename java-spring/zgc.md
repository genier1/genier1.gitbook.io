# ZGC 정리

#### ZGC(Z Garbage Collector)란

* JDK 11에 실험적 기능으로 추가되었고, JDK 15에서 정식 GC로 인정된 다음 LTS 버전인 JDK 17에도 반영되었다.
* STW 시간이 10ms를 초과하지 않는다.
  * 따라서 ZGC는 low latency를 요구하는 애플리케이션에 적합하다.
* ZGC는 8MB부터 16TB까지의 heap 크기를 지원한다.
* 커맨드 라인에서 `-XX:+UseZGC`옵션을 주면 ZGC를 사용할 수 있다.

#### ZGC 메모리 구조

*   ZPage

    * 이전의 G1 GC는 메모리를 Region 단위로 구분하였으나 ZGC에서는 메모리를 ZPage라는 논리적 단위로 구분하고 있음
    * Small, Medium, Large 타입이 있으며 Small타입은 2MB 이하의 객체가 들어갈 수있음
      * ZPage에는 단 하나의 객체만 할당할 수 있으므로, Large 타입의 ZPage 크기가 Small 타입의 ZPage 크기보다 작을 수 있다.



    <figure><img src="https://github-production-user-asset-6210df.s3.amazonaws.com/19471818/326256196-efeeac96-f61d-4c68-b695-b990d0f4e583.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&#x26;X-Amz-Credential=AKIAVCODYLSA53PQK4ZA%2F20240504%2Fus-east-1%2Fs3%2Faws4_request&#x26;X-Amz-Date=20240504T134040Z&#x26;X-Amz-Expires=300&#x26;X-Amz-Signature=b167e451440817e9e46eddc8beba30b4874aa73408bd59d28d15b12561f4c433&#x26;X-Amz-SignedHeaders=host&#x26;actor_id=19471818&#x26;key_id=0&#x26;repo_id=783676058" alt=""><figcaption><p>출처: <a href="https://d2.naver.com/helloworld/9581727">https://d2.naver.com/helloworld/9581727</a></p></figcaption></figure>
*   Reference coloring

    *   ZGC는 가상 메모리 주소를 위해, 기본보다 6비트 적은 42비트를 사용한다.

        * 다른 4비트는 GC metadata (finalizable, remapped, mark1, mark0) 를 저장하는 용도로 사용한다.

        ![](https://github.com/genier1/genier1.gitbook.io/assets/19471818/24f5e3f1-dffc-4240-b5a3-b8a0f176ba1b)
    * colored pointer에는 marked0, marked1, remapped, finalizable이 있다.
      * 이중 marked0과 marked1, remapped는 각 포인터를 사용하는 GC 단계에서 마스킹을 통해 가상 메모리 주소를 가져오는 데 사용된다.
      * marked0과 marked1, remapped은 다른 가상메모리 주소를 갖고 있지만 물리 메모리 주소는 모두 같다.
        * 3개의 가상주소를 하나의 물리주소에 매핑하려면 가상 주소 하나만 물리 주소에 매핑하고 각 단계에서 마스킹을 한번 더 실행해 물리 주소를 얻을수도 있다.
        * 그러나, ZGC는 가상 주소와 물리주소를 매핑하는 연산을 절약하기 위해 모든 가상 주소에 대해 mmap을 실행한다.
    * colored pointer로 인해 ZGC를 사용하는 환경에서는 RSS(resident set size)가 실제 메모리 사용량보다 3배 크게 관측된다.
      * 트래픽이 많은 환경에서 ZGC를 사용할 때에는 각 애플리케이션의 JVM heap 설정에 맞게 maxmapcount 값을 수정해야 한다. 그렇지 않으면 JVM 크래시가 날 가능성이 높다.
        * ZGC는 Linux 커널을 사용하는 환경에서 물리 메모리와 가상 메모리 매핑시 mmap을 사용하는데, mmap은 커널의 maxmapcount 설정만큼만 실행할 수 있다.
        * 공식적으로 (max\_capacity/ZGranuleSize) x 3 x 1.2을 maxmapcount로 설정하도록 권고함.

    #### load barrier

    * load barrier는 heap으로부터 참조가 일어날 때마다 실행되는 코드이다.
      * G1 GC에서는 참조가 해제될 때 참조의 상태를 검사하는 write barrier를 사용했다.
    * load barrier에서 객체가 정상적인 상태가 아니라고(bad color) 판단되면 slow path 과정을 실행한 후 참조를 진행한다.
      * coloring pointer 정보로 현 객체의 상태를 식별한다.
      * flag bit를 마스킹해 bad color인지를 판단하며, bad color일 경우 load barrier가 slow path를 실행한다.
      * slow\_path에 진입하면 reloacation, remark, remapping을 실행한다.

    #### GC 단계별 실행과정

    * GC는 크게 Marking(ZPhaseMark, ZPhaseMarkCompleted)와 relocating(ZPhaseRelocate)로 나뉜다.
      * 총 10단계로 이뤄지는데, phase 1 \~ 5는 Marking, phase 6 \~ 10은 relocating 과정이다.
    * 각 단계별 진행과정
      * phase 1 - Mark Start
        * STW를 발생시키고 각 스레드의 로컬 변수들을 스캔한다. 스레드에서 스캔되는 로컬 변수를 GC root라 하고, GC root set을 만든다. 스레드별 로컬 변수는 많지 않으므로 STW시간은 매우 짧다.
          * 이전세대 Major GC에서는 STW시간이 긴 것이 문제였다.
      * phase 2 \~4 - Concurrent Marking
        * 멀티 스레드로 GC root에서 접근 가능한 객체에 coloring과 remapping을 실행한다.
        * root set에서 시작해 객체 그래프를 탐색하며 도달할 수 있는 객체는 살아있는 걸로 표시한다.(marking)
        * STW를 발생시켜 스레드로컬의 marking 버퍼를 탐색하며 비운다.
      * phase 5 - Relocation Start
        * marking이 끝난 후 STW를 발생시켜 soft reference, week reference, phantom reference에 대한 처리를 진행한다.
      * phase 6\~7
        * 이전의 GC 사이클에서 식별한 relocation set을 초기화한다.
      * Phase 8\~10
        * ZGC는 garbage를 해제하고 살아 있는 객체를 ZPage에 재할당한다.
        * STW를 발생시켜 relocation을 실행한다.

#### 참고링크

* [https://d2.naver.com/helloworld/0128759](https://d2.naver.com/helloworld/0128759)
* [https://www.blog-dreamus.com/post/zgc에-대해서](https://www.blog-dreamus.com/post/zgc%EC%97%90-%EB%8C%80%ED%95%B4%EC%84%9C)
* [https://johngrib.github.io/wiki/java/gc/zgc/](https://johngrib.github.io/wiki/java/gc/zgc/)
