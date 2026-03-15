
**Tags:** #soap #xml #api #web-services #protocol #enterprise

## Overview

SOAP is a protocol that uses XML for message formatting. It is highly structured and standardized, often used in enterprise-level applications.

## SOAP Characteristics

- **Protocol:** Strict standards (WSDL, XML Schema)
- **Data Format:** XML only
- **Transport:** Mostly HTTP, but can also use SMTP, TCP
- **Built-in error handling**
- **More secure & ACID-compliant** (WS-Security)

---

## WSDL (Web Services Description Language)

SOAP APIs typically expose a WSDL file, which is an XML document published at a URL. This WSDL defines:

- The available operations (like GetUserDetails, AddCustomer, etc.)
- Request and response formats
- SOAP actions
- Data types
- Endpoint URLs

**Example WSDL URL:** `https://api.example.com/UserService?wsdl`

When you visit this URL, you'll see a long XML document describing everything needed to consume the service.

### WSDL Inspection Tools

- **SoapUI** (specialized SOAP testing)
- **Postman** (manually copy structure)
- **Programming SDKs** (like wsimport in Java, or Add Service Reference in .NET)

### Using WSDL in Postman

> [!note] Postman does not auto-parse WSDL, so you must:
> 
> 1. Open the WSDL URL in your browser
> 2. Find the operation you want
> 3. Copy the required XML structure
> 4. Paste it into Postman's Body
> 5. Set SOAPAction header (found in the WSDL too)

---

## SOAP Request Format in Postman

### Step-by-Step Setup

1. **Open Postman**
2. **Select POST** from the dropdown
3. **Enter URL:** `https://api.example.com/UserService`
4. **Go to Headers tab:**
    - Content-Type: `text/xml`
    - SOAPAction: `GetUserDetails`
5. **Go to Body tab:**
    - Select `raw`
    - Choose `XML` from dropdown

### Example SOAP Request

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:usr="http://example.com/user">
   <soapenv:Header/>
   <soapenv:Body>
      <usr:GetUserDetails>
         <usr:UserId>12345</usr:UserId>
      </usr:GetUserDetails>
   </soapenv:Body>
</soapenv:Envelope>
```

### Example SOAP Response

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:usr="http://example.com/user">
   <soapenv:Body>
      <usr:GetUserDetailsResponse>
         <usr:Name>John Doe</usr:Name>
         <usr:Email>john.doe@example.com</usr:Email>
      </usr:GetUserDetailsResponse>
   </soapenv:Body>
</soapenv:Envelope>
```

---

## SOAP Namespaces

### What Is a SOAP Namespace?

In XML (and SOAP), a namespace is a way to uniquely identify elements and avoid name collisions. It's like a "scope" or a "label" that tells the parser where a particular element comes from.

### Why Namespaces Are Needed

- XML elements can have the same names in different contexts (e.g., name, id)
- A namespace tells systems which schema or definition to follow

### SOAP Envelope with Namespaces

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                  xmlns:usr="http://example.com/user">
   <soapenv:Header/>
   <soapenv:Body>
      <usr:GetUserDetails>
         <usr:UserId>12345</usr:UserId>
      </usr:GetUserDetails>
   </soapenv:Body>
</soapenv:Envelope>
```

### Namespace Breakdown

- **`xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"`**
    
    - This is the standard SOAP namespace
    - It defines elements like `<Envelope>`, `<Header>`, and `<Body>`
- **`xmlns:usr="http://example.com/user"`**
    
    - This is a custom namespace for your specific service
    - It defines the structure of your custom elements (like GetUserDetails, UserId, etc.)
- **The prefixes** (soapenv, usr) are just aliases
    
- **The URI** is what actually matters
    

### XML Vocabulary Conflicts Example

```xml
<root xmlns:html="http://www.w3.org/1999/xhtml"
      xmlns:math="http://www.w3.org/1998/Math/MathML">
  <html:title>This is an HTML title</html:title>
  <math:title>This is a MathML title</math:title>
</root>
```

Both `html:title` and `math:title` are named title, but they belong to different namespaces. The namespace prefix (html, math) helps prevent naming collisions.

## Common Namespace URIs

|Prefix|URI|Purpose|
|---|---|---|
|soapenv|http://schemas.xmlsoap.org/soap/envelope/|SOAP 1.1 standard elements|
|soap|http://www.w3.org/2003/05/soap-envelope|SOAP 1.2 standard elements|
|xsd|http://www.w3.org/2001/XMLSchema|XML schema definitions|
|xsi|http://www.w3.org/2001/XMLSchema-instance|Used to indicate data types|

> [!tip] Pro Tip If you're ever debugging a SOAP call and getting strange parsing errors, double-check:
> 
> - The namespace prefixes
> - That your elements match the schema's namespace
> - The correct SOAP version and URI

---

## Troubleshooting

|Error|Possible Cause|
|---|---|
|500 Internal Server Error|Malformed XML or invalid request structure|
|401 Unauthorized|Bad or missing credentials (check the header or token)|
|No response|Wrong endpoint URL or SOAPAction|

---

## Complex SOAP Request Example

This example demonstrates multiple key features including namespaces, authentication headers, nested elements, attributes, and multiple data types.

