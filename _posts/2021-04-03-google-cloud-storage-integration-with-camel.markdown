---
layout: post
title: Google Cloud Storage integration with Apache Camel
date: 2021-04-03 00:00:00 +0000
description: Google Cloud Storage integration with Apache Camel
img: google-cloud-storage-camel/google-cloud-storage-camel.png
fig-caption: google-cloud-storage-camel/google-cloud-storage-camel.png
tags: [google, cloud, storage, camel, integration, java]
---
The rising adoption of Cloud services need also to integrate this features with new and existing application.
Apache Camel is an Open Source integration framework that empowers developers to quickly and easily integrate various systems consuming or producing data. It is widely used because is a small library that has minimal dependencies for easy embedding in java application. For more information about Camel and it's documentation visit [camel.apache.org](https://camel.apache.org).

I am Camel fan, I have used it in many projects, and now I decided to start to contribute on it on github. As start, I decided to help to write new missing components and my first official contribution is the development of the Google Cloud Storage component. After some work, tests and pull requests I am proud to announce that from the [version 3.9.0](https://camel.apache.org/blog/2021/03/Camel39-Whatsnew/) Camel comes with the Google Cloud Storage component. It can be used to quickly integrate your java applications with Google Cloud Storage buckets.
Let's see how to use it practically.

# The Google Cloud Storage Component
Before to start with the example, let's introduce the component. The Google Cloud Storage Component has the following endpoint:
{% highlight java %}
google-storage://bucketName?options=value&option2=value&…
{% endhighlight %}

It supports both producer and consumer.

Behind the scenes, the component uses a Google Storage Client to interact with the Cloud. The authentication is targeted for use with the GCP Service Accounts. For more information please refer to [Google Storage Auth Guide](https://cloud.google.com/storage/docs/reference/libraries#setting_up_authentication).
Once generated the service account key you can provide authentication credentials to your application code using two ways:
- Directly by the endpoint:
{% highlight java %}
String endpoint = "google-storage://myCamelBucket?serviceAccountKey=/home/user/Downloads/my-key.json";
{% endhighlight %}
- Or by setting the environment variable GOOGLE_APPLICATION_CREDENTIALS:
{% highlight shell %}
export GOOGLE_APPLICATION_CREDENTIALS="/home/user/Downloads/my-key.json"
{% endhighlight %}

## The consumer
The consumer can be used to read the objects inside the bucket and to send it to a destination. The consumer is a ScheduledBatchPollingConsumer this means that implements polling on the bucket.
To prevent objects from being reprocessed, the consumer can be configured to automatically move the processed object to another bucket. For this reason is possible to use the options:
- moveAfterRead
- destinationBucket
- deleteAfterRead. 

{% highlight java %}
from("google-storage://myCamel?serviceAccountKey=/home/user/Downloads/my-key.json&moveAfterRead=true&destinationBucket=processed_bucket&autoCreateBucket=true&deleteAfterRead=true")
    .log("consuming: ${header.CamelGoogleCloudStorageBucketName}/${header.CamelGoogleCloudStorageObjectName}")
    .to("direct:processed");
{% endhighlight %}

Notice that in this example, bucket names are not real names, if you use it can fail because bucket names are global.

## The producer

The Google Storage component provides the following operation on the producer side:
- copyObject
- listObjects
- deleteObject
- deleteBucket
- listBuckets
- getObject
- createDownloadLink

To upload a file to a bucket, we can define a route like this:

{% highlight java %}
//upload a file
byte[] payload = "Camel rocks!".getBytes();
ByteArrayInputStream bais = new ByteArrayInputStream(payload);
from("direct:start")
.process( exchange -> {
    exchange.getIn().setHeader(GoogleCloudStorageConstants.OBJECT_NAME, "camel.txt");
    exchange.getIn().setBody(bais);
})
.to("google-storage://myCamelBucket?serviceAccountKey=/home/user/Downloads/my-key.json")
.log("uploaded file object:${header.CamelGoogleCloudStorageObjectName}, body:${body}");
{% endhighlight %}

For many usage examples visit [Camel google storage component documentation](https://camel.apache.org/components/latest/google-storage-component.html#_google_storage_producer_operations).

Now that we know how to use the component, let's use it for a real example.

# Use case: Thumbnail creator
The use case will show how is possible to automate the thumbnail creation of some images.
Suppose that we have a "source-bucket" where we put some images and that we want to automatically create and store the relative thumbnail in another bucket.

We can define a route where the "source-bucket" can be the starting point. A google storage camel consumer will poll the source-bucket verifying if there are unprocessed images. For each new file, It will load the image and move the file to the "consumed-bucket" without any modification. The loaded image will be processed by "thumbnail-processor" that implements the logic to resize the image. Later the resulting image will be sent to a google storage producer that will write the image to a "result-bucket". At the end another producer will make a download URL for the object.

The use case can be represented by the following schema:
![camel-route]({{site.baseurl}}/assets/img/google-cloud-storage-camel/camel-route.png)

## Developement details: Camel main
All the source code will be available on my github from the following link: [Camel experiments](https://github.com/marcelloraffaele/camel-experiments).

The quickest way to start is to use the Camel Maven Archetypes to create a new maven project for running camel standalone:
{% highlight java %}
mvn archetype:generate -DarchetypeGroupId=org.apache.camel.archetypes -DarchetypeArtifactId=camel-archetype-main -DarchetypeVersion=3.9.0 -DgroupId=com.rmarcello.camelexperiment.googlestoragetest -DartifactId=camel-google-storage-test -Dversion=1.0.0-SNAPSHOT
{% endhighlight %}

From here is needed to setup our pom.xml file, adding the dependency to the camel google storage component:
{% highlight xml %}
<dependency>
    <groupId>org.apache.camel</groupId>
    <artifactId>camel-google-storage</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.camel</groupId>
    <artifactId>camel-direct</artifactId>
</dependency>
<dependency>
    <groupId>net.coobird</groupId>
    <artifactId>thumbnailator</artifactId>
    <version>[0.4, 0.5)</version>
</dependency>
{% endhighlight %}
Notice that I added also the dependency for the [thumbnailator library](https://github.com/coobird/thumbnailator) that will be used to implement the processor that make the thumbnails. 

After that the route can be defined. I decided to use the "direct" camel component to split the route in pieces to make it more readable and debuggable:
{% highlight java %}
package com.rmarcello.camelexperiment.googlestoragetest;

import org.apache.camel.builder.RouteBuilder;
import org.apache.camel.component.google.storage.GoogleCloudStorageConstants;
import org.apache.camel.component.google.storage.GoogleCloudStorageOperations;
import net.coobird.thumbnailator.Thumbnails;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import com.google.cloud.storage.Blob;

public class MyRouteBuilder extends RouteBuilder {

    final String sourceBucket = "sourcebucket576832";
    final String consumedBucket = "consumedbucket576832";
    final String resultBucket = "resultbucket576832";

    @Override
    public void configure() throws Exception {
        
        from("google-storage://" + sourceBucket
        + "?moveAfterRead=true"
        + "&destinationBucket=" + consumedBucket
        + "&autoCreateBucket=true"
        + "&deleteAfterRead=true"
        + "&includeBody=true"
        )
        .log("consumed file: ${header.CamelGoogleCloudStorageObjectName}")
        .to("direct:consumed");
        
        from("direct:consumed")
        .process( exchange -> {
            byte[] body = exchange.getIn().getBody(byte[].class);
            ByteArrayInputStream bias = new ByteArrayInputStream( body );
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            Thumbnails.of( bias )
                .size(50, 50)
                .toOutputStream(baos);
            exchange.getIn().setBody( baos.toByteArray() );
        })
        .to("direct:processed");

        from("direct:processed")
        .to("google-storage://" + resultBucket )
        .log("uploaded file object:${header.CamelGoogleCloudStorageObjectName} to ${header.CamelGoogleCloudStorageBucketName} bucket")
        .to("direct:result");
        

        from("direct:result")
        .process( exchange -> {
            Blob blob = exchange.getIn().getBody(Blob.class);
            exchange.getIn().setHeader(GoogleCloudStorageConstants.OBJECT_NAME, blob.getName() );
            exchange.getIn().setHeader(GoogleCloudStorageConstants.OPERATION, GoogleCloudStorageOperations.createDownloadLink);
            exchange.getIn().setHeader(GoogleCloudStorageConstants.DOWNLOAD_LINK_EXPIRATION_TIME, 600000L); //10 minutes
        })
        .to("google-storage://" + resultBucket )
        .log("URL for ${header.CamelGoogleCloudStorageBucketName}/${header.CamelGoogleCloudStorageObjectName}=${body}");
    }
}
{% endhighlight %}

Notice that the route is very simple and reflect the previous diagram. I also added some logs to help understanding what happens during the execution.

To execute it, is possible to execute the following commands:
{% highlight shell %}
$env:GOOGLE_APPLICATION_CREDENTIALS = 'C:\keys\my-key.json'
mvn camel:run
{% endhighlight %}

When the application starts, the Camel engine creates the routes and the consumer component waits to find a new file to consume.
{% highlight shell %}
AbstractCamelContext           INFO  Routes startup summary (total:4 started:4)
AbstractCamelContext           INFO      Started route1 (google-storage://sourcebucket576832)
AbstractCamelContext           INFO      Started route2 (direct://consumed)
AbstractCamelContext           INFO      Started route3 (direct://processed)
AbstractCamelContext           INFO      Started route4 (direct://result)
AbstractCamelContext           INFO  Apache Camel 3.9.0 (MyTimerCamel) started in 3s693ms (build:46ms init:309ms start:3s338ms)
{% endhighlight %}

Notice that automatically (by default, you can configure it ) if not present the component creates the buckets:

![bucket-list]({{site.baseurl}}/assets/img/google-cloud-storage-camel/bucket-list.png)


When a new file is available into the bucket:

![added-file]({{site.baseurl}}/assets/img/google-cloud-storage-camel/added-file.png)


the component consumes the file and moves the data though the routes, processes the image, save it inside the result bucket and creates a URL:
{% highlight shell %}
route1                         INFO  consumed file: hokusai.png
route3                         INFO  uploaded file object:hokusai.png to sourcebucket576832 bucket
route4                         INFO  URL for sourcebucket576832/hokusai.png=https://storage.googleapis.com/resultbucket576832/hokusai.png?GoogleAccessId=myjavaclient@functionexampleproject.iam.gserviceaccount.com&Expires=1617347431&Signature=r4a5KveTVOTemp4iP5CY%2B6D5hRnkLA8lGnBMt6uUY9cfLYxyoBJte8nyH6x%2F57wbiRCfktMjAJXIIqB1fyvH3Vc1r98nUuWBBcWvUO6uvlz2KePUMK8C3gNi7HY3cVDqNRTEuTIST4RjTKgj%2FJBnY6O1o4Es0r%2B9ElcdAwRRtelqaz3wg4Gv4vsEHcyBsflB%2BQZ2pqWw25ET%2B7IkNBT5AbfpvIpHfur%2F19Rx4L1HOq2TH%2FGYOcYZDWlcn7W0lVa9%2F3yZQHZ3p%2FPKGwW88J9pgmJIki7MCeyjSCmS%2BNoGjh9q10CyuqzcU78x87qCxYxemoU35cU008KjPEUdkthOkQ%3D%3D
{% endhighlight %}

Let's show how the buckets changed:

![post-execution]({{site.baseurl}}/assets/img/google-cloud-storage-camel/post-execution.png)


Notice that the source bucket is empty, the consumed bucket contains the original images and the result bucket contains the thumbnail. In addition, the original image was 1.3MB and the resulting thumbnail is 4KB. So, mission completed!!!

![hokusai-diff]({{site.baseurl}}/assets/img/google-cloud-storage-camel/hokusai-diff.png)



# Conclusion
This article wants to quickly explain how to use the camel google storage component for a simple use case but also to demonstrate the power of opensource software and contribution.
I am proud to have made my contribution to the Camel project. By developing this component I was able to solve my need (to integrate my applications with Google Cloud Storage) but also to share my work with the community that will be able to reuse the component and maybe, why not, even improve it. I also had the opportunity to study more closely the structure of the project, the best practices and standards they adopt and collaborate with the people involved in developing the project. I think that is not possible to study these things from books but it's only possible though experience. For this reason I highly recommend collaborating in opensource projects.

# Resources
If you want to learn more:

1. [camel.apache.org](https://camel.apache.org)
1. [Camel books](https://camel.apache.org/manual/latest/books.html)
1. [Contributing to Apache Camel](https://camel.apache.org/manual/latest/contributing.html)
1. [Thumbnailator library](https://github.com/coobird/thumbnailator)

