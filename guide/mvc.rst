.. _mvc:

4장. 엔드포인트
******************

이 장에서는 M2의 동작단위인 엔드포인트와 그 안의 MVC 구조에 대해 설명한다. 
이에 앞서 STON 가상호스트를 생성주어야 하는데 M2는 STON의 원본서버로 동작한다. 
따라서 다음과 같이 설정되어 있어야 한다. ::

   # vhosts.xml

   <Vhosts>
      <Vhost Name="www.example.com">
         <Origin>
            <!-- M2서비스 포트는 Loopback의 8585포트를 사용한다. -->
            <Address>127.0.0.1:8585</Address>
         </Origin>
         <M2 Status="Active">
            ... (생략) ...
         </M2>
         <Options>
            <!-- Bypass를 켜서 캐싱엔진을 거치지 않는다. -->
            <BypassPostRequest>ON</BypassPostRequest>
            <BypassGetRequest>ON</BypassGetRequest>
            <BypassPutRequest>ON</BypassPutRequest>
         </Options>
         <OriginOptions>
            <!-- M2를 배제시키지 않는다. -->
            <Exclusion>0</Exclusion>
            <ReuseTimeout>0</ReuseTimeout>
         </OriginOptions>
      </Vhost>
   </Vhosts>

엔드포인트는 멀티 구성이 가능하며 내부적으로 MVC 구조로 동작한다.

.. figure:: img/m2_13.png
    :align: center


.. note::

   생성된 결과를 STON이 캐싱할 경우 해당 엔드포인트는 TTL 시간동안 호출되지 않는다.


.. _mvc-conf:

엔드포인트 설정
====================================

엔드포인트는 `STON 가상호스트 <https://ston.readthedocs.io/ko/latest/admin/environment.html#vhosts-xml>`_ 의 ``<M2>`` 태그 하위에서 설정한다. ::

   # vhosts.xml - <Vhosts><Vhost>

   <M2 Status="Active">
       <Endpoints>
           <Endpoint Alias="inven" Post="ON" Get="ON">
               <Control ViewParam="view" ModelParam="model">/store/inventory</Control>
               <Model>https://foo.com/#model</Model>
               <Mapper>https://foo.com/mapper.json</Mapper>
               <View>https://bar.com/#view</View>
           </Endpoint>
           <Endpoint Alias="platinum_user" Post="ON" Get="ON">
               <Control ViewParam="myv" ModelParam="mym">/users/platinum</Control>
               <Model>https://alice.com/bob/#model.json</Model>
               <View>https://bar.com/#view</View>
           </Endpoint>
       </Endpoints>
   </M2>


``<M2>`` 태그의 ``Status`` 속성이 ``Active`` 일 때 활성화된다. 모델에 따라 독립된 ``<Endpoints>`` 를 멀티로 구성한다.

-  ``<Endpoint>`` 단위 엔드포인트를 설정한다.

   -  속성
      -  ``Alias (옵션)`` 엔드포인트의 별칭. 복합모델 생성에 사용.
      -  ``Post (기본: ON)`` Post 메소드 허용 여부
      -  ``Get (기본: ON)`` Get 메소드 허용 여부

   -  하위 태그

      -  ``<Control>`` 서비스할 URL을 설정한다. ``ViewParam`` , ``ModelParam`` 속성을 통해 HTTP QueryString 키 값을 설정한다.
      -  ``<Model> 모델 API 주소를 설정한다. ``ModelParam`` 의 값이 ``#model`` 키워드로 치환된다.
      -  ``<Mapper>`` JSON 모델일 경우 바로 뷰에서 참조 가능하지만 ``Mapper`` 를 추가해 JSON을 변경하거나 다른 포맷을 공통 포맷(M2-JSON)으로 맵핑한다.
      -  ``<View>`` 뷰가 게시된 URL을 설정한다. ``ViewParam`` 의 값이 ``#view`` 키워드로 치환된다.


