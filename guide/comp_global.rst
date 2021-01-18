
.. program:: M2

m2.global
******************

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
                "onEvent": "invalid"
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

.. data:: disks

    *  콘텐츠 저장 디스크 목록
    *  최대 개수 255개
    *  미구성시 메모리 모드로 동작

    .. data:: path=<PATH>

        디스크 경로

    .. data:: quota=<N>, GB
        
        디스크 최대 캐싱용량


.. data:: error

    *  :option:`cycle` 동안 :option:`count` 실패하면 디스크 배제

    .. option:: cycle=<N>, 초

        실패 체크주기

    
    .. option:: count=<N>, 회

        최대 실패회수




미분류 ``TO DO``
=======


*  <Server><Cache><Listen>
