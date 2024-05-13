DataInput&DataOutput

java IO包中提供了DataInput和DataOutput接口，DataInput接口提供了提供用于读取二进制流字节并重建为任何java原始数据类型，也提供了一个转换为UTF-8编码的字符串类型方法；DataOutput 接口用于将数据从任意 Java 基本类型转换为一系列字节，并将这些字节写入二进制流。同时还提供了一个将 String 转换成 UTF-8 修改版格式并写入所得到的系列字节的工具。

### DataInput

```java


package java.io;


public
interface DataInput {
    //从输入流中读取字节并存入到字节缓冲区b数组中
    void readFully(byte b[]) throws IOException;

    //从输入流中读取字节并存入到字节缓冲区b数组中，从off位置开始读取,读取len长度的字节
    void readFully(byte b[], int off, int len) throws IOException;

    //跳过n个字节的数据
    int skipBytes(int n) throws IOException;

    //读取一个字节，如果这个字节不是0,则返回true,否则返回false;
    boolean readBoolean() throws IOException;

    //读取并返回一个输入字节，该字节对应的值在-128到127之间
    byte readByte() throws IOException;

    //读取一个字节，返回无符号的int的值，值在0到255的范围之间
    int readUnsignedByte() throws IOException;

    //读取两个输入字节，并返回一个short类型的值，如果读取的第一、第二字节分别为a,b 则返回为(short)((a << 8) | (b & 0xff))
    short readShort() throws IOException;

    //读取两个输入字节，并返回一个无符号的short类型的值，如果读取的第一、第二字节分别为a,b 则返回为(((a & 0xff) << 8) | (b & 0xff))
    int readUnsignedShort() throws IOException;

    //读取两个输入字节，并返回一个char类型的值，如果读取的第一、第二字节分别为a,b 则返回为(char)((a << 8) | (b & 0xff))
    char readChar() throws IOException;

	//读取四个字节并返回一个int类型的值，如果读取的第一、第二，第三，第四字节分别为a,b,c,d 则返回为(((a & 0xff) << 24) | ((b & 0xff) << 16) | ((c & 0xff) << 8) | (d & 0xff))
    int readInt() throws IOException;

    //读取八个字节并返回一个long数据，如a-h为读取到的八字节，结果会返回(((long)(a & 0xff) << 56) |  ((long)(b & 0xff) << 48) |  ((long)(c & 0xff) << 40) |  ((long)(d & 0xff) << 32) |  ((long)(e & 0xff) << 24) |  ((long)(f & 0xff) << 16) |  ((long)(g & 0xff) <<  8) | ((long)(h & 0xff)))
    long readLong() throws IOException;

    //读取四个输入字节并返回一个float类型数据
    float readFloat() throws IOException;

    //读取八个如数字节并返回一个double数据类型
    double readDouble() throws IOException;

    //从输入流读取下一行文本，它读取连续字节，将每个字节分别转换为字符，直到遇到行终止符或文件结束符为止
    String readLine() throws IOException;


	//读取一个字符串,使用UTF-8编码格式。这些字符将会作为字符串返回
    String readUTF() throws IOException;
}

```

### DataOutput

```java
package java.io;

public
interface DataOutput {
	//取传入的int行数据的低8位数据写入到输出流中，高24位忽略
    void write(int b) throws IOException;

    //将传入的字节数组中的所有数据都写入输出流之中，b为null则抛出空指针异常。
    void write(byte b[]) throws IOException;

    //将传入的字节数组从off位置开始往后len长度的数据写入到输出流
    void write(byte b[], int off, int len) throws IOException;

    //该方法向输出流中写入一个boolean类型的值，如果传入的boolean型参数v是true，那么向流中写入1，如果传入的boolean型参数v是false，那么向流中写入0
    void writeBoolean(boolean v) throws IOException;

    //该方法将截取传入参数的低八位写入输出流中，高24位将被忽略
    void writeByte(int v) throws IOException;

    //向输出流中写入两个字节的数据，通过(byte)(0xff & (v >> 8))和(byte)(0xff & v)两个小操作来分别得到需要写入的数据，即int型数据后16位中的高8位和低八位.通过该方法向输出流中写入的Short型数据，需要通过DataInput接口中的readShort方法来获取
    void writeShort(int v) throws IOException;

    //该方法向输出流中写入一个包含两个字节的char型数据，通过(byte)(0xff & (v >> 8))和(byte)(0xff & v)两个小操作来得到需要写入的数据。通过该方法向输出流中写入的char型数据
    void writeChar(int v) throws IOException;

    //该方法向输出流中写入一个包含四个字节的int型数据，通过(byte)(0xff & (v >> 24))、(byte)(Oxff & (v >> 16))、(byte)(Oxff & (v >> 8))和(byte)(Oxff & v)四个操作来得到需要写入的数据
    void writeInt(int v) throws IOException;

    //该方法向输出流中写入一个包含八个字节的long型数据，通过(byte)(0xff & (v >> 56))、(byte)(0xff & (v >> 48))、(byte)(0xff & (v >> 40))、(byte)(0xff & (v >> 32))、(byte)(0xff & (v >> 24))、(byte)(0xff & (v >> 16))、(byte)(0xff & (v >>  8))和(byte)(0xff & v)八个小操作来得到需要写入数据。通过该方法向输出流中写入的int型数据
    void writeLong(long v) throws IOException;

    //向流中写入一个包含4个字节的float型数据
    void writeFloat(float v) throws IOException;

    //向流中写入一个包含8个字节的double型数据
    void writeDouble(double v) throws IOException;

    //用于将传入String类型s中的每个字符写入输出流中
    void writeBytes(String s) throws IOException;

    //将传入String类型s中的每个字符写入输出流中
    void writeChars(String s) throws IOException;

    //该方法将传入的字符串s以utf-8的编码格式写入输出流中，通过该方法向输出流中写入的数据可以通过DataInput接口中的readUTF来进行读取。
    void writeUTF(String s) throws IOException;
}

```