.. note::

   ``<Mapper>`` 가 하나인 이유는 M2의 철학에 기인한다.

   -  ``<Model>`` 은 상품정보처럼 다양하지만 그 형식은 단일하다. 그러므로 ``<Model>`` 을 해석/맵핑하는 방식은 단일하다.
   -  ``<Model>`` 과 ``<Mapper>`` 는 1:1의 관계이며 이를 하나의 ``<Endpoint>`` 로 게시한다.
   -  만약 단일한 모델 URL의 해석/맵핑 방식이 다양하다면 각각 구분된 ``<Endpoint>`` 로 구성해야 한다. 멀티 ``<Endpoint>`` 로의 라우팅은 STON이 처리한다.


Model
====================================

모델은 데이터를 의미하며 일반적으로 RESTful API로 제공된다. 단일모델 사용시 접두어는 ``model.`` 이며 모델배열의 경우 ``model[0].`` 처럼 배열 인덱스를 사용한다.


Model 배열
------------------------------------

하나의 뷰에 동일한 형태의 여러 모델이 필요한 경우 배열을 사용한다. ::

   /users/platinum?mym=[apple,banana,cherry]&view=catalog

위와 같이 ``#model`` 에 대응하는 값을 ``[ ... ]`` 형식으로 입력한다. M2는 ``<Model>`` 에 설정된 주소에 각각의 값을 바인딩하여 결과를 배열로 취합한다. 이렇게 생성된 배열의 키는 쿼리스트링 키로 맵핑된다. ::

   {
      "mym" : [
         { "name": "apple", ... },
         { "name": "banana", ... },
         { "name": "cherry", ... }
      ]
   }

위와 같은 모델 배열을 생성하기 위해 아래의 API 호출이 발생한다. ::

   https://alice.com/bob/apple.json
   https://alice.com/bob/banana.json
   https://alice.com/bob/cherry.json

모든 API 호출이 성공하면 다행이겠지만 일부만 성공할 가능성이 있다. 이런 일부 모델의 실패 상황을 ``Sparse`` 속성으로 대처할 수 있다. ::

   # vhosts.xml - <Vhosts><Vhost><M2><Endpoints><Endpoint>

   <Model Sparse="Off">https://alice.com/bob/#model.json</Model>

-  ``Sparse (기본: OFF)`` 모델 참조가 하나라도 실패하면 실패처리한다. ``ON`` 설정이라면 모든 모델 참조가 실패할 경우에만 실패처리 된다.

예를 들어 ``Sparse="On"`` 인 상황에서 apple과 cherry의 모델 참조가 실패하면 모델 배열은 다음과 같이 구성된다. ::

   {
      "mym" : [
         { },
         { "name": "banana", ... },
         { }
      ]
   }


쿼리스트링 모델변수
------------------------------------

``<Model>`` 설정 외에 클라이언트로부터 직접 모델을 입력받을 수 있다. ::

   /users/platinum?mym=[apple,banana,cherry]&view=catalog&mynumber=123456&myage=24


위 주소의 쿼리 스트링 중 ``ModelParam`` 과 ``ViewParam`` 를 제외하면 약속된 쿼리스트링이 아니다. 이러한 키/값 쌍은 ``__query`` 의 자식 노드로 접근 가능하다. ::

   {
      "__query" : {
         "mynumber": "123456",
         "myage": "24"
      }
   }


모델 결합
------------------------------------

``<Endpoint>`` 는 독립적으로 서로 영향을 받지 않는다. ::

   # vhosts.xml - <Vhosts><Vhost>

   <M2 Status="Active">
      <Endpoints>
         <Endpoint Alias="inven"> ... </Endpoint>
         <Endpoint Alias="golduser"> ... </Endpoint>
      </Endpoints>
   </M2>


.. figure:: img/m2_userguide_05.png
    :align: center


