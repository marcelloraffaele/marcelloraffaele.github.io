---
layout: post
title: JAVA ANNOTATION-BASED BYTE ARRAY PROTOCOL FRAMEWORK
author: rmarcello
date: 2020-04-16 00:00:00 +0000
image: assets/images/java-protocol-framework/java-byte-protocol-framework.png
categories: [java, bytearray, annotation, framework]
comments: false
---

It often happens that developers hate to make repetivive tasks and spend their precious time writing long, complicated and error prone pieces of code. <em>The more code you write the higher is the probability of errors.</em> What developers really love is to solve a standard problem only once and after <strong>reuse the solution</strong> wherever they want.

This is what I was thinking after one week writing the same repetitive code. I was writing hundreds of long and complex java classes used to marshall/unmarshall byte arrays to Java Object. Fortunately Protocol interfaces rarely changes and maybe there’s no need to spend time to create a common reusable solution that definitively solve the problem.
When create a library or a framework we must keep in mind that someone will believe on it and will trust his task. This means that we need to code it and also to test it well. For this reason not all the developers invest their time creating reuseble solutions and share it on Github.

I was almost giving up when I was inspired by Hibernate – one of the most famous open source Java Persistence framework project. I had a flashback and I saw something in common with my problem. Java Persistence maps java POJO beans to Database Table (not only…) and the underlying engine manage the conversion of each field from Java to Database and vice versa. Similarly, I could do the same mapping my POJO beans to byte array positions and creating an engine that converts every single field in byte array and viceversa. Exactly like Java Persistence I could use Java Annotations to define important metadata in each field that the engine will use to make the right mapping.

Off course, this project is still not a complete framework but it’s a feasibility exercise that can help me to show that it’s possible to define a set of annotations and develop an engine that enable developers to solve this kind of problems in a very easy, quick and safe way.


You can define simple POJO Beans using few simple Annotation to define the mapping:

{% highlight java %}
/**
 * SimpleProtocol is a simple text based protocol.
 * The filds list is the following:
 * type     String      2 bytes
 * sender   String      100 bytes
 * message  String      154 bytes
 * Each message is 256 bytes.
 */
@ProtocolEntity
public class SimpleProtocol {
    
    @ProtocolField(size=2)
    private String type;
    
    @ProtocolField(size=100)
    private String sender;
    
    @ProtocolField(size=154, filler = FillerType.RIGHT)
    private String message;
	
	//constructor, get and setter
}
{% endhighlight %}

Look this is the typical implementation that we do for Java Persistence!
The @ProtocolEntity says that this POJO Bean can be used for both parse from bytes and transformed into byte array.
The annotation @ProtocolField(size=2) tells to the engine how must be treated the fields, in this case like a String and it size in the buffer must be 2. Using the annotation we are able also to specify some extra information relative to text-alignment or bynary/decimal formatting.
{% highlight java %}
//String fill on left
@ProtocolField(size = 10, filler = FillerType.LEFT)

//String fill on right
@ProtocolField(size = 10, filler = FillerType.RIGHT)

//numeric conversion to string characters
@ProtocolField(size = 3, numericEncoding = NumericEncoding.TEXT)

//numeric conversion to binary representation
@ProtocolField(size = 3, numericEncoding = NumericEncoding.BYNARY)
{% endhighlight %}


But the best part is here, the conversion is very simple and it needs only one line of code:

{% highlight java %}
//parsing
SimpleProtocol simpleProtocol = engine.fromByte(sourceBuffer, SimpleProtocol.class); 

//covert to byte array
byte[] buffer = EngineFactory.create().toByte(simpleProtocol);
{% endhighlight %}

At the current version the framework supports the fallowing types:
<ol>
  <li>int/Integer</li>
  <li>short/Short</li>
  <li>byte/Byte/byte[]</li>
  <li>long/Long</li>
  <li>String</li>
</ol>

It can be extended to support also other generic types.
Another important feature is the support of inheritance. The developer can extend existing classes. The mapping works appending the fields to the super class fields.
In this way you can define header common part and separate Body entities will extend the header.

{% highlight java %}
/**
 * The Header is an abstract class that define the common attributes of the protocol header.
 * The filds list is the following:
 * version          String              2 bytes
 * sequenceNumber   Integer (bynary)    5 bytes
 * length           Integer (bynary)    3 bytes
 */
@ProtocolEntity
public abstract class HeaderProtocol {

    @ProtocolField(size=2, filler = FillerType.RIGHT )
    protected String version;

    @ProtocolField(size=5, numericEncoding = NumericEncoding.BINARY)
    protected Integer sequenceNumber;  
    
    @ProtocolField(size=3, numericEncoding = NumericEncoding.BINARY)
    protected Integer length;
    
    //constructor, get and setter
}


/**
 * The HeartBeatProtocol is an class that define HeartBeat protocol. It extends the header.
 * The filds list is the following:
 * timestamp   Long (bynary)    10 bytes
 * crc     byte (bynary)    3 bytes
 * 
 */
public class HeartBeatProtocol extends HeaderProtocol {

    @ProtocolField(size = 10, numericEncoding = NumericEncoding.BINARY)
    private Long timestamp;

    @ProtocolField(size = 3)
	private byte[] crc;
    
    //constructor, get and setter

}
{% endhighlight %}

The engine invocation doesn't change:

{% highlight java %}
//create an Engine
Engine engine = EngineFactory.create();

//parsing
HeartBeatProtocol heartBeatParsed = engine.fromByte(heartBeatByte, HeartBeatProtocol.class);

//covert to byte array
byte[] heartBeatByte = engine.toByte(heartBeat);

{% endhighlight %}


If interested, you can find my project on github:
<a href="https://github.com/marcelloraffaele/positional-protocol">https://github.com/marcelloraffaele/positional-protocol</a>
If you appreciate my idea or want more information feel free to contact me.