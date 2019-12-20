---
title: "Groovify Your Java Servlets (Part 2): Scripting the JVM"
date: 2019-11-25T17:49:37+01:00
description : "The purpose of this blog post is to show you how you can embed the Groovy Script Engine in a generic way whether you are building a Java console or web application"
draft: false
tags : [
    "GROOVY" , "JAVA" , "SERVLET"
]
series : [
    "Groovify your Java Servlets"
]
image : "https://mamadoulamineba.netlify.com/images/hands.png"
author : "Mamadou Lamine Ba"
---

I will start the second part of this series with a very simple answer to the question: Why Groovy over Java? Replacing Java has never been my intent, and I have never created a project using Groovy alone.

Over the years, I have learned that we must use the right language for the job, and if we take an example over **SoapUI**, which is essentially a Java automation tool, a Groovy script gives you a free hand for creating dynamic mock response content, for tweaking the test cases, and exploring the possibilities of testing your web service using various test scenarios without restarting your Java web server.

So I've been wondering for a while: Why can't we follow the same philosophy to take advantage of both languages since they interact quite well with one another? And on the other hand, Groovy can solve a lot of problems with less code than Java, producing more readable and compact solutions.

Therefore, the purpose of this blog post is to show you how you can embed the Groovy Script Engine in a generic way whether you are building a Java console or web application.

I will start with a brief presentation of the JSR 223, the Java Scripting API, which has been withdrawn in December 2016, following its maintenance review, where it was decided that this functionality would be included as an integral part of Java 9 and onward. 

It was aiming to let you embed scripts written with the language of your choice.

### Scripting With the JSR 223

The Java Scripting API is a set of classes and interfaces stored in the relatively small and simple javax.script package. The  _ScriptEngineManager_ class is the entry point to discover script engines through the JAR file service discovery mechanism, and to instantiate ScriptEngine objects that interpret scripts written in a specific scripting language. If we use this API, this is how a Groovy script must be written if we want to create a Java object from it.  

#### MyScript.groovy

{{< highlight java>}}

class MyScript {

   void sayHello() {
     println "Hello world"
   }
   
}

new MyScript()

{{< / highlight >}}



#### App.java

{{< highlight java>}}

import javax.script.*;
import java.io.FileReader;

public class App {

    public static void main(String[] args) {
        ScriptEngineManager manager = new ScriptEngineManager();
        ScriptEngine engine = manager.getEngineByName("groovy");
        Object object = engine.eval(new FileReader("MyScript.groovy"));
        Method method = object.getClass().getDeclaredMethod("sayHello");
        method.invoke(object);
    }
    
}

{{< / highlight >}}


You may notice that we are using the reflection API to invoke the sayHello method on the object. The reason is simple. Our Groovy script does not implement a Java interface or extend an abstract class so we can perform a cast.

The Java Scripting API provides also a _Compilable_ interface for the engines that support compilation so you can precompile your scripts before execution for a gain of performance. 

{{< highlight java>}}

import javax.script.*;
import java.io.FileReader;

public class App {

    public static void main(String[] args) {
        ScriptEngineManager manager = new ScriptEngineManager();
        Compilable engine = (Compilable) manager.getEngineByName("groovy");
        CompiledScript script = engine.compile(new FileReader("MyScript.groovy"))
        Object object = script.eval();
        Method method = object.getClass().getDeclaredMethod("sayHello");
        method.invoke(object);
    }
    
}

{{< / highlight >}}


If you don't plan to use multiple languages in the same application, it is highly recommended that you use the Groovy integration mechanisms instead of this limited API.

### Scripting With the GSE

The GroovyScriptEngine (GSE) class provides a flexible foundation for applications that rely on script reloading and script dependencies. In this section, I'm going to give you the simple recipe of how to embed it in a generic way whether you are building a Java console or web application.

First of all, we will start with the simple assertion that the scripts will be stored in a folder. Thereafter, we can create this simple context-agnostic _ScriptManager_ class to have the job done.

{{< highlight java>}}

import java.io.File;
import java.net.URL;
import groovy.util.GroovyScriptEngine;

public class ScriptManager {
    
    protected final GroovyScriptEngine engine;
    
    public ScriptManager(File folder) {
      engine = createScriptEngine(folder);
    }
    
    protected GroovyScriptEngine createScriptEngine(File folder) {
      URL[] urls = { folder.toURI().toURL() };
      return new GroovyScriptEngine(urls, this.getClass().getClassLoader());
    }
    
    public Object loadScript(String name) {
      return engine.loadScriptByName(name).newInstance();
    }
    
}

{{< / highlight >}}


