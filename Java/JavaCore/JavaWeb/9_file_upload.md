# 1.文件上传基础

```
文件上传是指将客户端（浏览器）的文件传输到服务器的过程。

文件上传的三要素：
  1. 表单的 method 必须是 POST（GET 请求只能传字符串，且长度有限）
  2. 表单的 enctype 必须是 multipart/form-data（不能用默认的 application/x-www-form-urlencoded）
  3. 表单中必须有 <input type="file"> 文件选择控件

普通的表单（不传文件）：
  请求头 Content-Type: application/x-www-form-urlencoded
  请求体格式：username=zhangsan&password=123

文件上传的表单：
  请求头 Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxk
  请求体格式：每个字段被 boundary 分隔，文件内容以二进制形式传输

HTML 文件上传表单：

<form action="/oa/upload" method="post" enctype="multipart/form-data">
    用户名：<input type="text" name="username"><br/>
    文件：<input type="file" name="myfile"><br/>
    <input type="submit" value="上传">
</form>
```

# 2.Servlet 3.0 之前的文件上传方式

```
在 Servlet 3.0 之前，没有内置的文件上传支持，需要借助第三方库：

  1. Apache Commons FileUpload（最常用）
  
    引入 jar 包：
      commons-fileupload-1.4.jar
      commons-io-2.11.0.jar
  
    代码示例：
    
    import org.apache.commons.fileupload.*;
    import org.apache.commons.fileupload.disk.*;
    import org.apache.commons.fileupload.servlet.*;
    
    public void doPost(request, response) {
        // 判断是否是 multipart 请求
        boolean isMultipart = ServletFileUpload.isMultipartContent(request);
        
        if (isMultipart) {
            DiskFileItemFactory factory = new DiskFileItemFactory();
            ServletFileUpload upload = new ServletFileUpload(factory);
            
            List<FileItem> items = upload.parseRequest(request);
            for (FileItem item : items) {
                if (item.isFormField()) {
                    // 普通表单字段
                    String name = item.getFieldName();
                    String value = item.getString("UTF-8");
                } else {
                    // 文件字段
                    String fileName = item.getName();
                    // 保存到服务器
                    item.write(new File("D:/upload/" + fileName));
                }
            }
        }
    }

  2. SmartUpload（Model2 时代常用，目前已淘汰）
```

# 3.Servlet 3.0+ 内置文件上传（Part 接口）

```
从 Servlet 3.0（Tomcat 7+）开始，规范内置了文件上传支持，不再需要第三方库。

核心类和注解：
  @MultipartConfig   —— 标注在 Servlet 上，表示该 Servlet 支持文件上传
  Part 接口           —— 代表上传文件的一个部分
  request.getPart()   —— 获取指定 name 的 Part
  request.getParts()  —— 获取所有 Part

@MultipartConfig 注解的属性：
  location          String  上传文件的临时存储目录（不指定则用系统默认临时目录）
  maxFileSize       long    单个文件最大大小（字节），默认 -1（无限制）
  maxRequestSize    long    整个请求最大大小（字节），默认 -1（无限制）
  fileSizeThreshold int     文件大小超过此阈值时，写入磁盘而非内存（字节），默认 0
```

# 4.使用 Part 接口实现文件上传

