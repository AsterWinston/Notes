# 1. 文件上传
```
使用SpringMVC6版本，不需要添加以下依赖：
<dependency>
    <groupId>commons-fileupload</groupId>
    <artifactId>commons-fileupload</artifactId>
    <version>1.5</version>
</dependency>
前端页面：
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>文件上传</title>
</head>
<body>

<!--文件上传表单-->
<form th:action="@{/file/up}" method="post" enctype="multipart/form-data">
    文件：<input type="file" name="fileName"/><br>
    <input type="submit" value="上传">
</form>

</body>
</html>

重点是：form表单采用post请求，enctype是multipart/form-data，并且上传组件是：type="file"
web.xml文件：
<!--前端控制器-->
<servlet>
    <servlet-name>dispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:springmvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
    <multipart-config>
        <!--设置单个支持最大文件的大小-->
        <max-file-size>102400</max-file-size>
        <!--设置整个表单所有文件上传的最大值-->
        <max-request-size>102400</max-request-size>
        <!--设置最小上传文件大小-->
        <file-size-threshold>0</file-size-threshold>
    </multipart-config>
</servlet>
<servlet-mapping>
    <servlet-name>dispatcherServlet</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>

重点：在DispatcherServlet配置时，添加 multipart-config 配置信息。（这是Spring6，如果是Spring5，则不是这样配置，而是在springmvc.xml文件中配置：CommonsMultipartResolver，配置方法略）
SpringMVC6中把这个类已经删除了。废弃了。
Controller中的代码：
package com.powernode.springmvc.controller;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.multipart.MultipartFile;

import java.io.*;
import java.util.UUID;

@Controller
public class FileController {

    @RequestMapping(value = "/file/up", method = RequestMethod.POST)
    public String fileUp(@RequestParam("fileName") MultipartFile multipartFile, HttpServletRequest request) 
                                                                                    throws IOException {
        String name = multipartFile.getName();
        System.out.println(name);
        // 获取文件名
        String originalFilename = multipartFile.getOriginalFilename();
        System.out.println(originalFilename);
        // 将文件存储到服务器中
        // 获取输入流
        InputStream in = multipartFile.getInputStream();
        // 获取上传之后的存放目录
        File file = new File(request.getServletContext().getRealPath("/upload"));
        // 如果服务器目录不存在则新建
        if(!file.exists()){
            file.mkdirs();
        }
        // 开始写
        //BufferedOutputStream out = 
                        // new BufferedOutputStream(new FileOutputStream(file.getAbsolutePath() + "/" + originalFilename));
        // 可以采用UUID来生成文件名，防止服务器上传文件时产生覆盖
        BufferedOutputStream out = new BufferedOutputStream(new FileOutputStream(file.getAbsolutePath() + "/" +
                UUID.randomUUID().toString() + originalFilename.substring(originalFilename.lastIndexOf("."))));
        byte[] bytes = new byte[1024 * 100];
        int readCount = 0;
        while((readCount = in.read(bytes)) != -1){
            out.write(bytes,0,readCount);
        }
        // 刷新缓冲流
        out.flush();
        // 关闭流
        in.close();
        out.close();

        return "ok";
    }

}

```
# 2. 文件下载
```
<!--文件下载-->
<a th:href="@{/download}">文件下载</a>
文件下载核心程序，使用ResponseEntity：
@GetMapping("/download")
public ResponseEntity<byte[]> downloadFile(HttpServletResponse response, HttpServletRequest request) throws IOException {
    File file = new File(request.getServletContext().getRealPath("/upload") + "/1.jpeg");
    // 创建响应头对象
    HttpHeaders headers = new HttpHeaders();
    // 设置响应内容类型
    headers.setContentType(MediaType.APPLICATION_OCTET_STREAM);
    // 设置下载文件的名称
    headers.setContentDispositionFormData("attachment", file.getName());

    // 下载文件
    ResponseEntity<byte[]> entity = new ResponseEntity<byte[]>(Files.readAllBytes(file.toPath()), headers, HttpStatus.OK);
    return entity;
}

# 3.多文件上传
```
前端页面修改（支持多文件选择）：
<!--多文件上传表单-->
<form th:action="@{/file/multiUp}" method="post" enctype="multipart/form-data">
    文件1：<input type="file" name="fileNames"/><br>
    文件2：<input type="file" name="fileNames"/><br>
    文件3：<input type="file" name="fileNames"/><br>
    <input type="submit" value="上传多个文件">