두 모델의 값을 비교,연산해야하는 경우가 있을 수 있다. 이런 경우 모델들을 결합하는 별도의 ``<Endpoint>`` 를 만들면 가능하다. ::

   # vhosts.xml - <Vhosts><Vhost>
   
   <M2 Status="Active">
      <Endpoints>
         <Endpoint Alias="inven"> ... </Endpoint>
         <Endpoint Alias="golduser"> ... </Endpoint>
         <Endpoint Alias="golditem">
            <Control ViewParam="view" ModelParam="model">/items/gold</Control>
            <Mapper>https://foo.com/mapper.json</Mapper>
            <View>https://bar.com/#view</View>
         </Endpoint>
      </Endpoints>
   </M2>

-  ``<Model>`` 태그가 없다면 모델 결합을 위한 ``<Endpoint>`` 로 인식한다.
-  ``@Alias`` 를 통해 다른 M2-JSON을 참조한다. (예. ``@inven`` , ``@golduser`` )

결합 맵퍼는 다음과 같이 작성한다. ::

   {
      "item" : {
         "inventory" : "@inven",
         "user" : "@golduser"
      },
      "description" : "this is a compound model"
   }

.. figure:: img/m2_userguide_06.png
    :align: center

``@Alias`` 뒤에 뷰를 명시하면 M2-JSON을 가공한 뷰를 참조할 수 있다. 단, 해당 뷰의 형식은 반드시 JSON이어야 한다.

.. figure:: img/m2_userguide_07.png
    :align: center

예제의 ``golditem`` 는 ``@inven`` 과 ``@golduser`` 의 엔드포인트를 참조한다. 따라서 각각의 모델 값을 ``키:값`` 을 콤마로 구분한다. ::

   /items/gold?mode=inven:1000,golduser:javalive&view=img



내장변수
------------------------------------

내장변수는 __XXX 형식으로 표기되며 주로 M2-JSON의 메타 속성을 다루는 역할을 한다. ::

   {
      "firstName": "...",
      "address": {
         "streetAddress": "...",
         "city": "..."
      },
      "phoneNumber": ["..."],
      "__model_url" : "http://www.foo.com/goods?no=12345",
      "__model_raw" : "<html> ...(생략)... </html>"
   }

-  ``__model_url`` 모델이 참조된 URL
-  ``__model_raw`` 모델의 원시(RAW) 데이터 문자열



Mapper
====================================

맵퍼(Mapper)를 작성해 다양한 소스를 M2-JSON으로 맵핑(Mapping)한다.

.. figure:: img/m2_userguide_04.png
    :align: center


M2-JSON은 정보를 다루기 위한 JSON형식일 뿐 그 자체가 특별한 의미를 가지지 않는다. ::

   {
      "firstName": "...",
      "address": {
         "streetAddress": "...",
         "city": "..."
      },
      "phoneNumber": ["..."]
}

규칙은 간단하다.

-  값 참조 구분자는 ``space`` 이다. 예로 웹 페이지의 타이틀은 ``"html head title"`` 으로 표현한다.
-  맵핑하고 싶은 대상이 복수인 경우 값을 배열 ``["..."]`` 로 한다.



JSON
------------------------------------

-  JSON은 별도의 맵핑 없이 M2-JSON으로 사용 가능하다.



HTML/XML
------------------------------------

-  HTML과 XML 맵핑 규칙은 동일하며 추가적인 표현을 제공한다.
-  class 는 접두어 # 으로 참조한다.
-  id 는 접두어 . 으로 참조한다.
-  <Element>의 속성은 Element.속성키 으로 참조한다.

::

   <!DOCTYPE html>
   <html>
      <style type="text/css">
      <!--
         .foo {color:red};
         #bar {color:yellow};
         .foobar {color:cyan};
      //-->
      </style>
      <head>
         <title>Amazon.com: Online Shopping</title>
      </head>
      <body>        
         <h1>Amazon.com, Inc.</h1>
         <img id="foobar" src="https://amazon.com/logo.jpg" />
         <p class="foo">is an American multinational technology company </p>
         <p class="foo">based in Seattle that focuses on e-commerce,</p>
         <p class="foo">cloud computing, digital streaming, and artificial intelligence.</p>
      </body>
   </html>

