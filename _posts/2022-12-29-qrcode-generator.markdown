---
layout: post
title: QR Generator in GO using Gin Web Framework
author: rmarcello
date: 2022-12-29 00:00:00 +0000
image: assets/images/qrcode-generator/main-image.png
categories: [Microservices, Cloud, Golang, Gin Web Framework, QRCode, Docker, Kubernetes]
comments: false
featured: true
---

In this short article I will talk about my experience in building a REST microservice using Golang. I've recently been using this programming language and I was curious to practically test it with the implementation of an entire project.
The QR-Generator is a REST service that allows you to generate simple QR Codes or with logo overlays. It is equipped with a REST API interface for generating QR codes and a UI useful for generating QR codes in a simple way.

For the realization I used the [Gin Web Framework](https://pkg.go.dev/github.com/gin-gonic/gin) and [go-qrcode](https://pkg.go.dev/github.com/skip2/go-qrcode) library in order to make it fast and lightweight.

The project is available on my personal github repo [qr-generator](https://github.com/marcelloraffaele/qr-generator).

# Microservice anatommy
## The API
QR-Generator is a small REST microservice that can be used to generate your QR-Code.
It is composed of two APIs respectively for the two types of QR codes:
- `/qrcode`: generates a simple black and white QR Code. This API can be used through `GET` method and the following parameters are required:
  - `txt`: represents the string that the user wants to encode;
  - `size`: represents the size of the encoded QR code express in pixels.
   
  The API returns a PNG file that can be saved in a file.
  
  An example of URL could the the following:
  ```
  curl "http://localhost:8080/qrcode?txt=https://marcelloraffaele.github.io&size=512"
    ```

- `/qrcode-overlay`: generates a QR Code with logo overlay. This API can be used through `POST` method and the following parameters are required:
  - `txt`: represents the string that the user wants to encode;
  - `size`: represents the size of the encoded QR code express in pixels.
  - `file`: represents the logo file in png format that will be overlayed on the encoded QR code.

    The API returns a PNG file that can be saved in a file.
    
    An example of URL could the the following:
    ```
    curl "http://localhost:8080/qrcode-overlay" \
        -X POST \
        -F txt=https://marcelloraffaele.github.io \
        -F size=512 \
        -F 'file=@"logo.png"'
    ```


## The UI
The user can use a simple UI to quicly interact with the API and to generate it's own QR code.
You can access the UI from a browser using the `http://<QR-Geerator-IP>:8080/ui` url. From here you can choose a QRCode type and generate it.

![UI Example](/assets/images/qrcode-generator/ui.png)

# How it's made?
The beating heart of the application is the Gin Web Framework that routes the HTTP requests.
The Gin web framework in addition to having excellent performance and small memory foot print, is equipped with:
- Middleware support: incoming HTTP request can be handled by a chain of middlewares;
- Crash-free: can catch a panic occurred during a HTTP request and recover it;
- JSON Validation: can parse and validate the JSON of a request;
- Error Management: provides a convenient way to collect all the errors occurred during a HTTP request;
- Rendering built-in: provides an easy to use API for JSON, XML and HTML rendering;
  
Last but not least the framework can be extended creating a new middleware.

The application is very simple to understand because the framework makes simple to define the entry point of our application:
```go
func main() {
	
	router := gin.Default()
	router.Static("/ui", "./ui")

	router.GET("/qrcode", func(c *gin.Context) {
		png, err := EncodeQRCode(c)
		if err != nil {
			c.AbortWithStatusJSON(500, gin.H{
				"error": err.Error(),
			})
			return
		}
		c.Data(200,"image/png", png)		
	})

	router.POST("/qrcode-overlay", func(c *gin.Context) {
		png, err := EncodeQRCodeOverlay(c)
		if err != nil {
			c.AbortWithStatusJSON(500, gin.H{
				"error": err.Error(),
			})
			return
		}
		c.Data(200,"image/png", png)		
	})
	router.Run("0.0.0.0:8080")	
}
```

As we can see, the `/ui` is defined as *static* content, the `/qrcode` is defined as GET and the `/qrcode-overlay` as POST methods.

The Gin Framework makes also the form Binding super simple and I used it to make a function that bind both the GET and POST APIs and to check their values.

```go
type QRCodeForm struct {
    Txt string `form:"txt"`
    Size int `form:"size"`
}

//...

func parseQRCodeRequest(c *gin.Context) (QRCodeForm, error) {
	var form QRCodeForm
	if err := c.ShouldBind(&form); err != nil {
		return form, err
	}	
	if len:= len(form.Txt); len < 3 {
		return form, fmt.Errorf("Txt length invalid %d should greater than 3", len)
	}
	if form.Size < 100 {
		return form, fmt.Errorf("Size value invalid %d should greater than 100", form.Size)
	}
	return form, nil
}
```

The remaining part is only a simple usage of the `go-qrcode`, `image/png` and `image/draw` libraries to encode the qr code and to overlay the logo on it.

if you are curious I invite you to view the code on the github repo [qr-generator](https://github.com/marcelloraffaele/qr-generator).

## Docker build
Another aspect I wanted to explore was the Docker build of the application and size of the resulting image.

As you can see from the following Dockerfile:

```Dockerfile
## Build
FROM golang:1.18-buster AS build
WORKDIR /
COPY go.mod ./
COPY go.sum ./
RUN go mod download
COPY *.go ./
RUN go build -o /app

## Deploy
FROM gcr.io/distroless/base-debian10
WORKDIR /
COPY --from=build /app /app
COPY /ui ./ui
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

I used a multistage build that gave me the opportunity to create very little images:

```shell
$ docker image ls
REPOSITORY                    TAG       IMAGE ID       CREATED         SIZE
rmarcello/qr-generator        latest    04ec2f6a52a0   22 hours ago    31.2MB
```




# Let's use it
The application can be executed building the source code or running a Docker image.

## Build and run locally
After a git clone, build the project using the following command:
```shell
go build -o app .
```
And run it locally:
```shell
./app
```

## Run with Docker
The image is already present on [Docker Hub](https://hub.docker.com/repository/docker/rmarcello/qr-generator).

To run with docker you can use this:
```shell
docker run -p 8080:8080 -it --rm --name qr-generator rmarcello/qr-generator:latest
```

## API Tests

If you need a simple QRCode you can use the `/qrcode` API using the GET method and the result can be written in a file:

Here's an example:

```shell
curl "http://localhost:8080/qrcode?txt=https://marcelloraffaele.github.io&size=512" \
    --output qrcode-without-logo.png
```

If you need a QR Code with logo overlay, you can use the `/qrcode-overlay` API using the POST method in order to upload the file that you want to use as logo. The image must be in PNG format, it will be resized and overlayed on the generated QR Code.
For the logo, I recommend using 150 x 150 px size. 

```shell
cd test
curl "http://localhost:8080/qrcode-overlay" \
    -X POST \
    -F txt=https://marcelloraffaele.github.io \
    -F size=512 \
    -F 'file=@"logo.png"' \
    --output qrcode-with-logo.png
```

# Conclusion
During the last few months I was actracted by the Golang programing language. I have seen a huge amount of project written in GO and for this reason I started to study it. But the best way to check the potentiality of a technology is to make your hands dirty using it.

During this experience I appreciated the peculiarities of Golang.
To list a few, I appreciated the compiler that actually manages dependencies on libraries and verifies that they are really needed. In Java, for example, it is necessary to use additional slow tools to manage dependencies.
The compiler enforces a quality programming style, which doesn't want unused variables or libraries.

Coming from Java I had no problem understanding the Golang systax and immediately felt at ease. I found many libraries all well documented and easy to use.

In addition, being a compiled language, Golang allows to create small images that use few resources and have excellent performance. In my opinion, these are some of the characteristics that make Go a successful language in the Cloud Native environment.
