---
layout: post
title: JAVA ANNOTATION-BASED BYTE ARRAY PROTOCOL FRAMEWORK
date: 2016-07-09 00:00:00 +0000
description: Create a Java Annotation-based byte array protocol framework
img: telegram-bot-php/telegram-bot-php.png
fig-caption: telegram-bot-php/telegram-bot-php.png
tags: [java, bytearray, annotation, framework]
---

The best way to invent something is to copy the original idea from another project. It often happens that developers hate to make repetivive tasks and spend their days writing long and complicated pieces of code. The more code you write the higher is the probability of errors. What developers really love is to solve a standard problem only once and after reuse the solution wherever they want.

This is what I was thinking every day after long coding session. I was writing hundreds of long and complex java classes used to marshall/unmarshall of positional based byte stream protocol. Fortunately Protocol interfaces rarely changes and maybe there’s no need to create a common reusable solution that require a development effort but mainly a huge test work. But why cannot I use a common solution that can help me in the future?

After some weeks I was inspired by Hibernate – one of the most famous open source Java Persistence framework project. I had a flashback and I saw something in common with my problem. Java Persistence maps java POJO beans to Database Table (not only…) and the underlying engine manage the conversion of each field from Java to Database and vice versa. Similarly, I could do the same mapping my POJO beans to byte array positions and creating an engine that converts every single field in byte array and viceversa. Exactly like Java Persistence I could use Java Annotations to define important metadata in each field that the engine will use to make the right mapping.

This project is still not a complete framework but it’s a feasibility exercise that can help me to show that it’s possible to define a set of annotations and develop an engine that enable developers to solve this kind of problems in a very easy, quick and safe way.


Here an example of bean definition and engine invocation:

{% highlight java %}
@BufferOut
public class ExampleBean {
 
   @ProtocolField(size=4)
   private String type;
 
   @ProtocolField(size=3)
   private Integer version;
 
   @ProtocolField(size=6)
   private String name;
 
   @ProtocolField(size=10)
   private String description;
 
   //…
 
}
{% endhighlight %}

Look this is the typical implementation that we do for Java Persistence!

Where @BufferIn says that the bean can be used to parse from byte the bean and @BufferOut that the bean can be transformed in byte array.

The annotation @ProtocolField(size=10) says that the field must be treated by the engine and it size in the buffer must be 10.The annotation can be extended and for example it can define also some unrequired information (for example text-alignment or decimal formatting).

But the best part is here, the conversion is very simple and it needs only one line of code:

{% highlight java %}
ExampleBean bean = (ExampleBean) Engine.fromBytes( byteArray, ExampleBean.class );
 
//converts to byte array
byte[] res = Engine.toBytes( bean );
{% endhighlight %}

You can find my project on github: https://github.com/marcelloraffaele/positional-protocol If you appreciate my idea and want more information feel free to contact me.