예제 HTML은 다음과 같이 맵핑 가능하다. ::

   {
      "myTitle" : "html head title",
      "meta" : {
         "logo" : "#foobar img.src",
         "name" : "html body h1",
      },
      "descriptions" : [ ".foo"],
   }

위 맵핑은 아래와 같은 M2-JSON으로 변환된다. ::

   {
      "myTitle" : "Amazon.com: Online Shopping",
      "meta" : {
         "logo" : "https://amazon.com/logo.jpg",
         "name" : "Amazon.com, Inc.",
      },
      "descriptions" : [ 
         "is an American multinational technology company",
         "based in Seattle that focuses on e-commerce,",
         "cloud computing, digital streaming, and artificial intelligence."
      ]
   }


View
====================================

뷰(View)는 M2-JSON을 가공하여 사용자가 원하는 출력을 생성하는 템플릿을 의미한다. 
`Nunjucks <https://mozilla.github.io/nunjucks/>`_ 언어를 통해 M2-JSON을 다룬다.

.. figure:: img/m2_userguide_08.png
    :align: center

.. note::
   
   View가 주어지지 않을 경우 렌더링 없이 Model을 응답한다.


`Nunjucks <https://mozilla.github.io/nunjucks/>`_ 는 Jinja2에 영감을 받은 JavaScript 템플릿 언어이다. 
따라서 기본적인 `Jinja2 <https://jinja.palletsprojects.com/>`_ 의 문법이나 필터를 그대로 사용 가능하다. ::

   {
      "firstName": "John",
      "lastName": "Smith",
      "age": 25,
      "address": {
         "streetAddress": "21 2nd Street",
         "city": "New York",
         "state": "NY",
      "postalCode": "10021"
         },
      "phoneNumber": [
         { "type": "home", "number": "212 555-1234" },
         { "type": "fax", "number": "646 555-4567" }
      ]
   }

`Nunjucks <https://mozilla.github.io/nunjucks/>`_ 형식으로 다음과 같이 참조 가능하다. ::

   {{ model.firstname }}
   {{ model.address.state }}
   {{ model.phoneNumber.0.number }}


조건문, 반복문을 지원한다. ::

   {% if hungry %}
     I am hungry
   {% elif tired %}
     I am tired
   {% else %}
     I am good!
   {% endif %}


::

   <h1>Posts</h1>
   <ul>
   {% for item in items %}
      <li>{{ item.title }}</li>
   {% else %}
      <li>This would display if the 'item' collection were empty</li>
   {% endfor %}
   </ul>


HTML, XML
------------------------------------

HTML, XML 템플릿을 만든다. ::

   <html>
   <body>
      <H1>{{ model.firstname }} {{ model.lastName }}</H1>
      <p>{{ model.address.city }}</p>
   </body>
   </html>


JPG, PNG, WEBP, BMP, PDF
------------------------------------

이미지 출력은 HTML 템플릿을 기반으로 렌더링한다. 
<meta> 태그를 통해 출력 포맷을 지정한다. 
다음은 PNG 이미지를 가로 400, 세로 300으로 출력하는 예제이다. ::

   <!DOCTYPE html>
   <html>
      <head>
         <meta name="m2-render-png" content="width=400;height=300;" />
         <style>
            p { display: block; margin-top: 1em; margin-bottom: 1em; }
         </style>
      </head>
      <body>
         <H1>{{ model.firstname }} {{ model.lastName }}</H1>
         <p>{{ model.address.city }}</p>
      </body>
   </html>

이하 이미지 포맷에 따라 ``name`` 값과 지원 옵션이 다르다. 입력되지 않은 기본 값은 다음과 같다.

