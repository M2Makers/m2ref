
.. program:: M2

m2.global
******************

모든 가상호스트에서 공용으로 사용하는 컴포넌트 또는 

``전역 컴포넌트`` 란 ``Runtime`` 내의 모든 가상호스트, 함수들이 공용으로 사용하는 컴포넌트를 의미한다. 
대표적으로 캐시가 존재한다.


cacheEnv
=======

``m2.vhost.cache`` 구동환경을 구성한다. ::

    "cacheEnv" : {
        "storage" : {
            "disks" : [
                { "path": "/cache1" }, 
                { "path": "/cache2",  "quota": 100 }, 
            ],
            "error": {
                "cycle": 60,
                "count": 10,
                "onCrash": "invalid"
            }
        },
        "memory": {
            "systemRatio": 100,
            "contentRatio": 50
        },
        "cleanUp": {
            "time": "02:00",
            "age": 0,
            "emptyDirectory": "delete"
        }
        "config": {
            "retentionDays": 30
        }
    }


storage
------

.. data:: disks=<LIST>

    *  콘텐츠 저장 디스크 목록
    *  최대 개수 255개
    *  미구성시 메모리 모드로 동작
    *  각 디스크마다 LRU(Least Recently Used)로 용량초과되지 않도록 동작

    .. data:: path=<PATH>

        디스크 경로

    .. data:: quota=<N>
        
        디스크 최대 캐싱 용량(GB)


.. data:: error

    :option:`cycle` 동안 :option:`count` 만큼 I/O가 실패하면 디스크 배제

    .. option:: cycle=<N>

        실패 카운팅 주기(초)
    
    .. option:: count=<N>

        최대 실패회수

    .. option:: onCrash=<ENUM>

        모든 디스크 배제시 동작방식

        *  ``hang (기본)`` - 복구없이 동작
        *  ``bypass`` - 모든 요청 원본 바이패스. 디스크 복구시 서비스 재개.
        *  ``selfkill`` - 데몬 종료


memory
------

.. data:: systemRatio=<PERCENTAGE>

    물리 메모리 사용비율. 예를 들어 8GB인 환경에서 이 값이 ``50`` 이라면 4GB로 처리함

.. data:: contentRatio=<PERCENTAGE>

    솔루션 가용메모리 중 Payload 적재비율


cleanUp
------

하루 한번 서비스부하가 가장 적은 시간에 디스크 클린업을 수행한다.


.. data:: time=<mm:ss>

    시작시간

.. data:: age=<N>

    ``0`` 보다 큰 경우 ``age`` 기간동안 미접근 콘텐츠 삭제

.. data:: emptyDirectory=<ENUM>

    빈 디렉토리 삭제 정책

    *  ``delete (기본)`` 삭제
    *  ``keep`` 유지



config
------

.. data:: retentionDays=<N>

    설정 유지기간(일)




미분류 ``TO DO``
=======


*  <Server><Cache><Listen>
