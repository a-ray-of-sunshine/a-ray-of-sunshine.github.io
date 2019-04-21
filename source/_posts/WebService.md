---
title: WebService
date: 2017-5-17 16:51:19
---

## Apache CXF

CXF 是一个开源的 service 框架。

CXF 使用 WSDL协议 描述发布一个服务，使用 SOAP协议 进行消息的交换，使用 HTTP协议 进行消息的传输。同时使用 JAX-WS API 向外暴露接口。

``` xml
<!-- request -->
<soap:Envelope
    xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
    <soap:Body>
        <ns2:getCustomersByName
            xmlns:ns2="http://customerservice.example.com/">
            <name>
                Smith
                </name>
            </ns2:getCustomersByName>
        </soap:Body>
    </soap:Envelope>

<!-- response -->
<soap:Envelope
    xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
    <soap:Body>
        <ns2:getCustomersByNameResponse
            xmlns:ns2="http://customerservice.example.com/">
            <return>
                <customerId>
                    0
                    </customerId>
                <name>
                    Smith
                    </name>
                <address>
                    Pine Street 200
                    </address>
                <numOrders>
                    1
                    </numOrders>
                <revenue>
                    10000.0
                    </revenue>
                <test>
                    1.5
                    </test>
                <birthDate>
                    2009-02-01+08:00
                    </birthDate>
                <type>
                    BUSINESS
                    </type>
                </return>
            <return>
                <customerId>
                    0
                    </customerId>
                <name>
                    Smith
                    </name>
                <address>
                    Pine Street 200
                    </address>
                <numOrders>
                    1
                    </numOrders>
                <revenue>
                    10000.0
                    </revenue>
                <test>
                    1.5
                    </test>
                <birthDate>
                    2009-02-01+08:00
                    </birthDate>
                <type>
                    BUSINESS
                    </type>
                </return>
            </ns2:getCustomersByNameResponse>
    </soap:Body>
</soap:Envelope>

```

## $. 参考
1. [SOAP (originally Simple Object Access Protocol)](https://en.wikipedia.org/wiki/SOAP)
2. [SOAP Version 1.1](https://www.w3.org/TR/2000/NOTE-SOAP-20000508/)
2. [SOAP Version 1.2](https://www.w3.org/TR/soap12-part1/)
3. [Web services protocol stack](https://en.wikipedia.org/wiki/Web_services_protocol_stack)
4. [Web Services Description Language(WSDL)](https://en.wikipedia.org/wiki/Web_Services_Description_Language)
5. [SOAP W3C page](https://www.w3.org/TR/soap/)
6. [Web Services Description Language (WSDL) Version 2.0 W3C page](https://www.w3.org/TR/wsdl20/)
7. [Web Services Description Language (WSDL) 1.1 W3C page](https://www.w3.org/TR/wsdl)