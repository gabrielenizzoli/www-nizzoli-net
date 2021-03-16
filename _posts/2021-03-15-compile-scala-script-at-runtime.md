---
title: "Compile scala script at runtime"
categories:
  - Software
tags:
  - scala
---

A useful trick: to be able to define code at runtime and use it in a scala/java class!

First we need our code to have the proper scala packages: 

```xml
<dependencies>
  <dependency>
    <groupId>org.scala-lang</groupId>
    <artifactId>scala-compiler</artifactId>
  </dependency>
  <dependency>
    <groupId>org.scala-lang</groupId>
    <artifactId>scala-reflect</artifactId>
  </dependency>
</dependencies>
```

Then we need to have a compiler utility:

```scala
import java.util.concurrent.ConcurrentHashMap
import scala.reflect.runtime.universe
import scala.tools.reflect.ToolBox

object ScriptEngine {

  val cache = new ConcurrentHashMap[String, Any]();

  def compileTo[T](code: String): T = {
    val toolBox = universe.runtimeMirror(getClass.getClassLoader).mkToolBox()
    val outcome = toolBox.eval(toolBox.parse(code))
    outcome.asInstanceOf[T]
  }

}
```

And finally we can compile and use:

```java
val function = ScriptEngine.compileTo[(Int)=>Int]("(i:Int)=>i+1")
// evaluates to 2!
val outcome = function(1)
```

And that is it :)

NOTE: output object can't be used in a serializer/deserializer process, since class in not (usually) available in deserializer class loader (class is defined in the script, into its own class loader).

