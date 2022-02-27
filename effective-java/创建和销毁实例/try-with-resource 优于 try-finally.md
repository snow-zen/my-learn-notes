# try-with-resource 优于 try-finally
#Java 

Java 类库中有许多资源类都必须通过调用 close 方法来手动关闭资源，例如 InputStream、OutputStream 或者 java.sql.Connection。

资源类实例手动关闭的过程中，必然会存在客户端忽略资源的关闭操作，造成严重的性能后果。即使这些资源中有许多都使用 finalize 方法作为“安全网”，但是效果并不理想。

例如，I/O 流的关闭：

```java
static void copy(String src, String dst) throws IOException {
    InputStream in = new FileInputStream(src);
    try {
        OutputStream out = new FileOutputStream(dst);
        try {
            byte[] buf = new byte[100];
            int n;
            while ((n = in.read(buf)) >= 0) {
                out.write(buf, 0, n);
            }
        } finally {
            out.close();
        }
    } finally {
        in.close();
    }
}
```

虽然例子中使用 try-finally 语句可以正确的关闭资源，但它也存在不足之处：如果在 try 块中抛出异常后，又再 finally 块中抛出异常，则会造成 finally 块中的异常被 try 块中的异常抑制，导致异常堆栈中没有关于第一个异常的记录，这会导致调试变得非常复杂。

Java 7 中引入 try-with-resource 语句，上述的问题就可以被解决。而要使用该语法自动关闭资源，资源类必须实现 AutoClosable 接口。

例如，使用 try-with-resource 关闭 I/O 流：

```java
static void copy(String src, String dst) throws IOException {
    try (InputStream in = new FileInputStream(src);
         OutputStream out = new FileOutputStream(dst)) {
        byte[] buf = new byte[100];
        int n;
        while ((n = in.read(buf)) >= 0) {
            out.write(buf, 0, n);
        }
    }
}
```

使用 try-with-resource 不仅使代码变得更加简洁，也更容易进行调试：如果在 try 块中抛出异常后，调用 close 方法也抛出异常时，则 try 块之后抛出的异常都会被禁止，保留最开始的异常。这些被禁止的异常还会打印在异常堆栈中，并标明它们都是被禁止的异常。

> 被禁止的异常可以通过调用 getSuppressed 方法进行访问。

在 try-with-resource 语句中还是可以通过 catch 子句进行捕获的，并且产生的异常会更加清晰和有价值。