---
layout: post
title: "Building a CXF client for a WCF webservice with SecureConversation"
date: 2012-08-19 13:25
comments: true
categories:
 - Java
 - SOAP
 - CXF
---

Although RESTful webservices and JSON is what everyone seems to be using nowadays, SOAP is still very present. But while SOAP itself is a very established standard, its addons – like WS-SecureConversation – seem to be interpreted differently by different frameworks. And apparently, this sometimes makes interoperability quite a difficult thing to accomplish (which I did, in the end).

But, I’m getting ahead of myself: What I wanted to accomplish was to have a Java-client talk to a .net-Webservice with WS-SecureConversation enabled. To be more specific, I decided to use [Apache CXF](http://cxf.apache.org/) because Spring-WS does not support WS-SecureConversation, apparently (citation needed). I did not explore Metro or Axis, but they might have been valid options, too. The endpoint was created using standard WCF, as far as I know. I ran into quite the number of problems on my way to success which made me want to write this blog post.

The important parts were:

1. Importing the public key from the WSDL into a keystore
1. Using the “.sct” suffix in the CXF configuration
1. Dectivating “chunking”
1. Using the infamous [Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files](http://www.oracle.com/technetwork/java/javase/downloads/jce-6-download-429243.html)
1. Having the endpoint’s operator deactivate [MTOM](http://en.wikipedia.org/wiki/Message_Transmission_Optimization_Mechanism)

## Public key from WSDL to keystore
WCF embeds the public key used for encryption into the WSDL. Alas, CXF does not recognize that, so I had to import that key into a keystore and provide that to CXF. The snippet from the WSDL looks something like this:

``` xml
<wsdl:port name="PortName" binding="tns:wsPortName">
	<soap12:address location="http://..." />
	<wsa10:EndpointReference>
		<wsa10:Address>...
		</wsa10:Address>
		<Identity xmlns="http://schemas.xmlsoap.org/ws/2006/02/addressingidentity">
			<KeyInfo xmlns="http://www.w3.org/2000/09/xmldsig#">
				<X509Data>
					<X509Certificate>MIIHm[...]2Ij5wXI=</X509Certificate>
				</X509Data>
			</KeyInfo>
		</Identity>
	</wsa10:EndpointReference>
</wsdl:port>
```
What I did was putting the X509 certificate into a standard keyfile (I called it `certFromWsdl.cer`) like this:
    -----BEGIN CERTIFICATE-----
    MIIHmTCCBYGgAwIBAgIKSPx66QAAAAAETTANBgkqhkiG9w0BAQUFADBJMRUwEwYKCZImiZPyLGQB
    [...]
    9c9RNCY22Ij5wXI=
    -----END CERTIFICATE-----
and import it into a java keystore using [keytool](http://docs.oracle.com/javase/6/docs/technotes/tools/windows/keytool.html) like this:
    $ keytool -importcert -trustcacerts -alias servicekey -keystore my-keystore.jks -storepass mystorepass -file certFromWsdl.cer
The keystore will automatically be created if it did not exist.

Now, to make CXF recognize the keystore, I used a `.properties`-file. So, I put a file called `clientKeystore.properties` on my classpath that looks like this:

``` properties clientKeystore.properties
org.apache.ws.security.crypto.merlin.keystore.file=my-keystore.jks
org.apache.ws.security.crypto.merlin.keystore.password=mystorepass
org.apache.ws.security.crypto.merlin.keystore.type=jks
org.apache.ws.security.crypto.merlin.keystore.alias=servicekey
``` 
Now, when creating my client object, I put the appropriate parameter into the port’s request context:
``` java
URL wsdlLocation = new URL(this.wsdlLocation);
EnpointName endpointName = new EnpointName(wsdlLocation);
port = endpointName.getPortName();
Map<String, Object> ctx = ((BindingProvider) port).getRequestContext();
ctx.put("ws-security.username.sct", wssUsername);
ctx.put("ws-security.password.sct", wssPassword);
ctx.put("ws-security.encryption.properties.sct", "clientKeystore.properties");
Client client = ClientProxy.getClient(port);
``` 

## Chunking
WCF cannot deal correctly with [chunked requests](http://cxf.apache.org/docs/client-http-transport-including-ssl-support.html#ClientHTTPTransport%28includingSSLsupport%29-ANoteAboutChunking), i.e. requests that are not being sent in one part. What I saw was an Exception like this:
```
Caused by: java.net.SocketException: Software caused connection abort: recv failed
	at java.net.SocketInputStream.socketRead0(Native Method)
	at java.net.SocketInputStream.read(Unknown Source)
	at java.io.BufferedInputStream.fill(Unknown Source)
	at java.io.BufferedInputStream.read1(Unknown Source)
	at java.io.BufferedInputStream.read(Unknown Source)
	at sun.net.www.http.HttpClient.parseHTTPHeader(Unknown Source)
	at sun.net.www.http.HttpClient.parseHTTP(Unknown Source)
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream(Unknown Source)
	at java.net.HttpURLConnection.getResponseCode(Unknown Source)
	at org.apache.cxf.transport.http.HTTPConduit.processRetransmit(HTTPConduit.java:1002)
	at org.apache.cxf.transport.http.HTTPConduit.access$400(HTTPConduit.java:148)
	at org.apache.cxf.transport.http.HTTPConduit$WrappedOutputStream.handleRetransmits(HTTPConduit.java:1494)
	at org.apache.cxf.transport.http.HTTPConduit$WrappedOutputStream.handleResponse(HTTPConduit.java:1515)
	at org.apache.cxf.transport.http.HTTPConduit$WrappedOutputStream.close(HTTPConduit.java:1428)
	... 48 more
```
To deactivate chunking, I had to put a “cxf.xml” into my classpath that looks like this:
``` xml cxf.xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:http-conf="http://cxf.apache.org/transports/http/configuration"
       xmlns:jaxws="http://cxf.apache.org/jaxws"
       xsi:schemaLocation="http://cxf.apache.org/transports/http/configuration
           http://cxf.apache.org/schemas/configuration/http-conf.xsd
           http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd
           http://cxf.apache.org/jaxws http://cxf.apache.org/schemas/jaxws.xsd">

  <!-- This file will automatically be picked up by CXF if it is named cxf.xml and placed on the classpath -->

  <http-conf:conduit name="*.http-conduit">
    <http-conf:client Connection="Keep-Alive"
                      MaxRetransmits="1"
                      AllowChunking="false" />
  </http-conf:conduit>
</beans>
```
The file will automatically be picked up by CXF, so, although this is technically a Spring context, you do not have to load it explicitly. But, you need to place the “spring-context” library in your classpath. Using maven that would mean:
``` xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-context</artifactId>
  <version>3.0.6.RELEASE</version>
</dependency>
``` 
## Deactivating MTOM
It [looks like](http://cxf.547215.n5.nabble.com/Signature-digest-mismatch-when-NET-supplies-MTOM-attachment-td3270961.html) CXF does not support talking to an endpoint with MTOM and encryption enabled.

I saw an exception like this:
``` java
org.apache.ws.security.WSSecurityException: The signature or decryption was invalid
	at org.apache.ws.security.processor.ReferenceListProcessor.decryptEncryptedData(ReferenceListProcessor.java:314)
	at org.apache.ws.security.processor.ReferenceListProcessor.decryptDataRefEmbedded(ReferenceListProcessor.java:172)
	at org.apache.ws.security.processor.ReferenceListProcessor.handleReferenceList(ReferenceListProcessor.java:100)
	at org.apache.ws.security.processor.ReferenceListProcessor.handleToken(ReferenceListProcessor.java:60)
	at org.apache.ws.security.WSSecurityEngine.processSecurityHeader(WSSecurityEngine.java:396)
	at org.apache.cxf.ws.security.wss4j.WSS4JInInterceptor.handleMessage(WSS4JInInterceptor.java:289)
	at org.apache.cxf.ws.security.wss4j.WSS4JInInterceptor.handleMessage(WSS4JInInterceptor.java:97)
	at org.apache.cxf.phase.PhaseInterceptorChain.doIntercept(PhaseInterceptorChain.java:262)
	at org.apache.cxf.endpoint.ClientImpl.onMessage(ClientImpl.java:798)
	at org.apache.cxf.transport.http.HTTPConduit$WrappedOutputStream.handleResponseInternal(HTTPConduit.java:1667)
	at org.apache.cxf.transport.http.HTTPConduit$WrappedOutputStream.handleResponse(HTTPConduit.java:1520)
	at org.apache.cxf.transport.http.HTTPConduit$WrappedOutputStream.close(HTTPConduit.java:1428)
	at org.apache.cxf.transport.AbstractConduit.close(AbstractConduit.java:56)
	at org.apache.cxf.transport.http.HTTPConduit.close(HTTPConduit.java:658)
	at org.apache.cxf.interceptor.MessageSenderInterceptor$MessageSenderEndingInterceptor.handleMessage(MessageSenderInterceptor.java:62)
	at org.apache.cxf.phase.PhaseInterceptorChain.doIntercept(PhaseInterceptorChain.java:262)
	at org.apache.cxf.endpoint.ClientImpl.doInvoke(ClientImpl.java:532)
	at org.apache.cxf.endpoint.ClientImpl.invoke(ClientImpl.java:464)
	at org.apache.cxf.endpoint.ClientImpl.invoke(ClientImpl.java:367)
	at org.apache.cxf.endpoint.ClientImpl.invoke(ClientImpl.java:320)
	at org.apache.cxf.ws.security.trust.STSClient.requestSecurityToken(STSClient.java:734)
	at org.apache.cxf.ws.security.trust.STSClient.requestSecurityToken(STSClient.java:614)
	at org.apache.cxf.ws.security.trust.STSClient.requestSecurityToken(STSClient.java:606)
	at org.apache.cxf.ws.security.policy.interceptors.SecureConversationOutInterceptor.issueToken(SecureConversationOutInterceptor.java:159)
	at org.apache.cxf.ws.security.policy.interceptors.SecureConversationOutInterceptor.handleMessage(SecureConversationOutInterceptor.java:69)
	at org.apache.cxf.ws.security.policy.interceptors.SecureConversationOutInterceptor.handleMessage(SecureConversationOutInterceptor.java:44)
	at org.apache.cxf.phase.PhaseInterceptorChain.doIntercept(PhaseInterceptorChain.java:262)
	at org.apache.cxf.endpoint.ClientImpl.doInvoke(ClientImpl.java:532)
	at org.apache.cxf.endpoint.ClientImpl.invoke(ClientImpl.java:464)
	at org.apache.cxf.endpoint.ClientImpl.invoke(ClientImpl.java:367)
	at org.apache.cxf.endpoint.ClientImpl.invoke(ClientImpl.java:320)
	at org.apache.cxf.frontend.ClientProxy.invokeSync(ClientProxy.java:89)
	at org.apache.cxf.jaxws.JaxWsClientProxy.invoke(JaxWsClientProxy.java:134)
	... 27 more
Caused by: java.lang.ArrayIndexOutOfBoundsException
	at java.lang.System.arraycopy(Native Method)
	at org.apache.xml.security.encryption.XMLCipher.decryptToByteArray(XMLCipher.java:1750)
	at org.apache.xml.security.encryption.XMLCipher.decryptElement(XMLCipher.java:1612)
	at org.apache.xml.security.encryption.XMLCipher.decryptElementContent(XMLCipher.java:1650)
	at org.apache.xml.security.encryption.XMLCipher.doFinal(XMLCipher.java:978)
	at org.apache.ws.security.processor.ReferenceListProcessor.decryptEncryptedData(ReferenceListProcessor.java:312)
	... 59 more
``` 
MTOM has to be deactivated on the server side, so it looks like you're out of luck if you have no influence on that.

## Using strong security

The webservice I tried to talk to had AES-256 encryption enabled. This is not supported by the default JRE/JDK installation. So I got an exception like this:
``` java
org.apache.ws.security.WSSecurityException: Cannot encrypt data
	at org.apache.ws.security.message.WSSecEncrypt.encryptElement(WSSecEncrypt.java:493)
	at org.apache.ws.security.message.WSSecEncrypt.doEncryption(WSSecEncrypt.java:406)
	at org.apache.ws.security.message.WSSecDKEncrypt.encryptForExternalRef(WSSecDKEncrypt.java:114)
	at org.apache.cxf.ws.security.wss4j.policyhandlers.SymmetricBindingHandler.doEncryptionDerived(SymmetricBindingHandler.java:482)
	... 50 more
Caused by: org.apache.xml.security.encryption.XMLEncryptionException: Illegal key size or default parameters
Original Exception was java.security.InvalidKeyException: Illegal key size or default parameters
	at org.apache.xml.security.encryption.XMLCipher.encryptData(XMLCipher.java:1140)
	at org.apache.xml.security.encryption.XMLCipher.encryptData(XMLCipher.java:1081)
	at org.apache.xml.security.encryption.XMLCipher.encryptElementContent(XMLCipher.java:855)
	at org.apache.xml.security.encryption.XMLCipher.doFinal(XMLCipher.java:985)
	at org.apache.ws.security.message.WSSecEncrypt.encryptElement(WSSecEncrypt.java:490)
	... 53 more
Caused by: java.security.InvalidKeyException: Illegal key size or default parameters
	at javax.crypto.Cipher.a(DashoA13*..)
	at javax.crypto.Cipher.a(DashoA13*..)
	at javax.crypto.Cipher.a(DashoA13*..)
	at javax.crypto.Cipher.init(DashoA13*..)
	at javax.crypto.Cipher.init(DashoA13*..)
	at org.apache.xml.security.encryption.XMLCipher.encryptData(XMLCipher.java:1137)
	... 57 more
```
The remedy to this: I downloaded the [Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files](http://www.oracle.com/technetwork/java/javase/downloads/jce-6-download-429243.html) and just followed the instructions in the README file contained in the downloaded zip. This boils down to replacing two magic jars in your JRE installation. It just sucks not to be able to deploy these along with your application.

## Useful links
- [http://www.ibm.com/developerworks/java/library/j-jws16/](http://www.ibm.com/developerworks/java/library/j-jws16/)