```java
import jakarta.servlet.annotation.MultipartConfig;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.Part;
import java.io.IOException;

@WebServlet("/upload")
@MultipartConfig(
    maxFileSize = 1024 * 1024 * 10,       // 单个文件最大 10MB
    maxRequestSize = 1024 * 1024 * 50,    // 整个请求最大 50MB
    fileSizeThreshold = 1024 * 1024 * 2   // 超过 2MB 先写入临时文件再读
)
public class UploadServlet extends HttpServlet {
    
    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) 
            throws ServletException, IOException {
        
        request.setCharacterEncoding("UTF-8");
        response.setContentType("text/html;charset=UTF-8");
        
        // 1. 获取普通表单字段
        String username = request.getParameter("username");
        System.out.println("用户名：" + username);
        
        // 2. 获取上传的文件
        Part filePart = request.getPart("myfile");  // 参数是表单中 file 控件的 name
        
        if (filePart != null && filePart.getSize() > 0) {
            
            // 获取上传的文件名
            String fileName = getSubmittedFileName(filePart);
            
            // 获取文件大小
            long fileSize = filePart.getSize();
            
            // 获取文件的 Content-Type（MIME 类型）
            String contentType = filePart.getContentType();
            
            System.out.println("文件名：" + fileName);
            System.out.println("文件大小：" + fileSize + " bytes");
            System.out.println("文件类型：" + contentType);
            
            // 3. 保存文件到服务器
            // 方式一：直接保存
            String savePath = getServletContext().getRealPath("/upload/");
            File saveDir = new File(savePath);
            if (!saveDir.exists()) {
                saveDir.mkdirs();
            }
            filePart.write(savePath + fileName);
            
            // 方式二：重命名保存（防止文件名冲突）
            String newFileName = UUID.randomUUID().toString() + "_" + fileName;
            filePart.write(savePath + newFileName);
        }
        
        // 4. 处理多个文件上传
        for (Part part : request.getParts()) {
            if (part.getContentType() != null) {
                // 这是一个文件 Part
                String fileName = getSubmittedFileName(part);
                if (fileName != null && !fileName.isEmpty()) {
                    String savePath = getServletContext().getRealPath("/upload/");
                    part.write(savePath + System.currentTimeMillis() + "_" + fileName);
                }
            }
        }
        
        response.getWriter().print("文件上传成功！");
    }
    
    /**
     * 从 Part 中获取原始文件名
     * 因为 Part.getName() 返回的是表单控件 name，不是文件名
     */
    private String getSubmittedFileName(Part part) {
        // Content-Disposition: form-data; name="myfile"; filename="test.jpg"
        String header = part.getHeader("Content-Disposition");
        for (String token : header.split(";")) {
            token = token.trim();
            if (token.startsWith("filename")) {
                // 去掉 filename=" 和结尾的 "
                return token.substring(token.indexOf('=') + 2, token.length() - 1);
            }
        }
        return null;
    }
}
```

# 5.文件上传时的注意事项

```
1. 文件大小限制
   - @MultipartConfig 的 maxFileSize 和 maxRequestSize
   - Tomcat 的全局限制：CATALINA_HOME/conf/server.xml 中 Connector 的 maxPostSize 属性
   - 超出限制时会抛出 IllegalStateException

2. 文件名冲突
   - 多个用户可能上传同名文件
   - 解决方案：使用 UUID + 原始文件名
     String newName = UUID.randomUUID().toString() + "_" + originalName;

3. 目录分散
   - 如果一个目录下文件太多，会影响文件系统性能
   - 解决方案：按日期创建子目录，或根据文件名 hash 分散存储
     String dateDir = new SimpleDateFormat("yyyy/MM/dd").format(new Date());

4. 文件类型校验
   - 不要只信任文件扩展名（用户可以改后缀）
   - 检查文件的 MIME 类型（Content-Type）
   - 检查文件魔数（文件头字节）
   
   // 允许的文件类型白名单
   List<String> allowedTypes = Arrays.asList("image/jpeg", "image/png", "image/gif");
   if (!allowedTypes.contains(filePart.getContentType())) {
       response.getWriter().print("不支持的文件类型！");
       return;
   }

5. 文件名安全
   - 防止路径穿越攻击（文件名中包含 ../ 等）
   - 从文件名中提取纯文件名，丢弃路径部分
     String fileName = Paths.get(originalName).getFileName().toString();

6. 存储位置安全
   - 上传的文件不要放在 webapp 目录下（能被直接访问）
   - 放在 WEB-INF 目录下（受保护，不能直接访问）
   - 或者放在磁盘上的独立目录（需要程序提供下载接口）
```

# 6.文件下载

```
文件下载的关键是设置响应头 Content-Disposition。

方式一：超链接直接下载（适合静态资源，但会暴露文件真实路径）

<a href="/oa/upload/test.pdf">下载文件</a>
缺点：浏览器可能会直接打开（如PDF、图片），而不是触发下载。

方式二：Servlet 实现下载（推荐）

@WebServlet("/download")
public class DownloadServlet extends HttpServlet {
    
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) 
            throws ServletException, IOException {
        
        // 1. 获取要下载的文件名（通过请求参数传递）
        String fileName = request.getParameter("filename");
        
        // 2. 防止路径穿越攻击
        fileName = Paths.get(fileName).getFileName().toString();
        
        // 3. 确定文件在服务器上的真实路径
        String filePath = getServletContext().getRealPath("/WEB-INF/files/") + fileName;
        File file = new File(filePath);
        
        if (!file.exists()) {
            response.setContentType("text/html;charset=UTF-8");
            response.getWriter().print("文件不存在！");
            return;
        }
        
        // 4. 设置响应头，告诉浏览器这是一个下载文件
        response.setContentType(getServletContext().getMimeType(fileName));
        response.setHeader("Content-Disposition", "attachment;filename=" 
                + URLEncoder.encode(fileName, "UTF-8"));
        response.setHeader("Content-Length", String.valueOf(file.length()));
        
        // 5. 读取文件并输出到响应流
        try (FileInputStream fis = new FileInputStream(file);
             OutputStream os = response.getOutputStream()) {
            
            byte[] buffer = new byte[4096];
            int bytesRead;
            while ((bytesRead = fis.read(buffer)) != -1) {
                os.write(buffer, 0, bytesRead);
            }
        }
    }
}

Content-Disposition 的两种值：
  attachment;filename=xxx    —— 强制浏览器弹出下载对话框
  inline;filename=xxx        —— 浏览器尝试直接打开（如PDF在线预览）

中文文件名处理：
  不同浏览器对中文文件名的处理方式不同：
  需要根据 User-Agent 判断浏览器类型，使用不同的编码方式。
  通用方案：使用 URLEncoder.encode(filename, "UTF-8")
```

