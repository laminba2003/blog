---
title: "Getting Started with GServlet"
date: 2021-02-06T16:50:56+01:00
description : "GServlet is an open source project inspired from the Groovlets, which aims to use the Groovy language and its provided modules to simplify Servlet API web development."
draft: false
tags : [
    "JAVA" , "SERVLET", "GROOVY"
]
series : [
    "Groovify your Java Servlets"
]
image : "https://mamadoulamineba.netlify.com/images/gservlet.png"
author : "Mamadou Lamine Ba"
---

[GServlet](https://gservlet.org) is an open source project inspired from the [Groovlets](http://docs.groovy-lang.org/latest/html/documentation/servlet-userguide.html), which aims to use the Groovy language and its provided modules to simplify Servlet API web development.
Groovlets are Groovy scripts executed by a servlet. They are run on request, having the whole web context (request, response, etc.) bound to the evaluation context. They are much more suitable for smaller web applications. 
Compared to Java Servlets, coding in Groovy can be much simpler. It has a couple of implicit variables we can use, for example, _request_, _response_ to access the [HttpServletRequest](https://javaee.github.io/javaee-spec/javadocs/javax/servlet/http/HttpServletRequest.html), and [HttpServletResponse](https://javaee.github.io/javaee-spec/javadocs/javax/servlet/http/HttpServletResponse.html) objects. We have access to the [HttpSession](https://javaee.github.io/javaee-spec/javadocs/javax/servlet/http/HttpSession.html) with the _session_ variable. If we want to output data, we can use _out_, _sout_, and _html_. This is more like a script as it does not have a class wrapper.

### Groovlet 

{{< highlight java>}}
if (!session) {
    session = request.getSession(true)
}
if (!session.counter) {
    session.counter = 1
}
html.html {
  head {
      title('Groovy Servlet')
  }
  body {
    p("Hello, ${request.remoteHost}: ${session.counter}! ${new Date()}")
  }
}
session.counter = session.counter + 1
{{< / highlight >}}

The same philosophy has been followed in the design of this API while maintaining a class-based programming approach.

### SessionCounterServlet.groovy
 
{{< highlight java>}}
import org.gservlet.annotation.Servlet

@Servlet("/counter")
class SessionCounterServlet {

  void get() {
    if (!session.counter) {
      session.counter = 1
    }
    html.html {
      head {
        title('Groovy Servlet')
      }
      body {
        p("Hello, ${request.remoteHost}: ${session.counter}! ${new Date()}")
      }
    }
    session.counter = session.counter + 1
  }

}
{{< / highlight >}}

## Main Features

* Servlet 3.1+ Support
* Groovy Scripting and Hot Reloading
* JSON, XML, HTML and JDBC Support

## Requirements

* Java 8+
* Java IDE (Eclipse, IntelliJ IDEA, NetBeans..)
* Java EE 7+ compliant WebServer (Tomcat, Wildfly, Glassfish, Payara..)

Five steps take you from writing your first Groovy servlet to running it. Using Maven, these steps are as follows:

1. Create a Maven webapp project
2. Create the groovy folder inside your webapp directory
3. Write the servlet source code
4. Run your Java web server
5. Call your servlet from a web browser

As an introduction, today we are going to rewrite the classic Java Hello World Servlet below to make you perceive the difference in terms of simplicity and clarity.

### HelloWorldServlet.java

{{< highlight java>}}
import javax.servlet.annotation.WebServlet;
import java.io.IOException;
import java.io.PrintWriter;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@WebServlet("/index.html")
public class HelloWorldServlet extends HttpServlet {

	public void doGet(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		response.setContentType("text/html");
		PrintWriter out = response.getWriter();
		out.println("<html><body>");
		out.println("Hello World!");
		out.println("</body></html>");
		out.close();
	}
  
}
{{< / highlight >}}

Let's start by setting things up.

### pom.xml

{{< highlight xml>}}

<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>gservlet-examples</groupId>
	<artifactId>hello-servlet</artifactId>
	<version>1.0.0</version>
	<packaging>war</packaging>
	<dependencies>
		<dependency>
			<groupId>org.gservlet</groupId>
			<artifactId>gservlet-api</artifactId>
			<version>1.0.0</version>
		</dependency>
	</dependencies>
	<build>
		<plugins>
			<plugin>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>3.8.0</version>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
				</configuration>
			</plugin>
			<plugin>
				<artifactId>maven-war-plugin</artifactId>
				<version>3.2.1</version>
				<configuration>
					<failOnMissingWebXml>false</failOnMissingWebXml>
				</configuration>
			</plugin>
		</plugins>
	</build>
</project>

{{< / highlight >}}

### HelloWorldServlet.groovy

{{< highlight java>}}
import org.gservlet.annotation.Servlet

@Servlet("/index.html")
class HelloWordServlet {

  void get() {
     html.html {
       body {
         p("Hello World!")
       }
     } 
  }
   
}
{{< / highlight >}}

### Generated HTML

{{< highlight html>}}

<!DOCTYPE html>
<html>
  <body>
    <p>Hello World!</p>
  </body>
</html>

{{< / highlight >}}

That's all for now and in the next articles, we will cover more ground and how this API can be even integrated in your SpringBoot applications. This example is available for download on [GitHub](https://github.com/GServlet/gservlet-examples/tree/1.0.0-examples/hello-servlet) and you can read as well the [documentation](https://gservlet.org/docs/1.0.0/) for in-depth knowledge of this project. 

[GServlet](https://gservlet.org) will start to support JakartaEE 9 in its 2.0.0 version. You can access snapshot builds from the [sonatype](https://oss.sonatype.org/#nexus-search;quick~gservlet) snapshot repository