</form>

<!--或者使用 multiple 属性一次选择多个文件 -->
<form th:action="@{/file/multiUp2}" method="post" enctype="multipart/form-data">
    选择多个文件：<input type="file" name="files" multiple/><br>
    <input type="submit" value="批量上传">
</form>

Controller接收方式一：使用数组接收
@RequestMapping(value = "/file/multiUp", method = RequestMethod.POST)
public String multiFileUp(@RequestParam("fileNames") MultipartFile[] fileNames, HttpServletRequest request) throws IOException {
    for (MultipartFile file : fileNames) {
        if (!file.isEmpty()) {
            String originalFilename = file.getOriginalFilename();
            String uuidFilename = UUID.randomUUID().toString() + 
                originalFilename.substring(originalFilename.lastIndexOf("."));
            File dir = new File(request.getServletContext().getRealPath("/upload"));
            if (!dir.exists()) dir.mkdirs();
            file.transferTo(new File(dir, uuidFilename));  // transferTo方法更简洁
            System.out.println("上传成功：" + uuidFilename);
        }
    }
    return "ok";
}

Controller接收方式二：使用List集合接收
@RequestMapping(value = "/file/multiUp2", method = RequestMethod.POST)
public String batchFileUp(@RequestParam("files") List<MultipartFile> files, HttpServletRequest request) throws IOException {
    for (MultipartFile file : files) {
        if (!file.isEmpty()) {
            String originalFilename = file.getOriginalFilename();
            String uuidFilename = UUID.randomUUID().toString() + 
                originalFilename.substring(originalFilename.lastIndexOf("."));
            File dir = new File(request.getServletContext().getRealPath("/upload"));
            if (!dir.exists()) dir.mkdirs();
            file.transferTo(new File(dir, uuidFilename));
        }
    }
    return "ok";
}

注意：
● transferTo()方法是MultipartFile提供的便捷方法，底层也是通过IO流实现的
● 使用file.isEmpty()判断文件是否为空（用户可能没有选择文件）
```

# 4.文件上传时的类型和大小校验
```
在实际开发中，上传文件时必须进行安全校验，防止用户上传恶意文件。

4.1 文件类型校验（通过文件扩展名）：
@RequestMapping(value = "/file/up", method = RequestMethod.POST)
public String fileUpWithCheck(@RequestParam("fileName") MultipartFile file, HttpServletRequest request) throws IOException {
    // 获取原始文件名
    String originalFilename = file.getOriginalFilename();
    
    // 获取文件扩展名
    String extension = originalFilename.substring(originalFilename.lastIndexOf(".")).toLowerCase();
    
    // 允许的文件类型白名单
    Set<String> allowedExtensions = new HashSet<>(Arrays.asList(".jpg", ".jpeg", ".png", ".gif", ".pdf"));
    if (!allowedExtensions.contains(extension)) {
        throw new RuntimeException("不支持的文件类型：" + extension + "，仅支持jpg/png/gif/pdf");
    }
    
    // ... 继续上传逻辑
}

4.2 文件类型校验（通过MIME类型）：
@RequestMapping(value = "/file/up", method = RequestMethod.POST)
public String fileUpWithMimeCheck(@RequestParam("fileName") MultipartFile file) {
    // 获取文件的MIME类型（Content-Type）
    String contentType = file.getContentType();
    
    // 允许的MIME类型
    Set<String> allowedContentTypes = new HashSet<>(Arrays.asList(
        "image/jpeg", "image/png", "image/gif", "application/pdf"
    ));
    if (!allowedContentTypes.contains(contentType)) {
        throw new RuntimeException("不支持的文件类型：" + contentType);
    }
    
    // 注：MIME类型由浏览器在上传时设置，可能会被伪造
    // 文件扩展名也可以被伪造
    // 最安全的方式是结合扩展名校验+文件魔数（magic number）校验
    // ... 继续上传逻辑
}

