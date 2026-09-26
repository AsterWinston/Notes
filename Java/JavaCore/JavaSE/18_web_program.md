# 1.InetAddress（IP地址）

```
InetAddress 类位于 java.net 包，表示 IP 地址对象。

1.1 获取 InetAddress 对象
  public static InetAddress getByName(String host)
    根据主机名（如 "www.baidu.com"）或 IP 地址字符串（如 "192.168.1.1"）获取对象。
  
  public static InetAddress getLocalHost()
    获取本机 IP 地址对象。
  
  public static InetAddress[] getAllByName(String host)
    获取指定主机名的所有 IP 地址（一个域名可能对应多个 IP）。

1.2 常用方法
  public String getHostName()        // 获取主机名
  public String getHostAddress()     // 获取 IP 地址（字符串形式，如 "192.168.1.1"）
  public byte[] getAddress()         // 获取原始 IP 地址（字节数组）
  public boolean isReachable(int timeout)  // 判断是否可达（ping 检测）
  public boolean isLoopbackAddress() // 判断是否为回环地址（127.0.0.1）
  public boolean isSiteLocalAddress() // 判断是否为私有地址（局域网）
  public boolean isMulticastAddress() // 判断是否为多播地址
```

# 2.UDP 通信（DatagramSocket / DatagramPacket）

```
UDP（User Datagram Protocol）是无连接的通信协议，发送数据不保证到达，不保证顺序。
速度快、开销小，适用于实时通信（视频直播、语音通话等）。

核心类：
  DatagramSocket：用于发送和接收 UDP 数据包的套接字
  DatagramPacket：数据报包，封装了数据和目标地址
```

## 2.1DatagramSocket（UDP 套接字）

```
2.1.1 构造方法
  public DatagramSocket()                                // 绑定到任意可用端口
  public DatagramSocket(int port)                        // 绑定到指定端口
  public DatagramSocket(int port, InetAddress bindAddr)  // 绑定到指定端口和地址

2.1.2 常用方法
  public void send(DatagramPacket p)    // 发送数据包
  public void receive(DatagramPacket p) // 接收数据包（阻塞方法，直到收到数据）
  public void close()                   // 关闭套接字
  public int getPort()                  // 获取绑定的端口
  public void setSoTimeout(int millis)  // 设置接收超时（毫秒）
```

## 2.2DatagramPacket（UDP 数据包）

```
2.2.1 发送数据包的构造
  public DatagramPacket(byte[] buf, int offset, int length, InetAddress address, int port)
    创建用于发送的数据包：指定数据、长度、目标 IP 和端口。

  public DatagramPacket(byte[] buf, int length, InetAddress address, int port)
    简化版发送数据包。

2.2.2 接收数据包的构造
  public DatagramPacket(byte[] buf, int length)
    创建用于接收的空数据包：指定缓冲区和长度。

2.2.3 常用方法
  public byte[] getData()         // 获取数据缓冲区
  public int getLength()          // 获取实际收到的数据长度
  public int getOffset()          // 获取偏移量
  public InetAddress getAddress() // 获取发送方 IP
  public int getPort()            // 获取发送方端口
  public void setData(byte[] buf) // 设置数据

2.2.4 完整示例 — UDP 发送端：
  DatagramSocket ds = new DatagramSocket();
  String msg = "Hello UDP!";
  byte[] data = msg.getBytes();
  InetAddress ip = InetAddress.getByName("127.0.0.1");
  DatagramPacket dp = new DatagramPacket(data, data.length, ip, 8888);
  ds.send(dp);
  ds.close();

2.2.5 完整示例 — UDP 接收端：
  DatagramSocket ds = new DatagramSocket(8888);
  byte[] buf = new byte[1024];
  DatagramPacket dp = new DatagramPacket(buf, buf.length);
  ds.receive(dp);  // 阻塞等待数据
  String msg = new String(dp.getData(), 0, dp.getLength());
  System.out.println("收到: " + msg + "，来自: " + dp.getAddress());
  ds.close();
```

## 2.3UDP 组播（MulticastSocket）

