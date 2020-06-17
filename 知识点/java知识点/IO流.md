### IO流种类

#### 字节流

- ##### 字节输入流 InputStream

  - FileInputStream 操作文件的字节输入流
  - BufferedInputStream高效的字节输入流

- ##### 字节输出流 OutputStream

  - FileOutputStream 操作文件的字节输出流
  - BufferedOutputStream 高效的字节输出流

#### 字符流

- ##### 字符输入流 Reader

  - FileReader 操作文件的字符输入流
  - BufferedReader 高效的字符输入流
  - InputStreamReader 输入操作的转换流(把字节流封装成字符流)

- ##### 字符输出流 Writer

  - FileWriter 操作文件的字符输出流
  - ufferedWriter 高效的字符输出流
  - OutputStreamWriter 输出操作的转换流(把字节流封装成字符流)

方法：

- 读数据方法：

  - read() 一次读一个字节或字符的方法
  - read(byte[]  char[]) 一次读一个数组数据的方法
  - readLine() 一次读一行字符串的方法(BufferedReader类特有方法)
  - readObject() 从流中读取对象(ObjectInputStream特有方法)

- 写数据方法：

  - write(int) 一次写一个字节或字符到文件中

  - write(byte[] char[]) 一次写一个数组数据到文件中

  - write(String) 一次写一个字符串内容到文件中

  - writeObject(Object ) 写对象到流中(ObjectOutputStream类特有方法)

  - newLine() 写一个换行符号(BufferedWriter类特有方法)

    

BufferedInputStream  读取流中的字节流数据，用于读取文件

BufferedOutputStream 把数据写入流中，用于写入文件

常用方式

```java
File file = new File(Environment.getExternalStorageState(), "test");
try {
    BufferedInputStream bis = new BufferedInputStream(new FileInputStream(file));
    BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream(file));
    int count = -1;
    byte[] array = new byte[1024];//临时byte数组
    while ((count = bis.read(array)) != -1) {
        bos.write(array, 0, count);
    }
    bis.close();
    bos.close();
} catch (FileNotFoundException e) {
    e.printStackTrace();
} catch (IOException e) {
    e.printStackTrace();
}
```

真正对磁盘操作的是``FileInputStream``和``FileOutputStream``，``BufferedInputStream``和``BufferedOutputStream``的操作只是为其内部做了一个缓存。

``BufferedInputStream``和``BufferedOutputStream``两者内部用了一个8192（8K）长度的数组来做缓存区。

- 当读写的长度超过这个8K长度时，你会发现这个BufferedInputStream没有丝毫作用，因为他只是清空了buffer（第6行）然后直接调用``FileOutputStream``直接写入输入的长度
- 当读写的长度超过缓存的剩余长度（比如前面已经缓存了7K,这时候又读进来两K）,先把缓存写入输出流，再将数据写入缓存.
- 当读写长度低于缓存长度时，存入缓存，等到缓存满了或者结束了再进行一次磁盘读写操作。

```java
    public synchronized void write(byte b[], int off, int len) throws IOException {
        if (len >= buf.length) {
            /* If the request length exceeds the size of the output buffer,
               flush the output buffer and then write the data directly.
               In this way buffered streams will cascade harmlessly. */
            flushBuffer();
            out.write(b, off, len);
            return;
        }
        if (len > buf.length - count) {
            flushBuffer();
        }
        System.arraycopy(b, off, buf, count, len);
        count += len;
    }
```

重点：从上面代码中我们可以发现``BufferedInputStream``和``BufferedOutputStream``这两个主要适用于读单个字节（一个字节进行一次磁盘读开销太大）或者读取未知字节长度时使用，正常的文件拷贝读写其实并不需要这两者，因为你自己定义一个8K的`` byte[] array``时，产生的效果和用buffer类是一模一样的

很多文章写的例子都说用了buffer类比没用快十几倍，那是因为他们写的例子都用了一个1K的array来进行读写。例如一个100K的文件，用1K的array并只使用``FileInputStream``,那么读写需要200次，用了buffer类后只需要30次的样子。

但是如果你使用一个8K的array来操作，那么时间是一摸一样的。

```java
   public void copyNoBuffer(File file) throws IOException {
        FileInputStream in = new FileInputStream(file);
        FileOutputStream out = new FileOutputStream(Environment.getExternalStorageDirectory()+"/Download/nobuffer.txt");
        byte[] buf = new byte[8192];
        while (in.read(buf) != -1) {
            out.write(buf);
        }
        in.close();
        out.close();
    }
    public void copyBuffer(File file) throws IOException {
        BufferedInputStream in = new BufferedInputStream(new FileInputStream(file));
        BufferedOutputStream out = new BufferedOutputStream(new FileOutputStream(Environment.getExternalStorageDirectory()+"/Download/buffer.txt"));
        byte[] buf = new byte[1024];
        while (in.read(buf) != -1) {
            out.write(buf);
        }
        in.close();
        out.close();
    }
/*
 * 当copyNoBuffer的byte[] buf = new byte[1024];
 * 时间是：
 * copyNoBuffer：6
 * copyBuffer：2
 * 当copyNoBuffer的byte[] buf = new byte[8192];
 * 时间是：
 * copyNoBuffer：2
 * copyBuffer：2
 */

```

字符输入流的原理和功能同字节流类似。

``BufferedInputStream``这里用到了装设者设计模式，``BufferedInputStream``为装饰者类，``FileInputStream``为被装饰者类，前者的作用就是为了加强后者已有的功能，这里就是为了提高数据流的读写效率。

``BufferedInputStream``的构造方法定义：``public BufferedInputStream(InputStream in)``可以看出，``Buffered``可以装饰任何一个``InputSteam``的 子类。





