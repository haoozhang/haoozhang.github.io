---
layout:     post
title:      Chapter 20. Exception and State Management
subtitle:   
date:       2022-09-07
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

Errors are always possible, so we need a way to handle those errors. The mechanism provided by .NET and all programming languages that support it is called **Exception Handling**.

## Exception Handling Mechanism

```c#
private void SomeMethod() {
    try {
        // Put the code that requires graceful recovery here...
    }
    catch (InvalidOperationException) {
        // Put code that recovers from an InvalidOperationException here...
    }
    catch (IOException) {
        // Put code that recovers from an IOException here...
    }
    catch {
        // Put code that recovers from any kind of exception other than those preceding this...
        // When catching any exception, you usually re­throw the exception.
        throw;
    }
    finally {
        // Put code that cleans up any operations started within the try block here...
        // The code in here ALWAYS executes, regardless of whether an exception is thrown.
    }
    // Code below the finally block executes if no exception is thrown within the try block
    // or if a catch block catches the exception and doesn't throw or re­throw an exception.
}
```

### try block

A **try** block can contain code that might potentially throw an exception. It must be associated with at least one **catch** or **finally** block., it makes no sense to have a **try** block only.

### catch block

A **catch** block contains code to execute in response to an exception. A **try** block can have zero or more **catch** blocks associated with it. 

CLR searched from top to bottom for matching **catch** type, so you should place the more specific exception types at the top.

If an exception is thrown by code within the **try** block, the CLR starts searching for **catch** blocks whose catch type is same or a base type of the thrown exception. \
If none of the catch types match, the CLR continue searching up the call stack. If after reaching the top of the call stack, no matching **catch** blocks is found, an unhandled exception occurs.

If the CLR locates a **catch** block with a matching catch type, it executes the code in all inner **finally** block, starting from within the **try** block throw exception and stopping with the **catch** block matching the exception.

*Note that after all the code in the inner **finally** blocks has executed, the code in the handling catch block executes*.

### finally block

A **finally** block contains code that’s guaranteed to execute. Typically, the **finally** block performs the cleanup operations. For example, if you open a file in a **try** block, you’d put the code to close the file in a **finally** block.

```c#
private void ReadData(String pathname) {
    FileStream fs = null;
    try {
        fs = new FileStream(pathname, FileMode.Open);
        // Process the data in the file...
    }
    catch (IOException) {
        // Put code that recovers from an IOException here...
    }
    finally {
        // Make sure that the file gets closed.
        if (fs != null) fs.Close();
    }
}
```

If the try **block** executes without exception, the file if guaranteed to be closed. If throw an exception, the **finally** block still executes and closed file. \
It's improper to put the statements to close file after the **finally** block, because that if an exception is thrown and not caught, the code after **finally** block will never be executed.

It is always possible that exception-recovery code or cleanup code could fail and throw an exception. In this time, CLR's exception handling mechanism will execute as though the exception were thrown after the **finally** block. This new exception will not be handled and turn into an unhandled exception. The CLR will then terminate your process.

## Throw an Exception

You should throw an exception when the method cannot complete its task. Recommend to throw meaningful exception type. If there may not the one in FCL, we can define own type, derived from **System.Exception**.

If you define an exception type hierarchy, it is highly recommended that the hierarchy be **shallow and wide** in order to create as few base classes as possible. Never throw a **System.Exception** object.

## Define Your Own Exception Class

Unfortunately, designing your own exception is error prone, because all Exception-derived types should be serializable so that ther can be written to log or database. So, to simplify things, I made own generic **Exception\<TExceptionArgs>** class.

```c#
[Serializable]
public sealed class Exception<TExceptionArgs> : Exception, ISerializable 
    where TExceptionArgs : ExceptionArgs {
    
    private const String c_args = "Args";  // For (de)serialization

    private readonly TExceptionArgs m_args;

    public  TExceptionArgs Args { get { return m_args; } }

    public Exception(String message = null, Exception innerException = null)
        : this(null, message, innerException) { }
    
    public Exception(TExceptionArgs args, String message = null, Exception innerException = null)
        : base(message, innerException) { m_args = args; }

    // This constructor is for deserialization; since the class is sealed, the constructor is
    // private. If this class were not sealed, this constructor should be protected
    [SecurityPermission(SecurityAction.LinkDemand, Flags=SecurityPermissionFlag.SerializationFormatter)]
    private Exception(SerializationInfo info, StreamingContext context)
        : base(info, context) {
        m_args = (TExceptionArgs)info.GetValue(c_args, typeof(TExceptionArgs));
    }

    // This method is for serialization; it’s public because of the ISerializable interface
    [SecurityPermission(SecurityAction.LinkDemand, Flags=SecurityPermissionFlag.SerializationFormatter)]
    public override void GetObjectData(SerializationInfo info,StreamingContext context) {
       info.AddValue(c_args, m_args);
       base.GetObjectData(info, context);
    }

    public override String Message {
        get {
            String baseMsg = base.Message;
            return (m_args == null) ? baseMsg : baseMsg + " (" + m_args.Message + ")";
        }
    }
   
    public override Boolean Equals(Object obj) {
        Exception<TExceptionArgs> other = obj as Exception<TExceptionArgs>;
        if (other == null) return false;
        return Object.Equals(m_args, other.m_args) && base.Equals(obj);
    }
   
    public override int GetHashCode() { return base.GetHashCode(); }

}
```

