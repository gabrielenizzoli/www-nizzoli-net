---
title: "Apache Spark: difficult to serialize Java Class in Dataset"
categories:
  - Code
tags:
  - spark
  - java
---

{% include toc %}

## Problem

In an Apache Spark Dataset, sometimes you need to carry aroubd a Java Class that is difficlut to serialize (a Proto Object) or that you can't simply modify (since it is part of an external package).
You might have tried the Kryo serializer, but with no luck.
What will help you now is actually the good old Java Serialization, with a littel help from a custom serializer!

### Our Problem class

Let say we have a problem class. 
This class may be a Proto object class, or something that neither the Kryo nor the Java serializer likes. 
In our dummy example we will use the following class, and pretend that we can'ty serialize it using Kryo or Java serialization:
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

### Workaround
To make sure we canm carry around in a Spark Dataset, the only thing we need to do is to warp it a class that will take care of transforming our difficult object in a bayte array and back again.
Such a wrapper will look like this:
```java
public static class WrapperBean implements Serializable {

    private DifficultToSerializeClass difficultToSerializeObject;

    public WrapperBean(DifficultToSerializeClass difficultToSerializeObject) {
        this.difficultToSerializeObject = difficultToSerializeObject;
    }

    private void readObject(ObjectInputStream aInputStream) throws ClassNotFoundException, IOException {
        difficultToSerializeObject = new DifficultToSerializeClass(
                aInputStream.readUTF(),
                aInputStream.readBoolean()
        );
    }

    private void writeObject(ObjectOutputStream aOutputStream) throws IOException {
        aOutputStream.writeUTF(difficultToSerializeObject.getValue());
        aOutputStream.writeBoolean(difficultToSerializeObject.isFlag());
    }

    public DifficultToSerializeClass getWrappedObject() {
        return difficultToSerializeObject;
    }
        
}
```
