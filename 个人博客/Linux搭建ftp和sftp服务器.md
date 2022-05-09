# 前言

## FTP

FTP（File Transfer Protocol，文件传输协议）是 TCP/IP 协议组中的协议之一，一般是为了方便数据共享的。FTP 包括一个 FTP 服务器和多个 FTP 客户端，其中 FTP 服务器上用来存储文件，用户可以使用 FTP 客户端通过 FTP 协议访问位于 FTP 服务器上的资源。

在开发网站的时候，通常利用 FTP 协议把网页或程序传到 Web 服务器上。此外，由于 FTP 传输效率非常高，在网络上传输大的文件时，一般也会采用该协议。

## SFTP

SFTP 是一种安全的文件传输协议，可以为传输文件提供一种安全的加密方法，有着与 FTP 几乎一样的语法和功能。

SFTP 要求客户端用户必须由服务器进行身份验证，并且数据传输必须通过安全通道（SSH）进行，即不传输明文密码或文件数据，它允许对远程文件执行各种操作。

> SFTP 和 FTP 的区别？
>
> 1. **安全通道**：FTP 不提供任何安全通道来在主机之间传输文件；而 SFTP 协议提供了一个安全通道，用于在网络上的主机之间传输文件。
> 2. **使用的协议**：FTP 使用 TCP/IP 协议，而 SFTP 是 SSH 协议的一部分，它是一种远程登录信息。
> 3. **链接方式**：FTP 使用 TCP 端口 21 上的控制连接建立连接；而 SFTP 是在客户端和服务器之间通过 SSH 协议（TCP 端口 22）建立的安全连接传输文件。
> 4. **安全性**：FTP 密码和数据以纯文本格式发送，大多数情况下不加密，安全性不高；而 SFTP 会在发送之前加密数据，二进制的形式传递，安全性较高。简单来说，FTP 基于 TCP 来传输文件，明文传输用户信息和数据，而 SFTP 基于 SSH 来加密传输文件，可靠性搞，可断点续传。

## 参考链接

> 声明：所有操作均在 Centos7 环境下进行。

- https://help.aliyun.com/document_detail/92048.html#section-821-887-8np

# 搭建FTP服务器

## 配置FTP服务器

1、安装 vsftpd：`yum install -y vsftpd`。

2、设置 ftp 服务开机启动：`systemctl enable vsftpd.service`。

3、启动 ftp 服务：`systemctl start vsftpd.service`。

4、查看 ftp 服务监听端口，默认为 21：`netstat -antup | grep ftp`。

5、为 ftp 服务创建一个用户，并指定供 ftp 服务使用的文件目录。

```sh
# 添加用户并指定密码
adduser ftptest
passwd ftptest

# 创建一个供FTP服务使用的文件目录
mkdir /home/ftptest/test

# 创建测试文件
touch /home/ftptest/test/testfile.txt

# 运行以下命令更改/var/ftp/test目录的拥有者为ftptest
chown -R ftptest:ftptest /home/ftptest/test
```

6、修改 vsftpd.conf 配置文件，`vim /etc/vsftpd/vsftpd.conf`。

```conf
#除下面提及的参数，其他参数保持默认值即可。

#修改下列参数的值：
#禁止匿名登录FTP服务器。
anonymous_enable=NO
#允许本地用户登录FTP服务器。
local_enable=YES
#监听IPv4 sockets。
listen=YES

#在行首添加#注释掉以下参数：
#关闭监听IPv6 sockets。
#listen_ipv6=YES

#在配置文件的末尾添加下列参数：
##设置本地用户登录后所在目录。
local_root=/home/ftptest/test
##全部用户被限制在主目录。
chroot_local_user=YES
##启用例外用户名单。
chroot_list_enable=YES
##指定例外用户列表文件，列表中用户不被锁定在主目录。
chroot_list_file=/etc/vsftpd/chroot_list
##开启被动模式。
pasv_enable=YES
allow_writeable_chroot=YES
##本教程中为Linux实例的公网IP。
pasv_address=192.168.58.100
##设置被动模式下，建立数据传输可使用的端口范围的最小值。
##建议您把端口范围设置在一段比较高的范围内，例如50000~50010，有助于提高访问FTP服务器的安全性。
pasv_min_port=50000
##设置被动模式下，建立数据传输可使用的端口范围的最大值。
pasv_max_port=50010
```

7、创建 chroot_list 文件，并在文件中写入例外用户名单。

