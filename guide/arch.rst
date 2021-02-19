.. program:: nghttpx

Architecture
==========

``M2`` 아키텍쳐 컨셉은 확장성 있는 마이크로 서비스와 손쉬운 통합이다.
구조는 다음과 같다.

.. figure:: img/0001.png
   :align: center

각 구성요소의 역할과 책임은 다음과 같다.

*  ``Core`` - M2 라이프 사이클, RESTful API, 설정관리, 라이선스
*  ``Runtime`` - 서비스 런타임, 가상호스트 관리, 전역자원, 시스템 추상화
*  ``Modules`` - HTTP와 Payload를 다루는 단위 기능 라이브러리
*  ``Virtual Host`` - 가상호스트, 로그, 통계, 세션, 라우팅, 업/다운 스트림
*  ``Call Chain`` - 빌트인, 커스텀, 파이프라인, 외부 자원연계, 분기, 트레이스
*  ``Workload`` - 비지니스 로직, 콘텐츠



Call Chain
-----------------------

준비된 모듈을 유연하게 연결하여 Workload를 on the fly로 처리한다.

.. figure:: img/0002.png
   :align: center


이렇게 하나의 Workload를 처리하기 위해 연결된 흐름을 콜체인 ``Call Chain`` 이라고 부른다.


``Call Chain`` 동작방식은 `Open Tracing <https://opentracing.io/>`_ 의 SPANS 와 TRACE 컨셉으로 이해하면 쉽다.

.. figure:: img/opentracing.png
   :align: center


``Call Chain`` 은 ``HTTP Transaction`` 을 처리하는 일종의 Micro Service Bus로 3가지 흐름이 존재한다.

1. ``Runtime`` 에 의한 가상호스트 사이의 연결

   .. figure:: img/0005.png
      :align: center


2. ``Virtual Host`` 내에 사전 정의된 모듈의 연결.
   대표적으로 "임의의 외부 이미지를 다운로드/RESIZE 한 뒤 캐싱 서비스한다." 라면 복잡한 구성없이 모듈 활성화만으로 ``Call Chain`` 을 구성한다.

   .. figure:: img/0004.png
      :align: center


3. ``Virtual Host`` 내에 관리자가 임의로 구성한 연결.
   복잡한 비지니스 로직 구현이나 Legacy 연계에 적합하다.

   .. figure:: img/0006.png
      :align: center



모듈
-----------------------

모듈은 HTTP Transaction의 개별 구성요소를 다룰 수 있도록 개발된 단위 기능이다.
모듈의 대분류는 다음과 같다.

*  ``Config`` - 설정, 라이선스, 클러스터 등
*  ``Management`` - 로그, 통계, API, SNMP 등
*  ``Network`` - DNS, 소켓, 풀링, 헬쓰체커 등
*  ``HTTP`` - 위변조, ACL, URL 전처리 등
*  ``Payload`` - 도큐먼트, 이미지, 비디오 등
*  ``Traffic`` - 바이패스, 쓰로틀링 등
*  ``Cache`` - 메모리/디스크 캐싱, 파일시스템 등
*  ``Authentication`` - URL 암복호화, AWS 인증 등
*  ``Cloud`` - AWS/GCP/Azure 연계
*  ``APM`` - Datadog 등

