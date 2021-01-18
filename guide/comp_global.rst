
.. program:: M2

Global Componenets
==========

``전역 컴포넌트`` 란 ``Runtime`` 내의 모든 가상호스트, 함수들이 공용으로 사용하는 컴포넌트를 의미한다. 
대표적으로 캐시가 존재한다.


m2.global.cacheEnv
=======

``m2.vhost.cache`` 구동환경을 구성한다. ::

    "cacheEnv" : {
        "storage" : {
            "failSec": 60,
            "failCount": 10,
            "onCrash": "hang",
            "disks" : [
                { "path": "/cache1" }, 
                { "path": "/cache2" }, 
            ]
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


.. data:: storage

    ``failSec`` 동안 ``failCount`` 만큼 디스크 I/O가 실패하면 해당 디스크는 자동으로 배제된다. 
        배제된 디스크 상태는 `` "Invalid" ``로 표기된다.

    .. data:: failSec=<N>

        Default: ``60`` 초

    .. data:: failCount=<N>

        Default: ``10`` 회

    .. note::

        ``failSec`` 동안 ``failCount`` 만큼 디스크 I/O가 실패하면 해당 디스크는 자동으로 배제된다. 
        배제된 디스크 상태는 `` "Invalid" ``로 표기된다.


    .. data:: failCount=<N>

        Default: ``10`` 회



미분류 ``TO DO``
=======


*  <Server><Cache><Listen>