```sh
# vim /etc/vsftpd/chroot_list
ftptest
```

8、重启 vsftpd 服务，并开放 21 端口。

```sh
# 重启vsftpd服务
systemctl restart vsftpd.service

# 我这里直接关闭防火墙
sudo systemctl stop firewalld.service
sudo systemctl disable firewalld.service
```

## 客户端连接

打开本地文件管理器，在地址栏中输入 `ftp://<FTP服务器公网IP地址>:FTP端口`。

![img](https://gitee.com/yanxuwei666/blogimage/raw/master/img/202205092116131.png)

## Java连接FTP服务器

1、引入 maven 依赖。

```xml
<dependencies>
    <dependency>
        <groupId>commons-net</groupId>
        <artifactId>commons-net</artifactId>
        <version>3.7.2</version>
    </dependency>

    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.13.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

2、编写 FTP 文件上传与下载工具类。

```java
public class FtpUtil {

    /**
     * Description: 向FTP服务器上传文件
     *
     * @param host     FTP服务器hostname
     * @param port     FTP服务器端口
     * @param username FTP登录账号
     * @param password FTP登录密码
     * @param basePath FTP服务器基础目录
     * @param filePath FTP服务器文件存放路径。文件的路径为basePath+filePath
     * @param filename 上传到FTP服务器上的文件名
     * @param input    输入流
     * @return 成功返回true，否则返回false
     */
    public static boolean uploadFile(String host, int port, String username, String password, String basePath,
                                     String filePath, String filename, InputStream input) {
        boolean result = false;
        FTPClient ftp = new FTPClient();
        try {
            int reply;
            ftp.connect(host, port);// 连接FTP服务器
            // 如果采用默认端口，可以使用ftp.connect(host)的方式直接连接FTP服务器
            ftp.login(username, password);// 登录
            reply = ftp.getReplyCode();
            if (!FTPReply.isPositiveCompletion(reply)) {
                ftp.disconnect();
                return result;
            }
            //切换到上传目录
            if (!ftp.changeWorkingDirectory(basePath + filePath)) {
                //如果目录不存在创建目录
                String[] dirs = filePath.split("/");
                String tempPath = basePath;
                for (String dir : dirs) {
                    if (null == dir || "".equals(dir)) continue;
                    tempPath += "/" + dir;
                    if (!ftp.changeWorkingDirectory(tempPath)) {  //进不去目录，说明该目录不存在
                        if (!ftp.makeDirectory(tempPath)) { //创建目录
                            //如果创建文件目录失败，则返回
                            System.out.println("创建文件目录" + tempPath + "失败");
                            return result;
                        } else {
                            //目录存在，则直接进入该目录
                            ftp.changeWorkingDirectory(tempPath);
                        }
                    }
                }
            }
            //设置上传文件的类型为二进制类型
            ftp.setFileType(FTP.BINARY_FILE_TYPE);
            //上传文件
            if (!ftp.storeFile(filename, input)) {
                return result;
            }
            input.close();
            ftp.logout();
            result = true;
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (ftp.isConnected()) {
                try {
                    ftp.disconnect();
                } catch (IOException ioe) {
                }
            }
        }
        return result;
    }

    /**
     * Description: 从FTP服务器下载文件
     *
     * @param host       FTP服务器hostname
     * @param port       FTP服务器端口
     * @param username   FTP登录账号
     * @param password   FTP登录密码
     * @param remotePath FTP服务器上的相对路径
     * @param fileName   要下载的文件名
     * @param localPath  下载后保存到本地的路径
     * @return
     */
    public static boolean downloadFile(String host, int port, String username, String password, String remotePath,
                                       String fileName, String localPath) {
        boolean result = false;
        FTPClient ftp = new FTPClient();
        try {
            int reply;
            ftp.connect(host, port);
            // 如果采用默认端口，可以使用ftp.connect(host)的方式直接连接FTP服务器
            ftp.login(username, password);// 登录
            reply = ftp.getReplyCode();
            if (!FTPReply.isPositiveCompletion(reply)) {
                ftp.disconnect();
                return result;
            }
            ftp.changeWorkingDirectory(remotePath);// 转移到FTP服务器目录
            FTPFile[] fs = ftp.listFiles();
            for (FTPFile ff : fs) {
                if (ff.getName().equals(fileName)) {
                    File localFile = new File(localPath + "/" + ff.getName());

                    OutputStream is = new FileOutputStream(localFile);
                    ftp.retrieveFile(ff.getName(), is);
                    is.close();
                }
            }

            ftp.logout();
            result = true;
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (ftp.isConnected()) {
                try {
                    ftp.disconnect();
                } catch (IOException ioe) {
                }
            }
        }
        return result;
    }
}
```

3、测试。

```java
public class FtpUtilTest {

    @Test
    public void uploadFile() {
        try {
            FileInputStream in=new FileInputStream(new File("D:\\configuration\\电脑壁纸\\3C6550DBC774EABF4FF17FACFB6EA175.jpg"));
            boolean flag = FtpUtil.uploadFile(
                    "192.168.58.100",
                    21,
                    "ftptest",
                    "ftptest",
                    "./",
                    "/www/images/",
                    "background.jpg",
                    in);
            System.out.println(flag);
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }
    }

    @Test
    public void downloadFile() {
        boolean flag = FtpUtil.downloadFile(
                "192.168.58.100",
                21,
                "ftptest",
                "ftptest",
                "./www/images/",
                "background.jpg",
                "D:\\configuration"
        );
        System.out.println(flag);
    }
}
```

4、查看测试结果。

![image-20220509155318620](https://gitee.com/yanxuwei666/blogimage/raw/master/img/202205092116132.png)

![image-20220509155329888](https://gitee.com/yanxuwei666/blogimage/raw/master/img/202205092116133.png)



# 搭建SFTP服务器

## 配置SFTP服务器

1、创建 sftp 用户组，组名为 sftptests：`groupadd sftptests`。

2、创建 sftptest 用户，并设置为 sftp 组：`useradd -g sftptests -s /sbin/nologin -M sftptest`。

3、修改 sftptest 用户密码。

4、创建 sftptest 用户的根目录和属主属组，修改权限为 755。

```shell
mkdir /home/sftptest
chown -R sftptest:sftptests /home/sftptest  # 该路径所属者为sftptest
chmod 755 /home/sftptest
```

5、创建该用户所属的数据路径。

```shell
mkdir -p /home/sftptest/sftpdata
chown sftptest:sftptests /home/sftptest/sftpdata # sftp用户在该路径下有读和写的权限，可进行创建和删除目录文件等操作
chmod 755 /home/sftptest/sftpdata
```

6、修改 /etc/ssh/sshd_config 的配置文件：

```nginx
#Subsystem      sftp    /usr/libexec/openssh/sftp-server
Subsystem       sftp    internal-sftp

# 最后一行新增
Match Group sftpusers
        X11Forwarding no
        AllowTcpForwarding no
        ChrootDirectory /home/sftptest 
        ForceCommand internal-sftp
```

7、配置完成后，重启 sshd：`systemctl restart sshd`。

8、sftp 登录：`sftp sftptest@127.0.0.1`。

## 客户端连接

> 我这里通过 FileZillz 这个软件进行连接。

![image-20220509160910965](https://gitee.com/yanxuwei666/blogimage/raw/master/img/202205092116134.png)

## Java连接SFTP服务器

1、引入 maven 依赖。

```xml
<dependencies>
    <dependency>
        <groupId>com.jcraft</groupId>
        <artifactId>jsch</artifactId>
        <version>0.1.54</version>
    </dependency>

    <dependency>
        <groupId>commons-net</groupId>
        <artifactId>commons-net</artifactId>
        <version>3.7.2</version>
    </dependency>

    <dependency>
        <groupId>commons-io</groupId>
        <artifactId>commons-io</artifactId>
        <version>2.11.0</version>
    </dependency>

    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.13.2</version>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>2.6.4</version>
    </dependency>
</dependencies>
```

2、编写 FTP 文件上传与下载工具类。

```java
package com.xuwei.util;


import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.util.Properties;
import java.util.Vector;
import org.apache.commons.io.IOUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import com.jcraft.jsch.Channel;
import com.jcraft.jsch.ChannelSftp;
import com.jcraft.jsch.JSch;
import com.jcraft.jsch.JSchException;
import com.jcraft.jsch.Session;
import com.jcraft.jsch.SftpException;


/**
 * @Author yxw
 * @Date 2022/5/9 16:23
 * @Description SFTP文件上传与下载工具类
 */
public class SFTPUtil {
    private transient Logger log = LoggerFactory.getLogger(this.getClass());

    private ChannelSftp sftp;

    private Session session;
    /** SFTP 登录用户名*/
    private String username;
    /** SFTP 登录密码*/
    private String password;
    /** 私钥 */
    private String privateKey;
    /** SFTP 服务器地址IP地址*/
    private String host;
    /** SFTP 端口*/
    private int port;


    /**
     * 构造基于密码认证的sftp对象
     */
    public SFTPUtil(String username, String password, String host, int port) {
        this.username = username;
        this.password = password;
        this.host = host;
        this.port = port;
    }

    /**
     * 构造基于秘钥认证的sftp对象
     */
    public SFTPUtil(String username, String host, int port, String privateKey) {
        this.username = username;
        this.host = host;
        this.port = port;
        this.privateKey = privateKey;
    }

    public SFTPUtil(){}


    /**
     * 连接sftp服务器
     */
    public void login(){
        try {
            JSch jsch = new JSch();
            if (privateKey != null) {
                jsch.addIdentity(privateKey);// 设置私钥
            }

            session = jsch.getSession(username, host, port);

            if (password != null) {
                session.setPassword(password);
            }
            Properties config = new Properties();
            config.put("StrictHostKeyChecking", "no");

            session.setConfig(config);
            session.connect();

            Channel channel = session.openChannel("sftp");
            channel.connect();

            sftp = (ChannelSftp) channel;
        } catch (JSchException e) {
            e.printStackTrace();
        }
    }

    /**
     * 关闭连接 server
     */
    public void logout(){
        if (sftp != null) {
            if (sftp.isConnected()) {
                sftp.disconnect();
            }
        }
        if (session != null) {
            if (session.isConnected()) {
                session.disconnect();
            }
        }
    }


    /**
     * 将输入流的数据上传到sftp作为文件。文件完整路径=basePath+directory
     * @param basePath  服务器的基础路径
     * @param directory  上传到该目录
     * @param sftpFileName  sftp端文件名
     * @param input   输入流
     */
    public void upload(String basePath,String directory, String sftpFileName, InputStream input) throws SftpException{
        try {
            sftp.cd(basePath);
            sftp.cd(directory);
        } catch (SftpException e) {
            //目录不存在，则创建文件夹
            String [] dirs=directory.split("/");
            String tempPath=basePath;
            for(String dir:dirs){
                if(null== dir || "".equals(dir)) continue;
                tempPath+="/"+dir;
                try{
                    sftp.cd(tempPath);
                }catch(SftpException ex){
                    sftp.mkdir(tempPath);
                    sftp.cd(tempPath);
                }
            }
        }
        sftp.put(input, sftpFileName);  //上传文件
    }


    /**
     * 下载文件。
     * @param directory 下载目录
     * @param downloadFile 下载的文件
     * @param saveFile 存在本地的路径
     */
    public void download(String directory, String downloadFile, String saveFile) throws SftpException, FileNotFoundException{
        if (directory != null && !"".equals(directory)) {
            sftp.cd(directory);
        }
        File file = new File(saveFile);
        sftp.get(downloadFile, new FileOutputStream(file));
    }

    /**
     * 下载文件
     * @param directory 下载目录
     * @param downloadFile 下载的文件名
     * @return 字节数组
     */
    public byte[] download(String directory, String downloadFile) throws SftpException, IOException{
        if (directory != null && !"".equals(directory)) {
            sftp.cd(directory);
        }
        InputStream is = sftp.get(downloadFile);

        byte[] fileData = IOUtils.toByteArray(is);

        return fileData;
    }


    /**
     * 删除文件
     * @param directory 要删除文件所在目录
     * @param deleteFile 要删除的文件
     */
    public void delete(String directory, String deleteFile) throws SftpException{
        sftp.cd(directory);
        sftp.rm(deleteFile);
    }


    /**
     * 列出目录下的文件
     * @param directory 要列出的目录
     */
    public Vector<?> listFiles(String directory) throws SftpException {
        return sftp.ls(directory);
    }
}
```

3、测试。

```java
public class SFTPUtilTest {

    @Test
    public void upload() throws SftpException, FileNotFoundException {
        SFTPUtil sftp = new SFTPUtil(
                "sftptest",
                "sftptest",
                "192.168.58.100",
                22
        );
        sftp.login();
        File file = new File("D:\\configuration\\background.jpg");
        InputStream is = new FileInputStream(file);

        sftp.upload("./","/images/", "test_sftp.jpg", is);
        sftp.logout();
    }

    @Test
    public void download() {

    }
}
```

4、查看测试结果。

![image-20220509163432327](https://gitee.com/yanxuwei666/blogimage/raw/master/img/202205092116135.png)