You can enhance it later to provide a configuration to the Groovy Script Engine in order to customize the compiler. One simple use case would be to apply the TypeChecked or  CompileStatic AST annotation to all source files. For further information, please read this blog post.

To load your scripts in your Java applications with the _ScriptManager_ that we created earlier is fairly easy, and unless you can come up with a better solution, make them always implement an interface or extend an abstract Java class so you can perform a cast. Reflection is by nature a costly operation due to the process of checking the class, fields, the methods, and so on.

In the example below, which is a simple skeleton that can be made later more sophisticated, we are going to execute a set of jobs. The design is taken from a Java batch archiving tool that I built to backup our database tables based on a period. The dependent scripts are run in parallel using threads, and the persistence is achieved with the Groovy SQL module, which is a facade over Java's normal JDBC APIs, providing greatly simplified resource management and result set handling.

For the sake of simplicity, in this example, the abstract Job class will expose a single _execute()_ method with no arguments

#### Job.java

{{< highlight java>}}

public abstract class Job {
  public abstract void execute();
}

{{< / highlight >}}


And the Groovy scripts can now extend it to provide their own implementation.

#### MyJob.groovy


{{< highlight java>}}

class MyJob extends Job {
  void execute() {
     println "I can use groovy sql to make queries" 
  }
}

{{< / highlight >}}

When the application is launched, the Job instances are created, and in a scenario other than this current one, they could have been executed asynchronously. 

#### Java Console App

{{< highlight java>}}

import java.io.File;

public class App {

  public static void main(String[] args) {
    File folder = new File("scripts");
    ScriptManager scriptManager = new ScriptManager(folder);
    File[] files = folder.listFiles();
    if(files!=null) {
      for(File script : files) {
        Job job = (Job) scriptManager.loadScript(script.getName());
        job.execute();
      }
    }
  }
  
}

{{< / highlight >}}


In the context of a web application and if not delayed, a _ServletContextListener_ can be the entry point to load the scripts at runtime. In the [GServlet](https://github.com/GServlet/gservlet-api) project, this is how a servlet, a filter, or a listener is registered at start-up into the web container though the registration has been delegated to another class. The scripts folder is also looped recursively to help you divide your application into packages.

#### Java Web App

{{< highlight java>}}

import java.io.File;
import javax.servlet.ServletContext;
import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;
import javax.servlet.annotation.WebListener;

@WebListener
public class StartupListener implements ServletContextListener {
  
   @Override
   public void contextInitialized(ServletContextEvent event) {
     ServletContext context = event.getServletContext();
     File folder = new File(context.getRealPath("/")+File.separator+"scripts");
     ScriptManager scriptManager = new ScriptManager(folder);
     File[] files = folder.listFiles();
     if(files!=null) {
       for(File script : files) {
         Job job = (Job) scriptManager.loadScript(script.getName());
         job.execute();
       }
     }
  }
  
  @Override
  public void contextDestroyed(ServletContextEvent event) {
  }
  
}

{{< / highlight >}}


As you can see it also, to use the _ScriptManager_ class in a test case is definitely something straightforward.


{{< highlight java>}}

import static org.junit.Assert.*;
import java.io.File;
import java.lang.annotation.Annotation;
import org.gservlet.annotation.Servlet;
import org.junit.Test;
public class ScriptManagerTest {

   @Test
   public void loadScripts() {
     File folder = new File("src/test/resources/scripts");
     assertEquals(true, folder.exists());
     ScriptManager scriptManager = new ScriptManager(folder);
     File[] files = folder.listFiles();
     if(files!=null) {
       for(File script : files) {
          Object object = scriptManager.loadScript(script.getName());
          Annotation[] annotations = object.getClass().getAnnotations();
          for(Annotation current : annotations) {
              if(current instanceof Servlet) {
                assertEquals("ServletTest",object.getClass().getName());
                assertEquals(HttpServlet.class, object.getClass().getSuperclass());
                Servlet annotation = (Servlet) current;
                assertEquals("/servlet", annotation.value()[0]);
              }
          }
       }
     }
   }
   
}

{{< / highlight >}}


In the next blog post, we will cover among other topics and how with an instance of a Groovy _MarkupBuilder_ bound to the response writer. We can make a servlet to generate an HTML markup like a Groovlet.

#### MyServlet.groovy

{{< highlight java>}}

import org.gservlet.annotation.Servlet

@Servlet("/myServlet")
class MyServlet {

   void get() {
      html.html {
        body {
          p("Hello World")
        }
      }
   }
   
}

{{< / highlight >}}

#### Generated HTML

{{< highlight html>}}

<!DOCTYPE html>
<html>
  <body>
    <p>Hello World</p>
  </body>
</html>

{{< / highlight >}}


Thanks for reading! And stay tuned for more on Groovy servlets!