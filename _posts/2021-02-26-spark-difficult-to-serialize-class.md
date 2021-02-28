---
title: "Apache Spark: difficult to serialize (Proto) Java Class in a Dataset"
categories:
  - Code
tags:
  - spark
  - java
  - protobuf
---

Have an issue with moving around an odd class in an Apache Spark Dataset? Here is a practical solution!

{% include toc %}

## Problem

In an [Apache Spark](https://spark.apache.org/) Dataset sometimes you *must* carry around a Java Class that is impossible to serialize (eg: a Proto Object) or that you can't simply modify (eg: it is part of an external package).
You might have tried the Kryo serializer, but with no luck.
There are actually 2 solutions to this:

* the good old Java Serialization, with a little twist, and
* an Apache Spark user defined type.

## The Unserializable class

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

    @Override
    public String toString() {
        return "DifficultToSerializeClass{" +
                "value='" + value + '\'' +
                ", flag=" + flag +
                '}';
    }

}
```

## Java Serialization

### The Workaround

To carry around an unserializable type in a Spark Dataset the only thing we need to do is to use a wrapper.
The wrapper will take care of transforming our troubled object to a byte array (and back again).
This can be achieved by making our wrapper `Serializable` and provide 2 methods: `readObject` and `writeObject`.
These two methods will be used to run around the default Java Serializer behavior and fully take charge of reading and writing the `DifficultToSerializeClass` type.

Such a wrapper, for our example, will look like this:

```java
public static class WrapperBean implements Serializable {

    private DifficultToSerializeClass difficultToSerializeObject;

    public WrapperBean(DifficultToSerializeClass difficultToSerializeObject) {
        this.difficultToSerializeObject = difficultToSerializeObject;
    }

    private void readObject(ObjectInputStream aInputStream) 
        throws ClassNotFoundException, IOException {
        // stream -> java
        difficultToSerializeObject = new DifficultToSerializeClass(
                aInputStream.readUTF(),
                aInputStream.readBoolean()
        );
    }

    private void writeObject(ObjectOutputStream aOutputStream) 
        throws IOException {
        // java -> stream
        // replace your preferred serializer, like a Proto object serializer
        aOutputStream.writeUTF(difficultToSerializeObject.getValue());
        aOutputStream.writeBoolean(difficultToSerializeObject.isFlag());
    }

    public DifficultToSerializeClass getWrappedObject() {
        return difficultToSerializeObject;
    }
        
}
```

### How it looks in a Dataset

Our Apache Spark Dataset will finally work after the problem class is wrapped in the `WrapperBean` type.
A sample code will look like this:

```java
var ds = sparkSession
    .createDataset(List.of(
        new WrapperBean(new DifficultToSerializeClass("one", true)),
        new WrapperBean(new DifficultToSerializeClass("two", true)),
        new WrapperBean(new DifficultToSerializeClass("three", false))
    ), 
    Encoders.javaSerialization(WrapperBean.class));
```

`DifficultToSerializeClass` will be properly encoded/decoded.  
The *downside*? The dataset will be a byte array if queried, and it will be be a single column named `value`.

```java
ds.printSchema();
ds.show();

// STDOUT -------------------------

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

But a mapper will be able to read it as expected:

```java
ds.map((MapFunction<WrapperBean, Integer>)m -> {
            return m.getWrappedObject().getValue().length();
        }, 
        Encoders.INT())
    .show();

// STDOUT -------------------------

+-----+
|value|
+-----+
|    3|
|    3|
|    5|
+-----+
```

## Spark User Defined Type

In this scenario, we need to define a `UserDefinedType` and register it. Its use is more tied to operating on a `DataFrame`.

### Define a new type

Our new type declaration does the same things as the `WrapperBean` was doing before: define how we serialize/deserialize an object of a given type.
This new user defined type is fairly simple:

```java
public class UDT extends UserDefinedType<DifficultToSerializeClass> {

    @Override
    public DataType sqlType() {
        return DataTypes.BinaryType;
    }

    @Override
    public Object serialize(DifficultToSerializeClass obj) {
        try {
            var bos = new ByteArrayOutputStream();
            try (var oos = new ObjectOutputStream(bos)) {
                oos.writeUTF(obj.getValue());
                oos.writeBoolean(obj.isFlag());
            }
            return bos.toByteArray();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public DifficultToSerializeClass deserialize(Object datum) {
        try {
            var bis = new ByteArrayInputStream((byte[]) datum);
            var ois = new ObjectInputStream(bis);
            return new DifficultToSerializeClass(
                    ois.readUTF(),
                    ois.readBoolean()
            );
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public Class<DifficultToSerializeClass> userClass() {
        return DifficultToSerializeClass.class;
    }

}
```

For our convenience, we will also define a new simple type to hold our `DifficultToSerializeClass` object.  
This wrapper is much simpler than the `WrapperBean` class (defined above) since it will not need to define how we serialize its internal fields.
This new bean also an advantage: it can add new fields and they will be visible to Spark SQL:

```java
public class CarrierBean {

    private final DifficultToSerializeClass bean;
    private final Long longCounter;

    public CarrierBean(DifficultToSerializeClass bean, Long longCounter) {
        this.bean = bean;
        this.longCounter = longCounter;
    }

    public DifficultToSerializeClass getBean() {
        return bean;
    }

    public Long getLongCounter() {
        return longCounter;
    }

}
```

The final step is to register the new type in the Spark type registry:

```java
UDTRegistration.register(
    DifficultToSerializeClass.class.getName(), 
    UDT.class.getName());
```

### How it looks in a DataFrame

As expected, the problem class is usable in our `CarrierBean` as a field.  
But also it is usable in every possible situation where the type is used, since the type is managed directly by Spark.
The integration code will be:

```java
UDTRegistration.register(
    DifficultToSerializeClass.class.getName(), 
    UDT.class.getName());

var data = List.of(
        new CarrierBean(new DifficultToSerializeClass("one", true), 1L),
        new CarrierBean(new DifficultToSerializeClass("two", true), 2L),
        new CarrierBean(new DifficultToSerializeClass("three", false), 3L)
);

var ds = sparkSession.createDataFrame(data, CarrierBean.class);

ds.map((MapFunction<Row, Integer>)row -> {
            return row.<DifficultToSerializeClass>getAs("bean").getValue().length();
        }, 
        Encoders.INT())
    .show();

ds.printSchema();
ds.show(1, 200);

// STDOUT -------------------------

+-----+
|value|
+-----+
|    3|
|    3|
|    5|
+-----+

root
 |-- bean:  (nullable = true)
 |-- longCounter: long (nullable = true)

+-------------------------------------------------+-----------+
|                                             bean|longCounter|
+-------------------------------------------------+-----------+
|DifficultToSerializeClass{value='one', flag=true}|          1|
+-------------------------------------------------+-----------+
only showing top 1 row

```

And finally, as a bonus, the DataFrame can be converted to a typed Dataset.  
And the reason is that the `DifficultToSerializeClass` type is fully managed by Apache Spark, so no additional code is needed:

```java
ds.as(Encoders.bean(CarrierBean.class)).show();

// STDOUT -------------------------

+--------------------+-----------+
|                bean|longCounter|
+--------------------+-----------+
|DifficultToSerial...|          1|
|DifficultToSerial...|          2|
|DifficultToSerial...|          3|
+--------------------+-----------+
```