4.3 文件大小校验（Controller层面）：
@RequestMapping(value = "/file/up", method = RequestMethod.POST)
public String fileUpWithSizeCheck(@RequestParam("fileName") MultipartFile file) {
    // 限制单个文件不超过10MB
    long maxSize = 10 * 1024 * 1024; // 10MB
    if (file.getSize() > maxSize) {
        throw new RuntimeException("文件过大，不能超过10MB，当前文件大小：" + file.getSize() / 1024 / 1024 + "MB");
    }
    // ... 继续上传逻辑
}

注意：
● web.xml中的<max-file-size>是Servlet容器级别的限制，超出后直接抛出异常
● Controller中的文件大小校验是应用级别的，可以在抛出异常前给用户友好提示
● 文件类型校验仅靠扩展名和MIME类型不够安全，建议结合文件头（magic number）校验
```

# 5.文件下载增强
```
5.1 下载中文文件名处理：
当下载的文件名包含中文时，不同的浏览器对文件名编码的处理方式不同，需要进行兼容处理。

@GetMapping("/download/{filename}")
public ResponseEntity<byte[]> downloadFileWithChineseName(
        @PathVariable("filename") String filename,
        HttpServletRequest request) throws IOException {
    
    File file = new File(request.getServletContext().getRealPath("/upload") + "/" + filename);
    
    // 获取浏览器的User-Agent，判断浏览器类型
    String userAgent = request.getHeader("User-Agent");
    
    // 处理中文文件名
    String encodedFileName;
    if (userAgent != null && userAgent.contains("MSIE")) {
        // IE浏览器
        encodedFileName = URLEncoder.encode(filename, "UTF-8").replaceAll("\\+", "%20");
    } else if (userAgent != null && userAgent.contains("Firefox")) {
        // Firefox浏览器：使用Base64编码
        encodedFileName = "=?UTF-8?B?" + 
            Base64.getEncoder().encodeToString(filename.getBytes(StandardCharsets.UTF_8)) + "?=";
    } else {
        // Chrome/Safari等现代浏览器
        encodedFileName = URLEncoder.encode(filename, "UTF-8").replaceAll("\\+", "%20");
    }
    
    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.APPLICATION_OCTET_STREAM);
    // 使用 encodedFileName 作为下载时的文件名
    headers.setContentDispositionFormData("attachment", encodedFileName);
    
    return new ResponseEntity<>(Files.readAllBytes(file.toPath()), headers, HttpStatus.OK);
}

5.2 大文件下载（使用InputStreamResource，避免内存溢出）：
上面的方式使用Files.readAllBytes会将整个文件读入内存，对于大文件下载容易导致OOM。
推荐使用InputStreamResource进行流式下载：

@GetMapping("/download/large")
public ResponseEntity<InputStreamResource> downloadLargeFile(HttpServletRequest request) throws IOException {
    File file = new File(request.getServletContext().getRealPath("/upload") + "/bigfile.zip");
    
    if (!file.exists()) {
        return ResponseEntity.notFound().build();
    }
    
    // 使用InputStreamResource，不会将整个文件加载到内存
    InputStreamResource resource = new InputStreamResource(new FileInputStream(file));
    
    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.APPLICATION_OCTET_STREAM);
    headers.setContentDispositionFormData("attachment", file.getName());
    headers.setContentLength(file.length());  // 设置文件大小，浏览器可以显示下载进度
    
    return ResponseEntity.ok()
            .headers(headers)
            .body(resource);
}

注意：
● InputStreamResource需要导入 org.springframework.core.io.InputStreamResource
● 设置Content-Length可以让浏览器显示下载进度
● 流式下载适用于大文件（如视频、压缩包等），避免服务器内存压力
```