============== ================= ========================
속성            설명               기본값
============== ================= ========================
``width``       가로 픽셀         400
``height``      세로 픽셀         300
``quality``     JPEG 품질(%)      100
============== ================= ========================


이미지 포맷별 ``<meta>`` 태그 예제는 다음과 같다.

-  JPG ::
      
      <meta name="m2-render-jpg" content="width=400;height=300;quality=85" />

-  PNG ::
      
      <meta name="m2-render-png" content="width=400;height=300;" />

-  WEBP ::
      
      <meta name="m2-render-webp" content="width=400;height=300;quality=85" />

-  BMP ::
      
      <meta name="m2-render-bmp" content="width=400;height=300;" />

-  PDF ::
      
      <meta name="m2-render-pdf" content="width=400;height=300;scale=1;margin-top: 10px;margin-bottom:10px;margin-right:10px;margin-left:10px;" />


MP4, GIF
------------------------------------

비디오, Animated GIF 등 시간흐름이 필요한 포맷은 연속된 장면( ``<Scene>``)을 연결하여 만든다.

.. figure:: img/m2_userguide_09.png
    :align: center


다음과 같이 ``<Scene>`` 태그를 통해 각 화면을 구성한다. ::

   <!DOCTYPE html>
   <html>
      <head>
         <meta name="m2-render-gif" content="width=400;height=300;delay=1000;" />
         <style>
            p { display: block; margin-top: 1em; margin-bottom: 1em; }
         </style>
      </head>
      <body>
         <Scene>
            <Div style="background-color: blue;">
               <H1>{{ model.firstname }} {{ model.lastName }}</H1>
               <p>{{address.city}}</p>
            </Div>
         </Scene>
         <Scene>
            <Div style="background-color: blue;">
               <H1>{{ model.lastName }} {{ model.firstname }} </H1>
               <p>{{ model.address.city }}</p>
            </Div>
         </Scene>
         <Scene>
            <Div style="background-color: green;">
               <H1>{{ model.lastName }} {{ model.firstname }} ({{ model.age }})</H1>
               <p>{{ model.address.city }}</p>
            </Div>
         </Scene>
      </body>
   </html>

``<Scene>`` 태그는 의미가 없다. 따라서 ``<Div>`` 를 넣어 영역을 구분하면 개발 단계에서 쉽게 확인이 가능하다.

-  MP4 ::
      
      <meta name="m2-render-mp4" content="width=400;height=300;interval=1000;" />


-  GIF ::
      
      <meta name="m2-render-gif" content="width=400;height=300;delay=1000;" />

   -  장면 시간( ``delay (단위: ms)`` ) = 1000


JSON
------------------------------------

JSON 템플릿을 만든다. ::

   {
      "myName" : "{{firstname}} {{lastName}}",
      "myCity" : "{{address.city}}"
   }



.. _mvc-control:

Control (Web API)
====================================

클라이언트는 M2가 게시한 엔드포인트(API)를 HTTP로 호출한다.


GET Method
------------------------------------

결합할 모델(=정보)과 뷰(=표현)를 QueryString으로 입력한다. ::

   GET /myendpoint?model=wine&view=catalog


POST Method
------------------------------------

Post 메소드는 캐싱되지 않지만 단위 테스트 및 개발 용도로 지원된다. Body와 QueryString을 혼합해 사용 가능하다. ::

   # GET 방식과 동일
   POST /myendpoint?model=wine&view=catalog
   
   { }


::

   # Model과 View 업로드

   POST /myendpoint

   {
        "model" : { ... },
        "view" : "<html>...</hmtl>"
   }


::

   # View만 업로드

   POST /myendpoint?model=wine

   {
       "view" : "<html>...</hmtl>"
   }



::

   # Model만 업로드
   POST /myendpoint?view=catalog

   {
       "model" : { }
   }




