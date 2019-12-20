---
title: "Watching Files With Java NIO"
date: 2019-12-20T18:07:51+01:00
draft: false
tags : [
    "JAVA" , "NIO", "MONITORING"
]
---

The _java.nio.file_ package provides a file change notification API, called the Watch Service API.
It enables us to register a folder with the watch service. When registering, we tell the service which types of events we are interested in: file creation, file modification, or file deletion. When the service detects an event of interest, it is forwarded to the registered process and handled as needed. This is basically how it works:

1. The first step is to create a new _WatchService_ by using the _newWatchService()_ method of the _FileSystem_ class.
2. Next, we register a _Path_ instance for the folder to be monitored with the types of events that we are interested in.
3. And at last, we implement an infinite loop to wait for incoming events. 
When an event occurs, the key is signaled and placed into the watcher's queue. 
After processing its events, we need to put it back into a ready state by invoking its _reset()_
 method. If it returns false, the key is no longer valid and the loop can exit.

{{< highlight java>}}
WatchService watchService = FileSystems.getDefault().newWatchService();
Path path = Paths.get("c:\\directory");
path.register(watchService, ENTRY_CREATE, ENTRY_MODIFY, ENTRY_DELETE);
boolean poll = true;
while (poll) {
  WatchKey key = watchService.take();
  for (WatchEvent<?> event : key.pollEvents()) {
    System.out.println("Event kind : " + event.kind() + " - File : " + event.context());
  }
  poll = key.reset();
}
{{< / highlight >}}
 

This is the console output 

{{< highlight plain>}}
Event kind : ENTRY_CREATE - File : file.txt
Event kind : ENTRY_DELETE - File : file.txt
Event kind : ENTRY_CREATE - File : test.txt
Event kind : ENTRY_MODIFY - File : test.txt
{{< / highlight >}}

The Watch Service API is fairly low level, allowing us to customize it. In this article, we are going to design a high-level API on top of this mechanism for listening to file events for a given folder. 
We will begin by creating a _FileEvent_ class which extends the _java.util.EventObject_ from which all event state objects shall be derived. 
A _FileEvent_ instance is constructed with a reference to the source, which is logically the file upon which the event occurred upon. 

### FileEvent.java

{{< highlight java>}}
import java.io.File;
import java.util.EventObject;

public class FileEvent extends EventObject {

  public FileEvent(File file) {
    super(file);
  }
  
  public File getFile() {
    return (File) getSource();
  }
  
}
{{< / highlight >}}


Next, we create the _FileListener_ interface that must be implemented by an observer in order to be notified for file events. 
It extends the _java.util.EventListener_ interface which is a tagging interface that all event listener interfaces must extend.


### FileListener.java

{{< highlight java>}}
import java.util.EventListener;

public interface FileListener extends EventListener {

    public void onCreated(FileEvent event);
    public void onModified(FileEvent event);
    public void onDeleted(FileEvent event);
    
}
{{< / highlight >}}


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


Whenever an event occurs, the file path is resolved and the listeners are notified accordingly . If it is the creation of a new folder, another _FileWatcher_ instance will be created for its monitoring. 

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


The final touch of our design can be the creation of a _FileAdapter_ class which provides a default implementation of the _FileListener_ class so we can process only few of the events to save code.

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

The _FileAdapter_ class is very useful in my case, to reload a Groovy script when developing a servlet application within my IDE. When a file is modified and republished in the deployment directory, it is first deleted before being recreated. Therefore, the modification event which is fired twice on my Windows platform, can be ignored and its deletion counterpart is unusable in my context since currently, we can't unregister a servlet, filter or listener from the web container. Thus, I found no reason yet to have such feature enabled in production and in this use case, performance is not even a concern since it will be hard to have even five packages to watch by a different _FileWatcher_ instance.

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


In my previous article, ["Groovify Your Java Servlets (Part 2): Scripting the JVM"](../groovify-your-servlets-part-2/), I shown how to instantiate an object from a script with the Groovy Script Engine using a simple _ScriptManager_ class. This one may be the perfect opportunity for me to correct its implementation, by replacing the deprecated _Class.newInstance()_ method with the _Class.getConstructor().newInstance()_ method in order to make it right without the exceptions thrown.

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