And the ExceptionArgs base class is very simple and lokks like this.

```c#
[Serializable]
public abstract class ExceptionArgs {
    public virtual String Message { get { return String.Empty; } }
}
```

Now, with two classes defined, we can define more exception classes, To define an exception type indicating the disk is full, I simply do the following.

```c#
[Serializable]
public sealed class DiskFullExceptionArgs : ExceptionArgs {

    private readonly String m_diskpath; //   private field set at construction time

    public DiskFullExceptionArgs(String  diskpath) { m_diskpath = diskpath; }

    // Public read­only property that returns the field
    public String DiskPath { get { return    m_diskpath; } }
    
    // Override the Message property to include our field (if set)
    public override String Message {
        get {
         return (m_diskpath == null) ? base.Message : "DiskPath=" + m_diskpath;
        } 
    }
}
```

And now I can write code like this, which throws and catches one of these.
```c#
public static void TestException() {
    try {
        throw new Exception<DiskFullExceptionArgs>(
            new DiskFullExceptionArgs(@"C:\"), "The disk is full");
    }
    catch (Exception<DiskFullExceptionArgs> e) {
       Console.WriteLine(e.Message);
    }
}
```

## Guidelines and Best Practices

### Use finally Blocks

**finally** block allows you to specify a block of code that’s guaranteed to execute no matter what kind of exception throw. You should use finally blocks to clean up from any operation.

```c#
 public sealed class SomeType {
    private void SomeMethod() {
        FileStream fs = new FileStream(@"C:\Data.bin ", FileMode.Open);
        try {
            // Display 100 divided by the first byte in the file.
            Console.WriteLine(100 / fs.ReadByte());
        }
        finally {
         // Put cleanup code in a finally block to ensure that the file gets closed regardless
         // of whether or not an exception occurs (for example, the first byte was 0).
         if (fs != null) fs.Dispose();
        } 
    }
}
```

Ensuring that cleanup code always executes is so important that many programming languages offer constructs that make writing cleanup code easier. For example, when you use the **lock**, **using**, **foreach** statements, C# will automatically emit **try/finally** blocks and puts the cleanup code inside the **finally** block.

For example, use **using** statement to write the previous example.

```c#
internal sealed class SomeType {
    private void SomeMethod() {
        using (FileStream fs = new FileStream(@"C:\Data.bin", FileMode.Open)) {
            // Display 100 divided by the first byte in the file.
            Console.WriteLine(100 / fs.ReadByte());
        } 
    }
}
```

### Don’t Catch Everything

When you catch an exception, it indicates that you expected this exception, you understand why it occurred, and you know how to deal with it. So, do NOT catch every exception, the exception should be returned to call stack and let application code to handle it properly.

### Recovering Gracefully from an Exception

When you catch specific exceptions, fully understand the circumstances that cause the exception, and know what exception types are derived. Don’t catch and handle **System.Exception** because it is impossible to know all kinds of exception types that could be thrown in **try** block.

### Maintaining State When an Unrecoverable Exception Occurs

Usually, methods call several other methods to perform a single abstract operation. Some of the individual methods might complete successfully, and some might not. 

For example, let’s say that you’re serializing a set of objects to a disk file. After serializing 10 objects, an exception is thrown. At this point, the exception should filter up to the caller, but the file is now corrupted because it contains a partially serialized object. It would be great if the application could back out of the partially completed operation.

```c#
public void SerializeObjectGraph(FileStream fs, IFormatter formatter, Object rootObj) {
    // Save the current position of the file.
    Int64 beforeSerialization = fs.Position;

    try {
        // Attempt to serialize the object graph to the file.
        formatter.Serialize(fs, rootObj);
    }
    catch {  // Catch any and all exceptions.
        // If ANYTHING goes wrong, reset the file back to a good state.
        fs.Position = beforeSerialization;
        // Truncate the file.
        fs.SetLength(fs.Position);

        // NOTE: The preceding code isn't in a finally block because
        // the stream should be reset only when serialization fails.
        
        // Let the caller(s) know what happened by re­throwing the SAME exception.
        throw; 
    }
}
```

### Re-throw Exception
Sometimes, we need to catch one exception and re-throw another exception. The new exception type that you throw should be a specific exception.

```c#
public String GetPhoneNumber(String name) {
    String phone;
    FileStream fs = null;
    try {
        fs = new FileStream(m_pathname, FileMode.Open);
        // Code to read from fs until name is found goes here
        phone = /* the phone # found */
    }
    catch (FileNotFoundException e) {
        // Throw a different exception containing the name, and
        // set the originating exception as the inner exception.
        throw new NameNotFoundException(name, e);
    }
    catch (IOException e) {
        // Throw a different exception containing the name, and
        // set the originating exception as the inner exception.
        throw new NameNotFoundException(name, e);
    }
    finally {
        if (fs != null) fs.Close();
    }
    return phone;
}
```

Throwing an exception still lets the caller know that the method cannot complete its task, and the **NameNotFoundException** type gives the caller an abstracted view as to why. Setting the inner exception to *FileNotFoundException* or *IOException* is important so that the real cause of the exception isn’t lost.

## Unhandled Exceptions

When an exception is thrown, CLR climbs up the call stack looking for matching catch blocks. If no catch block matched the exception type, an unhandled exception occurs. When CLR detects an unhandled exception, it will terminates the process.

Class library developers should not think about unhandled exceptions. Only application developers need to concern themselves with unhandled exceptions.
