---
layout: post
title:  "Azkaban中文乱码记录"
categories: Azkaban
tags: Azkaban 中文 乱码
author: Victor
---

* content
{:toc}

<p>Azkaban作为LinkedIn开源的任务流式管理工具，在工作中很大程度上被用到。但是，由于非国人开发，对中文的支持性很不好。大多数情况下，会出现几种乱码现象:
-  执行内置脚本生成log乱码
-  直接command执行中文乱码
-  中文包名乱码
</p>
<p>以实际操作过程为例，项目使用的是Azkaban-3.20.0版本，通过源码编译后打包，解压配置后运行项目。由于整个过程需要结合Datax来做，所以交给Azkaban的任务就变成了执行对应的Datax json文件。而Datax作为阿里巴巴开源的数据迁移工具，日志的生成与提示都是中文发布，这就造成了通过Azkaban的执行流中的job来看log却看不出乱码的中文是什么意思。摸索许久，决定从源码入手开始层层解惑。</p>
<!-- more -->
通过观看源码流程，大致判断可能产生乱码的出处:
1.zip包初始上传时解压乱码
2.zip包解压后解析.job格式properties文件乱码
3.logEvent事件存入Mysql时乱码
4.fetchJobLog时从BLOB类型转成String类型乱码

可疑处一：
3.20.0版本的源码中，解压zip包用的是官方的ZIP包，这样就会在解压时造成中文乱码问题，源码如下:
```
 private File unzipFile(File archiveFile) throws IOException {
    ZipFile zipfile = new ZipFile(archiveFile);
    File unzipped = Utils.createTempDir(tempDir);
    Utils.unzip(zipfile, unzipped);
    zipfile.close();

    return unzipped;
  }
  
 public static void unzip(ZipFile source, File dest) throws IOException {
    Enumeration<?> entries = source.entries();
    while (entries.hasMoreElements()) {
      ZipEntry entry = (ZipEntry) entries.nextElement();
      File newFile = new File(dest, entry.getName());
      if (entry.isDirectory()) {
        newFile.mkdirs();
      } else {
        newFile.getParentFile().mkdirs();
        InputStream src = source.getInputStream(entry);
        try {
          OutputStream output =
              new BufferedOutputStream(new FileOutputStream(newFile));
          try {
            IOUtils.copy(src, output);
          } finally {
            output.close();
          }
        } finally {
          src.close();
        }
      }
    }
  }
```
对于这一点，org.apache.ant包中提供了第三方对中文支持的库，这里我选择的是1.9.7版本的Ant包，添加依赖如下:
```
compile group: 'org.apache.ant', name: 'ant', version: '1.9.7'
```
接着对上面的源码就行更改替换，更改后的代码如下:
```
public static void unzip(ZipFile source, File dest) throws IOException {
        Enumeration<?> entries = source.getEntries();
        while (entries.hasMoreElements()) {
            ZipEntry entry = (ZipEntry) entries.nextElement();
            File newFile = new File(dest, entry.getName());
            if (entry.isDirectory()) {
                newFile.mkdirs();
            } else {
                newFile.getParentFile().mkdirs();
                InputStream in = null;
                OutputStream out = null;
                try {
                    in = source.getInputStream(entry);
                    out = new FileOutputStream(newFile);
                    byte[] buff = new byte[1024];
                    int len;
                    while ((len = in.read(buff)) > 0) {
                        out.write(buff, 0, len);
                    }
                } finally {
                    if (null != in) {
                        in.close();
                    }
                    if (null != out) {
                        out.close();
                    }
                }
            }
        }
    }
```
其中，之前的ZipEntry,ZipFile,ZipOutputStream对象的从属org.apache.tools，即导入包为:
import org.apache.tools.zip.ZipEntry;
import org.apache.tools.zip.ZipFile;
import org.apache.tools.zip.ZipOutputStream;

疑点处二:
读取properties同样也有可能是造成中文乱码的一大疑点，从源码上来看，并没有对文件进行编码处理。
```
  public Props(Props parent, File file) throws IOException {
    this(parent);
    setSource(file.getPath());

    InputStream input = new BufferedInputStream(new FileInputStream(file));
    try {
      loadFrom(input);
    } catch (IOException e) {
      throw e;
    } finally {
      input.close();
    }
  }
  
private void loadFrom(InputStream inputStream) throws IOException {
    Properties properties = new Properties();
    properties.load(inputStream);
    this.put(properties);
  }
```
如此，就需要对读入的流做编码处理，处理后的代码如下:
```
private void loadFrom(InputStream inputStream) throws IOException {
    Properties properties = new Properties();
    BufferedReader bf = new BufferedReader(new InputStreamReader(inputStream,"UTF-8"));
    properties.load(bf);
    this.put(properties);
}
```
疑点处三，四:
数据从Blob转化成String也有乱码的几率，不过在查看源码后，否定了这种猜想。
```
public LogData handle(ResultSet rs) throws SQLException {
      if (!rs.next()) {
        return null;
      }

      ByteArrayOutputStream byteStream = new ByteArrayOutputStream();

      do {
        // int execId = rs.getInt(1);
        // String name = rs.getString(2);
        @SuppressWarnings("unused")
        int attempt = rs.getInt(3);
        EncodingType encType = EncodingType.fromInteger(rs.getInt(4));
        int startByte = rs.getInt(5);
        int endByte = rs.getInt(6);

        byte[] data = rs.getBytes(7);

        int offset =
            this.startByte > startByte ? this.startByte - startByte : 0;
        int length =
            this.endByte < endByte ? this.endByte - startByte - offset
                : endByte - startByte - offset;
        try {
          byte[] buffer = data;
          if (encType == EncodingType.GZIP) {
            buffer = GZIPUtils.unGzipBytes(data);
          }

          byteStream.write(buffer, offset, length);
        } catch (IOException e) {
          throw new SQLException(e);
        }
      } while (rs.next());
```
可以看到比较标准的流式处理过程

最后，编译打包，执行./gradlew build -x test。将编译好的azkaban-common-0.1.0-SNAPSHOT.jar,azkaban-core-0.1.0-SNAPSHOT.jar,azkaban-exec-server-0.1.0-SNAPSHOT.jar放入Executor的lib目录下，将azkaban-common-0.1.0-SNAPSHOT.jar,azkaban-core-0.1.0-SNAPSHOT.jar当入WebServer的lib目录下。测试结果如下：  
![](https://V-I-C-T-O-R.github.io/pics/azkaban/zh_cn_1.png)  
![](https://V-I-C-T-O-R.github.io/pics/azkaban/zh_cn_2.png)
