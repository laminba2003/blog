---
title: "Getting Started with GServlet"
date: 2021-09-06T18:07:51+01:00
description : "GServlet is an open source project inspired from the Groovlets, which aims to use the Groovy language and its provided modules to simplify Servlet API web development."
draft: false
tags : [
    "JAVA" , "SERVLET", "GROOVY"
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

pom.xml
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

Next, we create a _FileListener_ interface that must be implemented by an observer in order to be notified for file events. 
It extends the _java.util.EventListener_ interface which is a tagging interface that all event listener interfaces must extend.


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

That's all for now and in the next articles, we will cover more ground and how this API can be even integrated in your SpringBoot applications. This example is available for download on GitHub and you can read as well the documentation for in-depth knowledge of this project. 

GServlet will start to support JakartaEE 9 in its 2.0.0 version. You can access snapshot builds from the sonatype snapshot repository

The last piece of the puzzle is to create the subject, which maintains the list of observers, and notifies them of any state changes, by calling one of their methods. 
We are going to name it _FileWatcher_ and given a folder, this is how an instance of this class is constructed.

{{< highlight java>}}


public class FileWatcher {

  protected List<FileListener> listeners = new ArrayList<>();
  protected final File folder;
  
  public FileWatcher(File folder) {
    this.folder = folder;
  }
  
  public List<FileListener> getListeners() {
    return listeners;
  }
  
  public FileWatcher setListeners(List<FileListener> listeners) {
    this.listeners = listeners;
    return this;
  }
  
}

{{< / highlight >}}


It can implement the _Runnable_ interface so we can start the watch process with a daemon thread when invoking its _watch()_ method if the folder exists.

{{< highlight java>}}

public class FileWatcher implements Runnable {

  public void watch() {
    if (folder.exists()) {
      Thread thread = new Thread(this);
      thread.setDaemon(true);
      thread.start();
    }
  }
  
  @Override
  public void run() {
  // implementation not yet provided
  }
  
}

{{< / highlight >}}



In the implementation of its _run()_ method, a _WatchService_ instance is created to poll for events within a try-with-resources statement. 
We will keep a track of it using a static final list in the _FileWatcher_ class, so we can later invoke its _close()_ method to cause any thread 
waiting to retrieve keys, to throw the unchecked _ClosedWatchServiceException_ which will interrupt the watch process in a clean way. 
Therefore,  we will get no memory leak [warnings](https://stackoverflow.com/questions/32881405/watchservice-causing-memory-leak-in-tomcat) when the application is being gracefully shutdown.

{{< highlight java>}}

@Override 
public void contextDestroyed(ServletContextEvent event) {
  for (WatchService watchService : FileWatcher.getWatchServices()){
    try {
      watchService.close();
    } catch (IOException e) {
    }
  }
}

{{< / highlight >}}

{{< highlight java>}}

public class FileWatcher implements Runnable {

  protected static final List<WatchService> watchServices = new ArrayList<>();
  
  @Override
  public void run() {
    try (WatchService watchService = FileSystems.getDefault().newWatchService()) {
      Path path = Paths.get(folder.getAbsolutePath());
      path.register(watchService, ENTRY_CREATE, ENTRY_MODIFY, ENTRY_DELETE);
      watchServices.add(watchService);
      boolean poll = true;
      while (poll) {
        poll = pollEvents(watchService);
      }
    } catch (IOException | InterruptedException | ClosedWatchServiceException e) {
       Thread.currentThread().interrupt();
    }
  }
  
  protected boolean pollEvents(WatchService watchService) throws InterruptedException {
    WatchKey key = watchService.take();
    Path path = (Path) key.watchable();
    for (WatchEvent<?> event : key.pollEvents()) {
      notifyListeners(event.kind(), path.resolve((Path) event.context()).toFile());
    }
    return key.reset();
  }
  
  public static List<WatchService> getWatchServices() {
    return Collections.unmodifiableList(watchServices);
  }
  
}

{{< / highlight >}}


Whenever an event occurs, the file path is fully resolved and the listeners are notified accordingly. If it is the creation of a new folder, another _FileWatcher_ instance will be created for its monitoring. 

{{< highlight java>}}

public class FileWatcher implements Runnable {

  protected void notifyListeners(WatchEvent.Kind<?> kind, File file) {
    FileEvent event = new FileEvent(file);
    if (kind == ENTRY_CREATE) {
      for (FileListener listener : listeners) {
         listener.onCreated(event);
      }
      if (file.isDirectory()) {
         // create a new FileWatcher instance to watch the new directory
         new FileWatcher(file).setListeners(listeners).watch();
      }
    } 
    else if (kind == ENTRY_MODIFY) {
      for (FileListener listener : listeners) {
        listener.onModified(event);
      }
    }
    else if (kind == ENTRY_DELETE) {
      for (FileListener listener : listeners) {
        listener.onDeleted(event);
      }
    }
  }
  
}

{{< / highlight >}}



Here is the complete listing of the _FileWatcher_ class.

### FileWatcher.java

{{< highlight java>}}

import static java.nio.file.StandardWatchEventKinds.*;
import java.io.File;
import java.io.IOException;
import java.nio.file.ClosedWatchServiceException;
import java.nio.file.FileSystems;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.WatchEvent;
import java.nio.file.WatchKey;
import java.nio.file.WatchService;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class FileWatcher implements Runnable {

  protected List<FileListener> listeners = new ArrayList<>();
  protected final File folder;
  protected static final List<WatchService> watchServices = new ArrayList<>();
  
  public FileWatcher(File folder) {
    this.folder = folder;
  }
  
  public void watch() {
    if (folder.exists()) {
      Thread thread = new Thread(this);
      thread.setDaemon(true);
      thread.start();
    }
  }
  
  @Override
  public void run() {
    try (WatchService watchService = FileSystems.getDefault().newWatchService()) {
      Path path = Paths.get(folder.getAbsolutePath());
      path.register(watchService, ENTRY_CREATE, ENTRY_MODIFY, ENTRY_DELETE);
      watchServices.add(watchService);
      boolean poll = true;
      while (poll) {
        poll = pollEvents(watchService);
      }
    } catch (IOException | InterruptedException | ClosedWatchServiceException e) {
       Thread.currentThread().interrupt();
    }
  }
  
  protected boolean pollEvents(WatchService watchService) throws InterruptedException {
    WatchKey key = watchService.take();
    Path path = (Path) key.watchable();
    for (WatchEvent<?> event : key.pollEvents()) {
      notifyListeners(event.kind(), path.resolve((Path) event.context()).toFile());
    }
    return key.reset();
  }
  
  protected void notifyListeners(WatchEvent.Kind<?> kind, File file) {
    FileEvent event = new FileEvent(file);
    if (kind == ENTRY_CREATE) {
      for (FileListener listener : listeners) {
        listener.onCreated(event);
      }
      if (file.isDirectory()) {
        new FileWatcher(file).setListeners(listeners).watch();
      }
   } 
   else if (kind == ENTRY_MODIFY) {
     for (FileListener listener : listeners) {
       listener.onModified(event);
     }
   }
   else if (kind == ENTRY_DELETE) {
     for (FileListener listener : listeners) {
       listener.onDeleted(event);
     }
   }
  }
  
  public FileWatcher addListener(FileListener listener) {
    listeners.add(listener);
    return this;
  }
  
  public FileWatcher removeListener(FileListener listener) {
    listeners.remove(listener);
    return this;
  }
  
  public List<FileListener> getListeners() {
    return listeners;
  }
  
  public FileWatcher setListeners(List<FileListener> listeners) {
    this.listeners = listeners;
    return this;
  }
  
  public static List<WatchService> getWatchServices() {
    return Collections.unmodifiableList(watchServices);
  }
  
}

{{< / highlight >}}


The final touch of our design can be the creation of a _FileAdapter_ class which provides a default implementation of the _FileListener_ interface so we can process only few of the events to save code.

### FileAdapter.java

{{< highlight java>}}

public abstract class FileAdapter implements FileListener {

  @Override
  public void onCreated(FileEvent event) {
    // no implementation provided
  }
  
  @Override
  public void onModified(FileEvent event) {
   // no implementation provided
  }
  
  @Override
  public void onDeleted(FileEvent event) {
   // no implementation provided
  }
  
}

{{< / highlight >}}

The _FileAdapter_ class is very useful in my case, to reload a Groovy script when developing a servlet application within my IDE. When a file is modified and republished in the deployment directory, it is first deleted before being recreated. Therefore, the modification event which is fired twice on my Windows platform, can be ignored and its deletion counterpart is unusable in my context since currently, we can't unregister a servlet, filter or listener from the web container. Thus, I found no reason yet to have such feature enabled in production. Also in this use case, performance is not even a concern since it will be hard to have even five packages to be watched by a different _FileWatcher_ instance.

{{< highlight java>}}

protected void loadScripts(File folder) {
   if (folder.exists()) {
	 File[] files = folder.listFiles();
	 if (files != null) {
	   for (File file : files) {
	     if (file.isFile()) {
		   Object object = scriptManager.loadScript(file);
		   register(object);
	     } else {
		   loadScripts(file);
	     }
	   }
     }
     watch(folder);
   }
}

protected void watch(File folder) {
  new FileWatcher(folder).addListener(new FileAdapter() {
    @Override
    public void onCreated(FileEvent event) {
      File file = event.getFile();
      if (file.isFile()) {
        logger.info("processing script " + file.getName());
        process(file);
     }
   }
  }).watch();
}

protected void process(File script) {
  Object object = scriptManager.loadScript(script);
  // update the application accordingly   
}

{{< / highlight >}}

Even though, it is said to use the _Thread.sleep()_ method in an unit test is generally a bad idea, we are going to use it to write a test case for the _FileWatcher_ class since we need a delay between the operations.

{{< highlight java>}}

import static org.junit.Assert.*;
import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;
import org.junit.Test;

public class FileWatcherTest {

   @Test
   public void test() throws IOException, InterruptedException {
      File folder = new File("src/test/resources");
      final Map<String, String> map = new HashMap<>();
      FileWatcher watcher = new FileWatcher(folder);
      watcher.addListener(new FileAdapter() {
         public void onCreated(FileEvent event) {
           map.put("file.created", event.getFile().getName());
         }
         public void onModified(FileEvent event) {
           map.put("file.modified", event.getFile().getName());
         }
         public void onDeleted(FileEvent event) {
           map.put("file.deleted", event.getFile().getName());
         }
      }).watch();
      assertEquals(1, watcher.getListeners().size());
      wait(2000);
      File file = new File(folder + "/test.txt");
      try(FileWriter writer = new FileWriter(file)) {
        writer.write("Some String");
      }
      wait(2000);
      file.delete();
      wait(2000);
      assertEquals(file.getName(), map.get("file.created"));
      assertEquals(file.getName(), map.get("file.modified"));
      assertEquals(file.getName(), map.get("file.deleted"));
   }
   
   public void wait(int time) throws InterruptedException {
      Thread.sleep(time);
   }
   
}

{{< / highlight >}}


In my previous blog post, ["Groovify Your Java Servlets (Part 2): Scripting the JVM"](../groovify-your-servlets-part-2/), I shown how to instantiate an object from a script with the Groovy Script Engine using a simple _ScriptManager_ class. This one may be the perfect opportunity for me to correct its implementation, by replacing the deprecated _Class.newInstance()_ method with the _Class.getConstructor().newInstance()_ method in order to make it right without the exceptions thrown.

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
      return engine.loadScriptByName(name).getConstructor().newInstance()
    }
    
}

{{< / highlight >}}


The class above can't load scripts located in the subdirectories of the given folder unless you pass the relative path in the script name argument. That is the reason why, it is better to write it like this:

{{< highlight java>}}

import java.io.File;
import java.net.URL;
import groovy.util.GroovyScriptEngine;

public class ScriptManager {
    
    protected final File folder;
    protected final GroovyScriptEngine engine;
    
    public ScriptManager(File folder) {
      this.folder = folder;
      engine = createScriptEngine();
    }
    
    protected GroovyScriptEngine createScriptEngine() {
      URL[] urls = { folder.toURI().toURL() };
      return new GroovyScriptEngine(urls, this.getClass().getClassLoader());
    }
    
    public Object loadScript(File file) {
      String name = file.getAbsolutePath().substring(folder.getAbsolutePath().length() + 1);
      return engine.loadScriptByName(name).getConstructor().newInstance()
    }
    
}

{{< / highlight >}}