```
MulticastSocket 继承自 DatagramSocket，用于实现组播通信。
组播地址范围为 224.0.0.0 ~ 239.255.255.255。

发送端：
  MulticastSocket ms = new MulticastSocket();
  byte[] data = "Hello Multicast".getBytes();
  InetAddress group = InetAddress.getByName("224.0.0.1");
  DatagramPacket dp = new DatagramPacket(data, data.length, group, 8888);
  ms.send(dp);
  ms.close();

接收端：
  MulticastSocket ms = new MulticastSocket(8888);
  ms.joinGroup(InetAddress.getByName("224.0.0.1"));  // 加入组播组
  byte[] buf = new byte[1024];
  DatagramPacket dp = new DatagramPacket(buf, buf.length);
  ms.receive(dp);
  System.out.println(new String(dp.getData(), 0, dp.getLength()));
  ms.leaveGroup(InetAddress.getByName("224.0.0.1"));
  ms.close();
```

# 3.TCP 通信（Socket / ServerSocket）

```
TCP（Transmission Control Protocol）是面向连接的协议，提供可靠的数据传输。
保证数据按序到达、不丢失、不重复。适用于要求可靠传输的场景（文件传输、网页浏览等）。

核心类：
  Socket：TCP 客户端套接字
  ServerSocket：TCP 服务器端套接字
```

## 3.1Socket（TCP 客户端）

```
3.1.1 构造方法
  public Socket(String host, int port)
    创建 Socket 并连接到指定主机和端口。

  public Socket(InetAddress address, int port)
    通过 InetAddress 对象连接。

3.1.2 常用方法
  public InputStream getInputStream()    // 获取输入流（从服务端读取数据）
  public OutputStream getOutputStream()  // 获取输出流（向服务端发送数据）
  public void close()                    // 关闭连接（同时关闭 IO 流）
  public void shutdownOutput()           // 禁用输出流（告知服务端发送完毕）
  public void shutdownInput()            // 禁用输入流
  public InetAddress getInetAddress()    // 获取远程 IP
  public int getPort()                   // 获取远程端口
  public int getLocalPort()              // 获取本地端口
  public boolean isConnected()           // 是否已连接
  public boolean isClosed()              // 是否已关闭
  public void setSoTimeout(int millis)   // 设置读取超时
```

## 3.2ServerSocket（TCP 服务器端）

```
3.2.1 构造方法
  public ServerSocket(int port)
    在指定端口上创建服务器套接字。

  public ServerSocket(int port, int backlog)
    backlog：等待队列的最大长度。

  public ServerSocket(int port, int backlog, InetAddress bindAddr)
    绑定到指定端口和 IP。

3.2.2 常用方法
  public Socket accept()          // 监听客户端连接（阻塞），返回与客户端通信的 Socket
  public void close()             // 关闭服务器套接字
  public int getLocalPort()       // 获取绑定的端口
  public boolean isClosed()       // 是否已关闭
```

## 3.3TCP 通信完整示例

```
3.3.1 单次通信 — 服务端：
  ServerSocket server = new ServerSocket(9999);
  System.out.println("服务器已启动，等待连接...");
  Socket socket = server.accept();  // 阻塞等待客户端

  // 获取输入流（读客户端发来的数据）
  BufferedReader reader = new BufferedReader(
      new InputStreamReader(socket.getInputStream()));
  String msg = reader.readLine();
  System.out.println("收到: " + msg);

  // 获取输出流（发送响应）
  PrintWriter writer = new PrintWriter(socket.getOutputStream(), true);
  writer.println("服务端已收到: " + msg);

  socket.close();
  server.close();

3.3.2 单次通信 — 客户端：
  Socket socket = new Socket("127.0.0.1", 9999);

  // 发送数据
  PrintWriter writer = new PrintWriter(socket.getOutputStream(), true);
  writer.println("Hello, Server!");

  // 接收响应
  BufferedReader reader = new BufferedReader(
      new InputStreamReader(socket.getInputStream()));
  System.out.println("服务端响应: " + reader.readLine());

  socket.close();

3.3.3 多客户端服务端（多线程）：
  ServerSocket server = new ServerSocket(9999);
  while (true) {
      Socket socket = server.accept();
      new Thread(() -> {
          try {
              // 处理客户端...（同单次通信逻辑）
              socket.close();
          } catch (IOException e) {
              e.printStackTrace();
          }
      }).start();
  }
```

## 3.4TCP 文件上传示例