### CreateOrder Service Request

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                  xmlns:ord="http://example.com/order"
                  xmlns:auth="http://example.com/auth">
  <!-- SOAP Header with authentication -->
  <soapenv:Header>
    <auth:AuthHeader>
      <auth:Username>api_user</auth:Username>
      <auth:Password>securePassword123</auth:Password>
      <auth:Token>xyz-abc-123-token</auth:Token>
    </auth:AuthHeader>
  </soapenv:Header>

  <!-- SOAP Body with the CreateOrder request -->
  <soapenv:Body>
    <ord:CreateOrderRequest>

      <!-- Customer Information -->
      <ord:Customer>
        <ord:CustomerID>12345</ord:CustomerID>
        <ord:Name>Jane Doe</ord:Name>
        <ord:Email>jane.doe@example.com</ord:Email>
        <ord:PhoneNumber type="mobile">+1-555-1234</ord:PhoneNumber>
        <ord:Address>
          <ord:Street>123 Elm Street</ord:Street>
          <ord:City>Springfield</ord:City>
          <ord:State>IL</ord:State>
          <ord:PostalCode>62701</ord:PostalCode>
          <ord:Country>USA</ord:Country>
        </ord:Address>
      </ord:Customer>

      <!-- Order Metadata -->
      <ord:OrderDetails>
        <ord:OrderDate>2025-06-30</ord:OrderDate>
        <ord:Priority>High</ord:Priority>
        <ord:Currency>USD</ord:Currency>
      </ord:OrderDetails>

      <!-- Order Items (repeating element) -->
      <ord:Items>
        <ord:Item>
          <ord:ProductID>1001</ord:ProductID>
          <ord:Quantity>2</ord:Quantity>
          <ord:Price>49.99</ord:Price>
        </ord:Item>
        <ord:Item>
          <ord:ProductID>1002</ord:ProductID>
          <ord:Quantity>1</ord:Quantity>
          <ord:Price>129.50</ord:Price>
        </ord:Item>
      </ord:Items>

      <!-- Optional Notes -->
      <ord:Notes>Leave package at the front door.</ord:Notes>

    </ord:CreateOrderRequest>
  </soapenv:Body>
</soapenv:Envelope>
```

### Key Features Illustrated

|Feature|Where It's Used|
|---|---|
|Namespaces|soapenv, ord, auth prefixes|
|SOAP Header|Auth block with credentials and token|
|Nested Elements|Customer, Address, Items > Item|
|Attributes|PhoneNumber type="mobile"|
|Multiple Data Types|String, integer, decimal, date|
|Optional Fields|Notes is optional|
|Repeating Elements|Multiple Item entries under Items|

---

## Example WSDL Structure

### OrderService.wsdl

```xml
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/"
             xmlns:tns="http://example.com/order"
             xmlns:xsd="http://www.w3.org/2001/XMLSchema"
             xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/"
             name="OrderService"
             targetNamespace="http://example.com/order">

  <!-- Data Types -->
  <types>
    <xsd:schema targetNamespace="http://example.com/order">
      <xsd:element name="CreateOrderRequest">
        <xsd:complexType>
          <xsd:sequence>
            <xsd:element name="CustomerID" type="xsd:string"/>
            <xsd:element name="OrderDate" type="xsd:date"/>
            <xsd:element name="Currency" type="xsd:string"/>
            <xsd:element name="Items" minOccurs="1" maxOccurs="unbounded">
              <xsd:complexType>
                <xsd:sequence>
                  <xsd:element name="ProductID" type="xsd:string"/>
                  <xsd:element name="Quantity" type="xsd:int"/>
                  <xsd:element name="Price" type="xsd:decimal"/>
                </xsd:sequence>
              </xsd:complexType>
            </xsd:element>
            <xsd:element name="Notes" type="xsd:string" minOccurs="0"/>
          </xsd:sequence>
        </xsd:complexType>
      </xsd:element>

      <xsd:element name="CreateOrderResponse">
        <xsd:complexType>
          <xsd:sequence>
            <xsd:element name="OrderID" type="xsd:string"/>
            <xsd:element name="Status" type="xsd:string"/>
          </xsd:sequence>
        </xsd:complexType>
      </xsd:element>
    </xsd:schema>
  </types>

  <!-- Messages -->
  <message name="CreateOrderInput">
    <part name="parameters" element="tns:CreateOrderRequest"/>
  </message>

  <message name="CreateOrderOutput">
    <part name="parameters" element="tns:CreateOrderResponse"/>
  </message>

  <!-- Port Type -->
  <portType name="OrderPortType">
    <operation name="CreateOrder">
      <input message="tns:CreateOrderInput"/>
      <output message="tns:CreateOrderOutput"/>
    </operation>
  </portType>

  <!-- Binding -->
  <binding name="OrderBinding" type="tns:OrderPortType">
    <soap:binding transport="http://schemas.xmlsoap.org/soap/http" style="document"/>
    <operation name="CreateOrder">
      <soap:operation soapAction="http://example.com/order/CreateOrder" />
      <input>
        <soap:body use="literal"/>
      </input>
      <output>
        <soap:body use="literal"/>
      </output>
    </operation>
  </binding>

  <!-- Service -->
  <service name="OrderService">
    <documentation>SOAP Order Service</documentation>
    <port name="OrderPort" binding="tns:OrderBinding">
      <soap:address location="https://api.example.com/soap/orderService"/>
    </port>
  </service>
</definitions>
```

### WSDL Sections Explained

|Section|Description|
|---|---|
|Types|Defines request/response structure (XSD)|
|Messages|Wrap the input/output XML elements|
|PortType|Defines the available operations|
|Binding|SOAP specifics (style, encoding, action URL)|
|Service|The actual endpoint and address|

---

## Related Topics

- [[REST APIs]] - Compare with RESTful services
- [[XML Schema]] - Understanding XSD validation
- [[Web Services Security]] - WS-Security implementation
- [[API Testing]] - Testing strategies for SOAP services