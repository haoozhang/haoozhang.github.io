---
layout:     post
title:      Chapter 22. CLR Hosting and AppDomains
subtitle:   
date:       2022-09-14
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

In this chapter, I’ll discuss two main topics that show the incredible value provided by .NET Framework: **hosting and AppDomains**. 

Hosting allows any application to use the features of CLR, this allows existing applications to be at least partially written using managed code. Furthermore, hosting allows applications customization and extension via programming.

Allowing extensibility means that third-party code will be running inside your process, which is fraught with peril because the third party DLL may hurt application's data structure, or access those resources that should not have access to. The CLR’s AppDomain feature solves all of these problems.

## CLR Hosting

The .NET Framework runs on top of Windows. So .NET Framework must be built using technologies that Windows can interface with.

All managed module and assembly files must be Windows portable executable file format, either a **EXE or DLL**.

When developing the CLR, Microsoft implemented it as a COM server contained inside a DLL. Any Windows application can host the CLR. You should create in instance of CLR COM server by calling **CLRCreateInstance** function declared in **MetaHost.h** and implemented in **MSCorEE.dll** file. 

A single machine may have multiple versions of the CLR installed, but there will be only one version of the **MSCorEE.dll** file (called the **shim**).

## AppDomains

When the CLR COM server initializes, it creates an AppDomain. An AppDomain is a logical container for a set of assemblies. \
The first AppDomain created when the CLR is initialized is called the default AppDomain; it is destroyed only when the Windows process terminates.

In addition to the default AppDomain, a host can instruct CLR to create additional AppDomains. The whole purpose of an AppDomain is to provide isolation. 
+ Objects created in one AppDomain cannot be accessed directly by another AppDomain.
+ AppDomains can be unloaded. CLR doesn't support to unload a single assembly from an AppDomain. But you can tell CLR to unload an AppDomain.
+ AppDomains can be individually secured. When created, an AppDomain can have a per- mission set applied to determine the max right.
+ AppDomains can be individually configured. When created, an AppDomain can have a bunch of configuration settings associated with it.

![img](/img/CLR/CLR_and_AppDomain.png)

The figure shows a single Windows process that has one CLR COM server running in it. CLR is currently managing two AppDomains. Each AppDomain has its own loader heap, which maintains which types have been accessed. Each type object has a method table, points to JIT-compiled native code if the method has been executed at least once.

In addition, each AppDomain has some assemblies loaded into it. They are no shared.

Some assemblies are expected to be used by several AppDomains, e.g., **MSCorLib.dll**, it contains System.Object, System.Int32, and all of the other types. This assembly is automatically loaded when the CLR initializes, and all AppDomains share the types in this assembly. To reduce resource usage, MSCorLib.dll is loaded in an **AppDomain-neutral** fashion. Unfortunately, assemblies that are loaded domain-neutral can never be unloaded until Windows process terminates.

### Accessing Objects Across AppDomain Boundaries

Code in one AppDomain can communicate with types and objects contained in another AppDomain. However, it is allowed only through well-defined mechanisms. The code shows the different behaviors when constructing a type: marshaled by reference, marshaled by value, and can’t be marshaled.

