.. _example:

Appendix C: 예제
***********************


``<Endpoint>`` 설정
====================================

최소 설정
------------------------------------

최소(필수) 설정 예제. ::

   # vhosts.xml - <Vhosts><Vhost><M2><Endpoints>

   <Endpoint>
      <Model>
         <Source>https://foo.com/#model</Source>
      </Model>      
      <View>
         <Source>https://bar.com/#view</Source>
      </View>
      <Control>
         <Path>/banner</Path>
      </Control>
   </Endpoint>



최대 설정
------------------------------------

가능한 모든 설정 예제. ::

   # vhosts.xml - <Vhosts><Vhost><M2><Endpoints>

   <Endpoint Alias="sample">
      <Model>
         <Source>https://foo.com/#model</Source>
         <Sparse>ON</Sparse>
         <Mapper>https://foo.com/mapper.json</Mapper>
      </Model>      
      <View>
         <Source Mandatory="off">https://bar.com/#view</Source>
         <WellFormed>ON</WellFormed>
         <MetaDefault>
            <Item><![CDATA[ <meta name="m2-render-jpg" width="400" height="300" quality="85"> ]]></Item>
            <Item><![CDATA[ <meta name="m2-render-png" width="400" height="300"> ]]></Item>
            <Item><![CDATA[ <meta name="m2-render-webp" width="400" height="300" quality="85"> ]]></Item>
            <Item><![CDATA[ <meta name="m2-render-bmp" width="400" height="300"> ]]></Item>
            <Item><![CDATA[ <meta name="m2-render-pdf" width="400" height="300" scale="1" margin-top="10px" margin-bottom="10px" margin-right="10px" margin-left="10px"> ]]></Item>
            <Item><![CDATA[ <meta name="m2-render-mp4" width="400" height="300" interval="1000"> ]]></Item>
            <Item><![CDATA[ <meta name="m2-render-gif" width="400" height="300" delay="1000"> ]]></Item>
            <Item><![CDATA[ <meta name="m2-function-image" host="https://www.example.com/m2/image" split-height="500" class="mym2div" full="no" tool="/optimize" max-size="10"> ]]></Item>
         </MetaDefault>
      </View>
      <Control>
         <Path ModelParam="model" ViewParam="view" Post="off" Get="on">/banner</Path>
         <Module Name="aws_s3-backup">bucket:mybucket; object:/my/desired/key.txt;</Module>
      </Control>
   </Endpoint>

   

