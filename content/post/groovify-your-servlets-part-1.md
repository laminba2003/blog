---
title: "Groovify Your Java Servlets (Part 1) : Getting Started"
date: 2019-10-28T16:50:56+01:00
description : "In this first part of this series, we are going to set the building blocks of how to enhance the Servlet API using Groovy."
draft: false
tags : [
    "GROOVY" , "JAVA" , "SERVLET"
]
series : [
    "Groovify your Java Servlets"
]
image : "https://mamadoulamineba.netlify.com/images/handshake.png"
---

This blog post is not about [Groovlets](http://docs.groovy-lang.org/latest/html/documentation/servlet-userguide.html), which are Groovy scripts executed by a servlet. They are run on request, having the whole web context (request, response, etc.) bound to the evaluation context. Groovlets are much more suitable for smaller web applications. Compared to Java Servlets, coding in Groovy can be much simpler.

This post provides a simple example demonstrating the kinds of things you can do with a Groovlet. It has a couple of implicit variables we can use, for example, _request_, _response_ to access the _HttpServletRequest_, and _HttpServletResponse_ objects. We have access to the _HttpSession_ with the session variable. If we want to output data, we can use _out_, _sout_, and _html_. This is more like a script as it does not have a class wrapper. 


### Groovlet 

{{< highlight java>}}

if (!session) {
    session = request.getSession(true)
}
if (!session.counter) {
    session.counter = 1
}
html.html { // html is implicitly bound to new MarkupBuilder(out)
  head {
      title('Groovy Servlet')
  }
  body {
    p("Hello, ${request.remoteHost}: ${session.counter}! ${new Date()}")
  }
}
session.counter = session.counter + 1
{{< / highlight >}}


In this first part of this series, we are going to set the building blocks of how to bring the same logic to enhance the Servlet API. The goal is to write our artifacts (servlets, filters, listeners) with the Groovy language, which is in constant evolution in the Tiobe Index of language popularity, coming in this month at No. 11. 

Next, we will register them in the ServletContext class. We will consider a live development as a mandatory feature, even if it is well-known that the _ServletContext_ class will throw an _IllegalStateException_ when adding a new Servlet, Filter, or Listener after its initialization. Also, updating the artifacts at runtime is another challenging task.

To lay things down, I will first uncover the final design. In the next blog posts, I will cover more ground to show you, in detail, how things are achieved technically. The main vision is to use the Groovy language and its provided modules (JSON, SQL, etc.) to simplify Servlet API web development while waiting to apply the same principles to the JAX-RS API.

It is worth it to mention that we are going to build a non-intrusive API that will not affect the current structure of your Java EE project, and you are free to groovify your existing Java Servlets over time. Let's get started!


### New Servlet Signature 

{{< highlight java>}}

import org.gservlet.annotation.Servlet

@Servlet("/customers")
class CustomerServlet {

    void get() {
      def customers = []
      customers << [firstName : "John", lastName : "Doe"]
      customers << [firstName : "Kate", lastName : "Martinez"]
      customers << [firstName : "Allisson", lastName : "Becker"]
      json(customers)
    }
    
    void post() {
      def customer = request.body // get the json request payload as object 
      json(customer)
    }
    
    void put() {
      def customer = request.body // get the json request payload as object
      json(customer)
    }
    
    void delete() {
      def param = request.param // shortcut to request.getParameter("param")
      def attribute = request.attribute // shortcut to request.getAttribute("attribute")
    }
    
    void head() {
    }
    
    void trace() {
    }
    
    void options() {
    }
    
}

{{< / highlight >}}


The _@WebServlet_ annotation, which is used to define a Servlet component in a web application, will be reduced to our own _@Servlet_ annotation with the same attributes. In the same spirit, and as read above, the name of the HTTP request method handlers (_doGet_, _doPost_, _doPut_, and so on) will be shortened and they will take no arguments since the request and the response are now implicit variables.

### New Servlet Signature 

{{< highlight java>}}

package org.gservlet.annotation;

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface Servlet {
    String name() default "";
    String[] value() default {};
    String[] urlPatterns() default {};
    int loadOnStartup() default -1;
    WebInitParam [] initParams() default {};
    boolean asyncSupported() default false;
    String smallIcon() default "";
    String largeIcon() default "";
    String description() default "";
    String displayName() default "";
}

{{< / highlight >}}


We will no longer explicitly extend the _HttpServlet_ class. When a script is loaded by the GroovyScriptEngine, we will use _javassist_, a library for dealing with Java bytecode, to extend dynamically our own derived _HttpServlet_ class within a BytecodeProcessor instance, which is set anonymously to the Groovy _CompilerConfiguration_ object.  

### New Servlet Signature 

{{< highlight java>}}

package org.gservlet;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpServlet;

public abstract class AbstractServlet extends HttpServlet {
  
  @Override
  public void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException {
     service(request, response, "get");
  }
  
  @Override
  public void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException {
     service(request, response, "post");
  }
  
  @Override
  public void doPut(HttpServletRequest request, HttpServletResponse response) throws ServletException {
     service(request, response, "put");
  }
  
  @Override
  public void doDelete(HttpServletRequest request, HttpServletResponse response) throws ServletException {
     service(request, response, "delete");
  }
  
  @Override
  public void doHead(HttpServletRequest request, HttpServletResponse response) throws ServletException {
     service(request, response, "head");
  }
  
  @Override
  public void doTrace(HttpServletRequest request, HttpServletResponse response) throws ServletException {
     service(request, response, "trace");
  }
  
  @Override
  public void doOptions(HttpServletRequest request, HttpServletResponse response) throws ServletException {
     service(request, response, "options");
  }
  
}

{{< / highlight >}}


The code of the _HttpServlet_ base class above is not complete. I'm going to wait for the next blog posts to show you how we are going to make the _HttpServletRequest_, the _HttpServletResponse_, and so on, to become implicit variables.

As stated in the style guide, in Groovy, a getter and a setter form what we call a property. Instead of the Java-way of calling getters/setters, we can use a field-like access notation for accessing and setting such properties, but there is more to write about.


To route an HTTP request to our new handler methods (get, post, put, and so on), we will register our servlets in the ServletContext as dynamic proxies with an invocation handler instance. A dynamic proxy can be thought of as a kind of Facade as it allows one single class with one single method to service multiple method calls to arbitrary classes with an arbitrary number of methods. When a method is invoked on a proxy instance, the method invocation is encoded and dispatched to the invoke method of its invocation handler. 

### InvocationHandler class

{{< highlight java>}}

package org.gservlet;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class DynamicInvocationHandler implements InvocationHandler {
    protected Object target;
    
    public DynamicInvocationHandler(Object target) {
      this.target = target;
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      return method.invoke(target, args);
    }
    
    public Object getTarget() {
      return target;
    }
    
    public void setTarget(Object target) {
        this.target = target;
    }
    
}

{{< / highlight >}}


### Servlet Registration

{{< highlight java>}}

protected void addServlet(ServletContext context, Servlet annotation, Object object) throws ServletException {
		String name = annotation.name().trim().equals("") ? object.getClass().getName() : annotation.name();
		ServletRegistration registration = context.getServletRegistration(name);
		if (registration == null) {
			DynamicInvocationHandler handler = new DynamicInvocationHandler(object);
			Object servlet = Proxy.newProxyInstance(this.getClass().getClassLoader(),
					new Class[] { javax.servlet.Servlet.class }, handler);
			handlers.put(name, handler);
			registration = context.addServlet(name, (javax.servlet.Servlet) servlet);
			if (annotation.value().length > 0) {
				registration.addMapping(annotation.value());
			}
			if (annotation.urlPatterns().length > 0) {
				registration.addMapping(annotation.urlPatterns());
			}
			for (InitParam param : annotation.initParams()) {
				registration.setInitParameter(param.name(), param.value());
			}
		} else {
			String message = "The servlet with the name " + name
					+ " has already been registered. Please use a different name or package";
			throw new ServletException(message);
		}
	}

{{< / highlight >}}

To reload an artifact means to update the corresponding target object of the proxy's invocation handler once our file watcher can detect a file change.

Like for a Listener, the same design is applied to a Filter and we will extend another base class dynamically at runtime. To perceive the difference in terms of simplicity and clarity, the Groovy class above is a complete rewriting of the Java CORS Filter example published at the HowToDoInJava website. Below is the Java Filter base class used to achieve such transformation through inheritance.

{{< highlight java>}}

import org.gservlet.annotation.Filter

@Filter("/*")
class CORSFilter {
  void filter() {
    response.addHeader("Access-Control-Allow-Origin", "*")
    response.addHeader("Access-Control-Allow-Methods","GET, OPTIONS, HEAD, PUT, POST, DELETE")
    if (request.method == "OPTIONS") {
       response.status = response.SC_ACCEPTED
       return
    }
    next()
  }
}

{{< / highlight >}}


Without an explicit inheritance, we might think that we will lose the benefits of code completion in our IDEs but such is not the case since the Groovy language is also an excellent platform for the easy creation of domain-specific languages (DSLs). Using a DSLD, which is a DSL descriptor, it is possible to teach the editor some of the semantics behind these custom DSLs. For a while now, it has been possible to write an Eclipse plugin to extend Groovy-Eclipse, which requires specific knowledge of the Eclipse APIs. This is no longer necessary.

### DSLD (DSL Descriptor)

{{< highlight java>}}

//@org.gservlet.AbstractServlet

contribute(currentType(annos: annotatedBy(Servlet))) {
	property name : 'logger', type : java.util.Logger, provider : 'org.gservlet.AbstractServlet', doc : 'the logger'
	property name : 'request', type : javax.servlet.http.HttpServletRequest, provider : 'org.gservlet.AbstractServlet', doc : 'the http request'
	property name : 'response', type : javax.servlet.http.HttpServletResponse, provider : 'org.gservlet.AbstractServlet', doc : 'the http response'
	property name : 'session', type : javax.servlet.http.HttpSession, provider : 'org.gservlet.AbstractServlet', doc : 'the http session'
	property name : 'context', type : javax.servlet.ServletContext, provider : 'org.gservlet.AbstractServlet', doc : 'the servlet context'
	property name : 'connection', type : groovy.sql.Sql, provider : 'org.gservlet.AbstractServlet', doc : 'the sql connection'
	property name : 'out', type : java.io.PrintWriter, provider : 'org.gservlet.AbstractServlet', doc : 'the response writer'
	property name : 'html', type : groovy.xml.MarkupBuilder, provider : 'org.gservlet.AbstractServlet', doc : 'the markup builder'
}

contribute(currentType(annos: annotatedBy(Servlet))) {
	delegatesTo type : org.gservlet.AbstractServlet, except : [
		'service',
		'doGet',
		'doPost',
		'doHead',
		'doPut',
		'doTrace',
		'doOptions',
		'doDelete'
	]
}

//@org.gservlet.AbstractFilter

contribute(currentType(annos: annotatedBy(Filter))) {
	property name : 'logger', type : java.util.Logger, provider : 'org.gservlet.AbstractFilter', doc : 'the logger'
	property name : 'request', type : javax.servlet.http.HttpServletRequest, provider : 'org.gservlet.AbstractFilter', doc : 'the http request'
	property name : 'response', type : javax.servlet.http.HttpServletResponse, provider : 'org.gservlet.AbstractFilter', doc : 'the http response'
	property name : 'session', type : javax.servlet.http.HttpSession, provider : 'org.gservlet.AbstractFilter', doc : 'the http session'
	property name : 'context', type : javax.servlet.ServletContext, provider : 'org.gservlet.AbstractFilter', doc : 'the servlet context'
	property name : 'connection', type : groovy.sql.Sql, provider : 'org.gservlet.AbstractFilter', doc : 'the sql connection'
	property name : 'out', type : java.io.PrintWriter, provider : 'org.gservlet.AbstractFilter', doc : 'the response writer'
	property name : 'html', type : groovy.xml.MarkupBuilder, provider : 'org.gservlet.AbstractFilter', doc : 'the markup builder'
}

contribute(currentType(annos: annotatedBy(Filter))) {
	delegatesTo type : org.gservlet.AbstractFilter, except : ['init', 'doFilter']
}


{{< / highlight >}}