```c#
private static void Marshalling() {
    // Get a reference to the AppDomain that the calling thread is executing in
    AppDomain adCallingThreadDomain = Thread.GetDomain();
 
    // Every AppDomain is assigned a friendly string name (helpful for debugging)
    // Get this AppDomain's friendly string name and display it
    String callingDomainName = adCallingThreadDomain.FriendlyName;
    Console.WriteLine("Default AppDomain's friendly name={0}", callingDomainName);
 
    // Get and display the assembly in our AppDomain that contains the 'Main' method
    String exeAssembly = Assembly.GetEntryAssembly().FullName;
    Console.WriteLine("Main assembly={0}", exeAssembly);
 
    // Define a local variable that can refer to an AppDomain
    AppDomain ad2 = null;
 
    // *** DEMO 1: Cross­AppDomain Communication Using Marshal­by­Reference ***
    Console.WriteLine("{0}Demo #1", Environment.NewLine);
 
    // Create new AppDomain (security and configuration match current AppDomain)
    ad2 = AppDomain.CreateDomain("AD #2", null, null);
 
    MarshalByRefType mbrt = null;
    // Load our assembly into the new AppDomain, construct an object, marshal it back to our AD 
    // Because isolation would be broken if a reference can be returned directly, we really get a reference to a proxy
    mbrt = (MarshalByRefType) ad2.CreateInstanceAndUnwrap(exeAssembly, "MarshalByRefType");
    Console.WriteLine("Type={0}", mbrt.GetType());  // The CLR lies about the type
 
    // Prove that we got a reference to a proxy object
    Console.WriteLine("Is proxy={0}", RemotingServices.IsTransparentProxy(mbrt));
    
    // This looks like we're calling a method on MarshalByRefType but we're not.
    // We're calling a method on the proxy type. The proxy transitions the thread
    // to the AppDomain owning the object and calls this method on the real object.
    mbrt.SomeMethod();
    
    // Unload the new AppDomain
    AppDomain.Unload(ad2);

    // mbrt refers to a valid proxy object; the proxy object refers to an invalid AppDomain
    try {
        // We're calling a method on the proxy type. The AD is invalid, exception is thrown
        mbrt.SomeMethod();
        Console.WriteLine("Successful call.");
    }
    catch (AppDomainUnloadedException) {
        Console.WriteLine("Failed call.");
    }

    // *** DEMO 2: Cross­AppDomain Communication Using Marshal­by­Value ***
    Console.WriteLine("{0}Demo #2", Environment.NewLine);

    // Create new AppDomain (security and configuration match current AppDomain)
    ad2 = AppDomain.CreateDomain("AD #2", null, null);
    
    // Load our assembly into the new AppDomain, construct an object, marshal
    // it back to our AD (we really get a reference to a proxy)
    mbrt = (MarshalByRefType) ad2.CreateInstanceAndUnwrap(exeAssembly, "MarshalByRefType");

    // The object's method returns a COPY of the returned object;
    // the object is marshaled by value (not by reference).
    MarshalByValType mbvt = mbrt.MethodWithReturn();
    
    // Prove that we did NOT get a reference to a proxy object
    Console.WriteLine("Is proxy={0}", RemotingServices.IsTransparentProxy(mbvt));
    
    // Here we're calling a method on MarshalByValType really
    Console.WriteLine("Returned object created " + mbvt.ToString());

    // Unload the new AppDomain
    AppDomain.Unload(ad2);

    // mbvt refers to valid object; unloading the AppDomain has no impact.
    try {
        // We're calling a method on an object; no exception is thrown
        Console.WriteLine("Returned object created " + mbvt.ToString());
        Console.WriteLine("Successful call.");
    }
    catch (AppDomainUnloadedException) {
        Console.WriteLine("Failed call.");
    }

    // DEMO 3: Cross­AppDomain Communication Using non­marshalable type ***
    Console.WriteLine("{0}Demo #3", Environment.NewLine);

    // Create new AppDomain (security and configuration match current AppDomain)
    ad2 = AppDomain.CreateDomain("AD #2", null, null);
    
    // Load our assembly into the new AppDomain, construct an object, marshal
    // it back to our AD (we really get a reference to a proxy)
    mbrt = (MarshalByRefType) ad2.CreateInstanceAndUnwrap(exeAssembly, "MarshalByRefType");
   
    // The object's method returns a non­marshalable object; exception
    NonMarshalableType nmt = mbrt.MethodArgAndReturn(callingDomainName);
    
    // We won't get here...
}

// Instances can be marshaled­by­reference across AppDomain boundaries
public sealed class MarshalByRefType : MarshalByRefObject {
    public MarshalByRefType() {
        Console.WriteLine("{0} ctor running in {1}", this.GetType().ToString(), Thread.GetDomain().FriendlyName);
    }
    public void SomeMethod() {
        Console.WriteLine("Executing in " + Thread.GetDomain().FriendlyName);
    }

    public MarshalByValType MethodWithReturn() {
        Console.WriteLine("Executing in " + Thread.GetDomain().FriendlyName);
        MarshalByValType t = new MarshalByValType();
        return t;
    }

    public NonMarshalableType MethodArgAndReturn(String callingDomainName) {
        // NOTE: callingDomainName is [Serializable]
        Console.WriteLine("Calling from '{0}' to '{1}'.", callingDomainName, Thread.GetDomain().FriendlyName);
        NonMarshalableType t = new NonMarshalableType();
        return t;
    } 
}

// Instances can be marshaled­by­value across AppDomain boundaries
[Serializable]
public sealed class MarshalByValType : Object {
    private DateTime m_creationTime = DateTime.Now; // NOTE: DateTime is [Serializable]
    public MarshalByValType() {
        Console.WriteLine("{0} ctor running in {1}, Created on {2:D}",
            this.GetType().ToString(),
            Thread.GetDomain().FriendlyName,
            m_creationTime);
    }
   
    public override String ToString() {
        return m_creationTime.ToLongDateString();
    }
}

// Instances cannot be marshaled across AppDomain boundaries
// [Serializable]
public sealed class NonMarshalableType : Object {
    public NonMarshalableType() {
        Console.WriteLine("Executing in " + Thread.GetDomain().FriendlyName);
    } 
}
```

## AppDomain Unloading

One of the great features of AppDomains is that you can unload them, which causes the CLR to unload all of the assemblies and loader heap in the AppDomain. You can call AppDomain’s **Unload** static method to achieve it.

1. The CLR suspends all threads in the process that have ever executed managed code.
2. The CLR examines all of the threads’ stacks to see which threads are currently using the AppDomain being unloaded. The CLR forces those threads to throw a **ThreadAbortException**. If no code catches the **ThreadAbortException**, it will eventually become an unhandled exception, and CLR swallows it, the process can continue running. (for other unhandled exception, CLR kills the process)
3. CLR walks the heap and set a flag for each object that referred to an object created by the unloaded AppDomain. 
4. The CLR forces a garbage collection to occur, reclaiming the memory used by any objects that were created by the now unloaded AppDomain.
5. The CLR resumes all of the remaining threads. 

## AppDomain Monitoring