# 7.批量文件上传（多文件同时上传）

```html
<!-- 前端：多文件选择 -->
<form action="/oa/uploadBatch" method="post" enctype="multipart/form-data">
    文件1：<input type="file" name="myfile"><br/>
    文件2：<input type="file" name="myfile"><br/>
    文件3：<input type="file" name="myfile"><br/>
    <input type="submit" value="批量上传">
</form>

<!-- 或使用 multiple 属性 -->
<input type="file" name="myfile" multiple>
```

```java
@WebServlet("/uploadBatch")
@MultipartConfig(maxFileSize = 1024 * 1024 * 10, maxRequestSize = 1024 * 1024 * 100)
public class BatchUploadServlet extends HttpServlet {
    
    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) 
            throws ServletException, IOException {
        
        String savePath = getServletContext().getRealPath("/upload/");
        
        // 获取所有文件 Part（name 相同的文件控件，Part 接口会自动处理为多个 Part）
        for (Part part : request.getParts()) {
            // 获取原始文件名
            String header = part.getHeader("Content-Disposition");
            String fileName = extractFileName(header);
            
            if (fileName != null && !fileName.isEmpty()) {
                // 用 UUID 防止重名
                String newName = UUID.randomUUID().toString() + "_" + fileName;
                part.write(savePath + newName);
                System.out.println("上传成功：" + fileName + " → " + newName);
            }
        }
        
        response.setContentType("text/html;charset=UTF-8");
        response.getWriter().print("所有文件上传成功！");
    }
    
    private String extractFileName(String header) {
        for (String token : header.split(";")) {
            if (token.trim().startsWith("filename")) {
                return token.substring(token.indexOf('=') + 2, token.length() - 1);
            }
        }
        return null;
    }
}
```

# 8.文件上传 + AJAX（无刷新上传）

```html
<!-- 前端：使用 FormData 实现无刷新上传 -->
<input type="file" id="fileInput" multiple>
<button onclick="uploadFiles()">上传</button>
<div id="progress"></div>

<script type="text/javascript">
    function uploadFiles() {
        var files = document.getElementById("fileInput").files;
        var formData = new FormData();
        
        // 将文件添加到 FormData
        for (var i = 0; i < files.length; i++) {
            formData.append("myfile", files[i]);
        }
        
        var xhr = new XMLHttpRequest();
        
        // 监听上传进度
        xhr.upload.onprogress = function(event) {
            if (event.lengthComputable) {
                var percent = Math.round(event.loaded / event.total * 100);
                document.getElementById("progress").innerHTML = "上传进度：" + percent + "%";
            }
        };
        
        xhr.onreadystatechange = function() {
            if (xhr.readyState == 4 && xhr.status == 200) {
                document.getElementById("progress").innerHTML = "上传完成！";
            }
        };
        
        xhr.open("POST", "/oa/upload", true);
        // 注意：不要手动设置 Content-Type！浏览器会自动设置带 boundary 的 multipart/form-data
        xhr.send(formData);
    }
</script>
```

# 9.总结：文件上传下载关键点

```
上传关键点：
  1. form 的 method="post" + enctype="multipart/form-data"
  2. Servlet 使用 @MultipartConfig 注解
  3. 使用 request.getPart("表单name") 获取文件
  4. 调用 part.write(路径) 保存文件
  5. 校验文件类型和大小
  6. 用 UUID 防止文件名冲突

下载关键点：
  1. response.setContentType() 设置正确的 MIME 类型
  2. response.setHeader("Content-Disposition", "attachment;filename=xxx") 触发下载
  3. 使用 FileInputStream + OutputStream 读取并输出文件
  4. 中文文件名需要 URLEncoder.encode() 编码
```
