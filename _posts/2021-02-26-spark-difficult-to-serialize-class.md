---
title: "Apache Spark: difficult to serialize Java Class in Dataset"
categories:
  - Code
tags:
  - spark
  - java
---

Have an issue with moving around an odd class in an Apache Spark Dataset? Here is a practical solution!

{% include toc %}

## Problem

In an Apache Spark Dataset sometimes you *must* carry around a Java Class that is impossible to serialize (eg: a Proto Object) or that you can't simply modify (eg: it is part of an external package).
You might have tried the Kryo serializer, but with no luck.
What will help you now is actually the good old Java Serialization, with a little twist!

## Our Problem class

Let say we have a problem class named `DifficultToSerializeClass`. 
This class may be a Proto class, or something that neither the Kryo nor the Java Serializer likes. 
In our example we will use the following class, and pretend that we can't serialize it using Kryo (or Java Serialization):
```java
public class DifficultToSerializeClass {

    private final String value;
    private final boolean flag;

    public DifficultToSerializeClass(String value, boolean flag) {
        this.value = value;
        this.flag = flag;
    }

    public String getValue() {
        return value;
    }

    public boolean isFlag() {
        return flag;
    }

}
```

## Workaround

To carry around an unserializable type in a Spark Dataset the only thing we need to do is to use a wrapper.
The wrapper will take care of transforming our troubled object to a byte array (and back again).
This can be achieved by making our wrapper both `Serializable` and also providing 2 methods: `readObject` and `writeObject`. 
These two methods will be used to run around the default Java Serializer behavior and fully take charge of reading and writing the `DifficultToSerializeClass` type.

Such a wrapper, for our example, will look like this:
```java
public static class WrapperBean implements Serializable {

    private DifficultToSerializeClass difficultToSerializeObject;

    public WrapperBean(DifficultToSerializeClass difficultToSerializeObject) {
        this.difficultToSerializeObject = difficultToSerializeObject;
    }

    private void readObject(ObjectInputStream aInputStream) throws ClassNotFoundException, IOException {
        // stream -> java
        difficultToSerializeObject = new DifficultToSerializeClass(
                aInputStream.readUTF(),
                aInputStream.readBoolean()
        );
    }

    private void writeObject(ObjectOutputStream aOutputStream) throws IOException {
        // java -> stream, replace your preferred serializer, like a Proto object serializer
        aOutputStream.writeUTF(difficultToSerializeObject.getValue());
        aOutputStream.writeBoolean(difficultToSerializeObject.isFlag());
    }

    public DifficultToSerializeClass getWrappedObject() {
        return difficultToSerializeObject;
    }
        
}
```

## How all will come together


Our Apache Spark Dataset will work after the problem class is wrappaed in the `WrapperBean` type. 
A sample code will look like this:
```java
var ds = sparkSession
    .createDataset(List.of(
        new WrapperBean(new DifficultToSerializeClass("one", true)),
        new WrapperBean(new DifficultToSerializeClass("two", true)),
        new WrapperBean(new DifficultToSerializeClass("three", false))
    ), Encoders.javaSerialization(WrapperBean.class));
```

`DifficultToSerializeClass` will be properly encoded/decoded. 
The *downside*? The dataset will be a byte array if queried, and it will be be a single column named `value`.
```java
ds.printSchema();
ds.show();
```

Output:
```
root
 |-- value: binary (nullable = true)

+--------------------+
|               value|
+--------------------+
|[AC ED 00 05 73 7...|
|[AC ED 00 05 73 7...|
|[AC ED 00 05 73 7...|
+--------------------+
```

But a mapper will be able to read it as expected ... TBD