A host application can monitor the resources that an AppDomain consumes, by setting AppDomain’s static **MonitoringEnabled** property to **true**.

After monitoring id turned on, you can query the following read-only properties:
+ **MonitoringSurvivedProcessMemorySize** This static Int64 property returns the number of bytes that are currently in use by **all** AppDomains controlled by the current CLR instance.
+ **MonitoringTotalAllocatedMemorySize** This instance Int64 property returns the num- ber of bytes that have been allocated by a **specific** AppDomain.
+ **MonitoringSurvivedMemorySize** This instance Int64 property returns the number of bytes that are currently in use by a specific AppDomain.
+ **MonitoringTotalProcessorTime** This instance TimeSpan property returns the amount of CPU usage incurred by a specific AppDomain.

The following code shows how to use these properties to see what has changed before and after GC.

```c#
private sealed class AppDomainMonitorDelta : IDisposable {
    private AppDomain m_appDomain;
    private TimeSpan m_thisADCpu;
    private Int64 m_thisADMemoryInUse;
    private Int64 m_thisADMemoryAllocated;

    static AppDomainMonitorDelta() {
        // Make sure that AppDomain monitoring is turned on
        AppDomain.MonitoringIsEnabled = true;
    }

    public AppDomainMonitorDelta(AppDomain ad) {
        m_appDomain = ad ?? AppDomain.CurrentDomain;
        m_thisADCpu = m_appDomain.MonitoringTotalProcessorTime;
        m_thisADMemoryInUse = m_appDomain.MonitoringSurvivedMemorySize;
        m_thisADMemoryAllocated = m_appDomain.MonitoringTotalAllocatedMemorySize;
    }

    public void Dispose() {
        GC.Collect();

        Console.WriteLine(“FriendlyName={0}, CPU={1}ms”, m_appDomain.FriendlyName,
          (m_appDomain.MonitoringTotalProcessorTime -­ m_thisADCpu).TotalMilliseconds);
        Console.WriteLine(“   Allocated {0:N0} bytes of which {1:N0} survived GCs”,
          m_appDomain.MonitoringTotalAllocatedMemorySize -­ m_thisADMemoryAllocated,
          m_appDomain.MonitoringSurvivedMemorySize -­ m_thisADMemoryInUse);
    }
}

private static void AppDomainResourceMonitoring() {
    using (new AppDomainMonitorDelta(null)) {
       // Allocate about 10 million bytes that will survive collections
       var list = new List<Object>();
       for (Int32 x = 0; x < 1000; x++) list.Add(new Byte[10000]);
       // Allocate about 20 million bytes that will NOT survive collections
       for (Int32 x = 0; x < 2000; x++) new Byte[10000].GetType();
       // Spin the CPU for about 5 seconds
       Int64 stop = Environment.TickCount + 5000;
       while (Environment.TickCount < stop) ;
    } 
}
```

## AppDomain Exception Notifications

Each AppDomain can associate with a series of callback methods, which get invoked when the CLR begins looking for catch blocks within an AppDomain. \
These methods can perform logging. The callbacks cannot handle or swallow the exception, they are just receiving the notification that the exception has occurred.

To register a callback method, just add a delegate to AppDomain’s instance **FirstChanceException** event

### How the CLR processes an exception

when the exception is first thrown, the CLR invokes any **FirstChanceException** callback methods registered with the AppDomain, then CLR looks for any catch blocks to handle the exception, if a catch block handles it, execution continues as normal. \
If the AppDomain has no catch block to handle the exception, then the CLR walks up the stack to the calling AppDomain and throws the same exception again, CLR invokes any FirstChanceException callback methods registered with the now current AppDomain. \
This continues until the top of the thread’s stack is reached.

## How Hosts Use AppDomains

I will describe some common hosting and AppDomain scenarios.

### Executable Applications

Console applications, Windows Forms applications, and WPF applications are examples of **self-hosted** applications. They have managed EXE files.

When Windows intialize a process using the managed EXE file, it **loads the shim**, and the shim examines the CLR header information of application's assembly (EXE file). The header information indicates the version of the CLR version that used to build and test. The shim **determine the corresponding version of CLR** to load into the process. After the CLR loads and initializes, it again examines the assembly’s CLR header to determine the application's entry point (Main method). The CLR invokes this method, and the applica- tion is now up and running.

As the code runs, when referencing a type contained in another assembly, the CLR locates the necessary assembly and loads it into the same AppDomain. When the application’s Main method returns, the Windows process terminates (destorying all AppDomains).

### ASP.NET and XML Web Service Applications

ASP.NET is implemented as an ISAPI DLL. The first time a client requests a URL, ASP.NET loads the CLR.

When a client makes a request of a web application, ASP.NET determines if this is the first time a request has been made. If it is, ASp.NET tells the CLR to create a new AppDomain for this web application, then loads the assembly into this new AppDomain,  to reponse the client's request.

When future clients make requests of an already running web application, ASP.NET uses the existing AppDomain, creates a new instance of the web appli- cation’s type, and starts calling methods. The methods will already be JIT-compiled into native code, so the performance of processing all subsequent client requests is excellent.
