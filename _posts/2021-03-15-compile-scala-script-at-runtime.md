---
title: "Compile scala script at runtime"
categories:
  - Code
tags:
  - scala
  - java
---

A useful trick: to be able to compile code at runtime and use its result(s) in a scala/java class!

First we need the proper scala dependencies: 

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

Then we need to have a compiler utility (written in scala):

```scala
import java.util.concurrent.ConcurrentHashMap
import scala.reflect.runtime.universe
import scala.tools.reflect.ToolBox

object ScriptEngine {

  val cache = new ConcurrentHashMap[String, Any]();

  def compile(code: String): Any = {
    val toolBox = universe.runtimeMirror(getClass.getClassLoader).mkToolBox()
    toolBox.eval(toolBox.parse(code))
  }

}
```

And finally we can compile and use:

```scala
val function = ScriptEngine.compile("(i:Int)=>i+1").asInstanceOf[(Int)=>Int]
// evaluates to 2!
val outcome = function(1)
```

And that is it :)

*NOTE: output object can't be used in a serializer/deserializer process, since class in not (usually) available in deserializer class loader (class is defined in the script, into its own class loader).*