```
服务端：
  ServerSocket server = new ServerSocket(9999);
  Socket socket = server.accept();

  BufferedInputStream bis = new BufferedInputStream(socket.getInputStream());
  BufferedOutputStream bos = new BufferedOutputStream(
      new FileOutputStream("uploaded_file.txt"));

  byte[] buf = new byte[8192];
  int len;
  while ((len = bis.read(buf)) != -1) {
      bos.write(buf, 0, len);
  }

  // 响应客户端
  PrintWriter pw = new PrintWriter(socket.getOutputStream(), true);
  pw.println("上传成功");

  bos.close();
  socket.close();
  server.close();

客户端：
  Socket socket = new Socket("127.0.0.1", 9999);

  BufferedInputStream bis = new BufferedInputStream(
      new FileInputStream("file_to_upload.txt"));
  BufferedOutputStream bos = new BufferedOutputStream(socket.getOutputStream());

  byte[] buf = new byte[8192];
  int len;
  while ((len = bis.read(buf)) != -1) {
      bos.write(buf, 0, len);
  }
  bos.flush();
  socket.shutdownOutput();  // 告知服务端发送完毕

  // 接收响应
  BufferedReader br = new BufferedReader(
      new InputStreamReader(socket.getInputStream()));
  System.out.println(br.readLine());

  bis.close();
  socket.close();
```

# 4.URL 和 URLConnection

```
URL（Uniform Resource Locator）统一资源定位符，用于定位网络上的资源。
```

## 4.1URL 类

```
4.1.1 构造方法
  URL(String spec)           // 例如：new URL("https://www.example.com/file.txt")
  URL(String protocol, String host, int port, String file)  // 指定各组成部分

4.1.2 常用方法
  String getProtocol()       // 获取协议（http, https, ftp 等）
  String getHost()           // 获取主机名
  int getPort()              // 获取端口号（未指定返回 -1）
  String getPath()           // 获取路径部分
  String getFile()           // 获取文件名（路径+参数+锚点）
  String getQuery()          // 获取查询参数（? 后面的部分）
  String getRef()            // 获取锚点（# 后面的部分）
  URLConnection openConnection()  // 打开连接，返回 URLConnection
  InputStream openStream()   // 直接打开输入流（简化版 openConnection().getInputStream()）
```

## 4.2URLConnection 类

```
4.2.1 获取方式
  URL url = new URL("https://www.example.com/api");
  URLConnection conn = url.openConnection();  // 返回 HttpURLConnection（HTTP 时）

4.2.2 常用方法
  void connect()             // 打开通信链接（通常不需要显式调用）
  InputStream getInputStream()   // 获取输入流（用于读取响应）
  OutputStream getOutputStream() // 获取输出流（用于 POST 请求写入参数）
  String getContentType()    // 获取内容类型（MIME type）
  int getContentLength()     // 获取内容长度
  long getLastModified()     // 获取最后修改时间
  void setRequestProperty(String key, String value)  // 设置请求头
  String getHeaderField(String name)  // 获取响应头
  Map<String, List<String>> getHeaderFields()  // 获取所有响应头

4.2.3 HTTP 请求示例（GET）：
  URL url = new URL("https://api.example.com/data");
  HttpURLConnection conn = (HttpURLConnection) url.openConnection();
  conn.setRequestMethod("GET");

  int code = conn.getResponseCode();  // 200 表示成功
  if (code == 200) {
      BufferedReader br = new BufferedReader(
          new InputStreamReader(conn.getInputStream()));
      String line;
      while ((line = br.readLine()) != null) {
          System.out.println(line);
      }
      br.close();
  }
  conn.disconnect();

4.2.4 HTTP 请求示例（POST 发送 JSON）：
  URL url = new URL("https://api.example.com/submit");
  HttpURLConnection conn = (HttpURLConnection) url.openConnection();
  conn.setRequestMethod("POST");
  conn.setRequestProperty("Content-Type", "application/json");
  conn.setDoOutput(true);  // 允许发送数据

  String json = "{\"name\":\"张三\",\"age\":25}";
  try (OutputStream os = conn.getOutputStream()) {
      os.write(json.getBytes(StandardCharsets.UTF_8));
  }

  // 读取响应...
  conn.disconnect();
```
