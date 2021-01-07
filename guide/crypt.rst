.. _engine-crypt:

11장. 암호화 엔진
******************

암호화 엔진은 ``on the fly`` 로 콘텐츠를 암호화한다.


.. toctree::
   :maxdepth: 2



암호화
====================================

On-the-fly로 콘텐츠를 암호화하여 전송한다. 

.. figure:: img/drm1.png
   :align: center

::

   $ server.xml - <Server><VHostDefault><Options>
   $ vhosts.xml - <Vhosts><Vhost><Options>

   <Drm Status="Inactive" Keyword="drm" MaxSourceSize="500">
      <Algorithm>RC4</Algorithm>
      <IV> ... </IV>
      <Token> ... </Token>
      <Key Hash="none">$Token</Token>
   </Drm>

-  ``<Drm>`` DRM 방식을 설정한다. ``Status="Active"`` 로 설정되면 활성화된다. 
   서비스 주소 뒤에 ``Keyword`` 를 suffix로 붙여 DRM을 구동한다. ::

      // URL
      www.example.com/music.mp3

      // DRM 처리된 URL
      www.example.com/music.mp3/drm/

   ``MaxSourceSize (기본: 500 MB)`` 를 초과하는 콘텐츠에 대해서는 500 Internal Error로 처리한다.


-  ``<Algorithm> (기본: RC4)`` 
   암호화 알고리즘을 선택한다.
   사용가능한 알고리즘은 다음과 같다.

   ================== ============
   <Algorithm>        Bits
   ================== ============
   RC4                40 ~ 2048
   ================== ============

-  ``<IV>`` Initial Vector.

-  ``<Token>`` 키 생성에 사용될 토큰

-  ``<Key> (기본: $Token)`` 변수를 조합하여 암/복호화에 사용될 키를 생성할 수 있다.
   
   ================== ==================================
   변수                설명
   ================== ==================================
   $Token             <Token> 의 값
   $url               클라이언트가 요청한 URL
   $filename1         확장자를 포함한 파일이름
   $filename2         확장자를 제외한 파일이름
   ================== ==================================

   콤마(,)를 구분자로 사용하여 키를 생성한다. 
   
   URL이 /music/iu.mp3 이고 ``<Token>`` 을 ABC로 가정한다면 ``<Key>`` 표현에 따른 암/복호화 키는 다음과 같다.
   
   ========================= ==================================
   <Key Hash="none">         암/복호화 키
   ========================= ==================================
   $Token                    ABC
   $url,$Token               /music/iu.mp3ABC
   $Token,$filename1         ABCiu.mp3
   $filename2,$Token,$url    iuABC/music/iu.mp3
   ========================= ==================================

   ``Hash (기본: none)`` 속성이 ``none`` 이라면 위에 조합된 문자열을 암/복호화 키로 사용한다.

   ``Hash`` 속성을 지정하면 아래와 같이 조합된 문자열을 해쉬한 값을 키로 사용한다. ::

      Hash( iuABC/music/iu.mp3 )

   ``Hash`` 속성은 ``none`` , ``MD5`` , ``SHA-1`` , ``SHA-256`` 를 지원한다.
   

.. note::

   DRM 관련 변수들( ``<Algorithm>`` , ``<IV>`` , ``<Token>`` , ``<Key>`` ) 은 동적으로 변경이 불가능하므로 신중히 설정해야 한다.
   왜냐하면 이전 키로 암호화된 파일들을 새로운 키로 풀 수 없기 때문이다.
   따라서 키를 변경해야 하는 상황이라면 해당 가상호스트의 캐싱을 초기화한 뒤 서비스하는 것이 안전하다.

    
``<IV>`` 와 ``<Token>`` 를 평문(Plain Text)으로 제공하면 보안적으로 취약하다.
이를 아래 API를 이용해 암호화한 뒤 설정할 것을 권장한다. ::

   /command/encryptpassword?plain=abcdefghijklmnop

암호화된 ``<IV>`` , ``<Token>`` 설정을 위해 ``Type="enc"`` 속성을 추가한다. ::

   $ server.xml - <Server><VHostDefault><Options>
   $ vhosts.xml - <Vhosts><Vhost><Options>

   <Drm Status="Active" Keyword="drm">
      <Algorithm>RC4</Algorithm>
      <IV Type="enc">RokyekMd0IjDnRGKjVE7sQ==</IV>
      <Token Type="enc">x4KHA1b+AirBOIoaeEBHmg==</Token>
      <Key>$Token</Key>
   </Drm>


.. note::

   암호화 API는 인증서에 기반하여 동작한다. 
   따라서 인증서가 다르면 암/복호화 결과가 다르다.