[英文](./README.md)

# 适用于Power BI报表服务器和SQL Reporting Services 2017的Reporting Services自定义安全示例

此项目包含一个示例以及允许您将自定义安全扩展部署到SQL Reporting Services 2017或Power BI Report Server的步骤。

# 概述
# SSRS和Power BI报表服务器中的自定义身份验证

SSRS 2016引入了一个新门户，用于托管新的OData API并托管新的报告工作负载，如移动报告和KPIS。 这个新门户使用更新的技术，并通过在单独的进程中运行而与熟悉的ReportingServicesService隔离。 此程序不是ASP.NET托管应用，因此会破坏现有自定义安全扩展的假设。 此外，自定义安全扩展的当前接口不允许传入任何外部上下文，使实现者只能选择检查众所周知的全局ASP.NET对象，这需要对接口进行一些更改。

## 改变了什么？

引入了一个可实现的新接口，该接口提供IRSRequestContext，提供扩展使用的更常见属性，以做出与身份验证相关的决策。 在以前的版本中，ReportManager是前端，可以使用自己的自定义登录页面进行配置，在SSRS2016中，只支持由reportserver托管的一个页面，并且应该对两个应用程序进行身份验证。

在以前的版本扩展中，可以依赖于ASP.NET对象随时可用的常见假设，因为新门户不在asp.net中运行，扩展可能会遇到对象为NULL的问题。
最通用的示例是访问HttpContext.Current以读取标头和cookie等请求信息。 为了允许扩展做出相同的决定，我们在扩展中引入了一个提供请求信息的新方法，并在从门户进行身份验证时调用。

扩展应该实现IAuthenticationExtension2接口来利用它。 扩展将需要实现两个版本的GetUserInfo方法，其中一个在reportserver上下文中被调用，另一个用于webhost进程中。 下面的示例显示了由ReportServer标识的门户网站的一个简单实现。

```csharp
    public void GetUserInfo(IRSRequestContext requestContext, out IIdentity userIdentity, out IntPtr userId)
    {
        userIdentity = null;
        if (requestContext.User != null)
        {
            userIdentity = requestContext.User;
        }
        
        // initialize a pointer to the current user id to zero
        userId = IntPtr.Zero;
   }
```

# 实现

## 第1步：创建UserAccounts数据库

该示例包含一个数据库脚本Createuserstore.sql，使您可以在SQL Server数据库中为Forms示例设置用户存储。
脚本位于CustomSecuritySample\Setup文件夹中。

创建UserAccounts数据库

 - 打开SQL Server Management Studio，然后连接到本地SQL Server实例。
 - 找到Createuserstore.sql SQL脚本文件。 脚本文件包含在示例项目文件中。
 - 运行查询以创建UserAccounts数据库。
 - 退出SQL Server Management Studio。


## 第2步：构建示例

您必须首先编译并安装扩展。 该过程假定您已将Reporting Services安装到默认位置：`C：\Program Files\Microsoft Power BI Report Server\PBIRS\ReportServer\`或`C：\ Program Files\Microsoft SQL Server Reporting Services\SSRS\ReportServer\`。 在本主题的其余部分中，此位置将被称为“`<install>```。

如果尚未创建强名称密钥文件，请使用以下说明生成密钥文件。

生成强名称密钥文件

- 打开Microsoft Visual Studio提示符并指向.Net Framework 4.0。

- 使用更改目录命令（CD）将命令提示符窗口的当前目录更改为保存项目的文件夹。

- 在命令提示符处，运行以下命令以生成密钥文件：sn -k SampleKey.snk。

使用Visual Studio编译示例
-	在Microsoft Visual Studio中打开CustomSecuritySample.sln。
 - 在Solution Explorer中，选择CustomSecuritySample项目。
 - 查看CustomSecuritySample项目的引用。 如果您没有看到Microsoft.ReportingServices.Interfaces.dll，请完成以下步骤：
 - 在“项目”菜单上，单击“添加引用”。 将打开“添加引用”对话框。
 - 单击.NET选项卡。
 - 单击“浏览”，在本地驱动器上找到Microsoft.ReportingServices.Interfaces。 默认情况下，程序集位于`<install>\ReportServer\bin`目录中。 单击确定。 选定的参考将添加到您的项目中。
 - 在“build”菜单上，单击“build solutions”

调试

要调试扩展，您可能希望将调试器附加到ReportingServicesService.exe和Microsoft.ReportingServices.Portal.Webhost.exe。 并为实现接口IAuthenticationExtension2的方法添加断点。


## 第3步：部署和配置

自定义安全扩展所需的基本配置与以前的版本相同。 对于ReportServer文件夹中存在的web.config和rsreportserver.config，需要进行以下更改。 报表管理器不再有单独的web.config，门户网站将继承与reportserver端点相同的设置。

部署示例
-	将Logon.aspx页面复制到```<install>\ReportServer目录```。
 - 将Microsoft.Samples.ReportingServices.CustomSecurity.dll和Microsoft.Samples.ReportingServices.CustomSecurity.pdb复制到```<install>\ReportServer\bin```目录。
 - 将Microsoft.Samples.ReportingServices.CustomSecurity.dll和Microsoft.Samples.ReportingServices.CustomSecurity.pdb复制到```<install>\Portal```目录。
 - 将Microsoft.Samples.ReportingServices.CustomSecurity.dll和Microsoft.Samples.ReportingServices.CustomSecurity.pdb复制到```<install>\PowerBI```目录。 （仅Power BI Report Server需要完成。）

如果PDB文件不存在，则不是由上面提供的Build步骤创建的。 确保将“调试/构建的项目属性”设置为生成PDB文件。

修改ReportServer文件夹中的文件

- 修改RSReportServer.config文件。

 - 使用Visual Studio或简单的文本编辑器（如记事本）打开RSReportServer.config文件。 RSReportServer.config位于```<install>\ReportServer```目录中。
 - 找到```<AuthenticationTypes>``元素并修改设置如下：

   ```xml
   <Authentication>
   	<AuthenticationTypes> 
   		<Custom/>
   	</AuthenticationTypes>
   	<RSWindowsExtendedProtectionLevel>Off</RSWindowsExtendedProtectionLevel>
   	<RSWindowsExtendedProtectionScenario>Proxy</RSWindowsExtendedProtectionScenario>
   	<EnableAuthPersistence>true</EnableAuthPersistence>
   </Authentication>
   ```

-	在`<Extensions>`元素中找到```<Security>```和```<Authentication>```元素，并按如下方式修改设置：

	```xml
	<Security>
		<Extension Name="Forms" Type="Microsoft.Samples.ReportingServices.CustomSecurity.Authorization, Microsoft.Samples.ReportingServices.CustomSecurity" >
		<Configuration>
			<AdminConfiguration>
				<UserName>username</UserName>
			</AdminConfiguration>
		</Configuration>
		</Extension>
	</Security>
	```
	```xml
	<Authentication>
		<Extension Name="Forms" Type="Microsoft.Samples.ReportingServices.CustomSecurity.AuthenticationExtension,Microsoft.Samples.ReportingServices.CustomSecurity" />
	</Authentication> 
	```
	

**注意**：
如果在未安装安全套接字层（SSL）证书的开发环境中运行示例安全扩展，则必须在先前的配置条目中将`<UseSSL>`元素的值更改为False。 我们建议您在将Reporting Services与表单身份验证结合使用时始终使用SSL。

修改RSSrvPolicy.config文件
- 您需要为自定义安全扩展添加一个代码组，以便为您的扩展授予FullTrust权限。 您可以通过将代码组添加到RSSrvPolicy.config文件来完成此操作。

 - 打开位于```<install> \ ReportServer```目录中的RSSrvPolicy.config文件。

 - 在安全策略文件中具有URL成员资格为$ CodeGen的现有代码组之后添加以下```<CodeGroup>``元素，如下所示，然后在RSSrvPolicy.config中添加如下条目。 确保根据ReportServer安装目录更改以下路径：

   ```xml
   <CodeGroup
   	class="UnionCodeGroup"
   	version="1"
   	Name="SecurityExtensionCodeGroup" 
   	Description="Code group for the sample security extension"
   	PermissionSetName="FullTrust">
   <IMembershipCondition 
   	class="UrlMembershipCondition"
   	version="1"
   	Url="C:\Program Files\Microsoft Power BI Report Server\PBIRS\ReportServer\bin\Microsoft.Samples.ReportingServices.CustomSecurity.dll"/>
   </CodeGroup>
   ```
   注意：
   为简单起见，表单身份验证示例名称不明确，需要在安全策略文件中使用简单的URL成员资格条目。 在生产安全性扩展实现中，您应该在为程序集添加安全策略时创建强名称程序集并使用强名称成员资格条件。 有关强名称程序集的详细信息，请参阅MSDN上的“创建和使用强名称程序集”主题。

修改Report Server的Web.config文件
- 在文本编辑器中打开Web.config文件。 默认情况下，该文件位于`<install>\ReportServer`目录中。

 - 找到`<identity>`元素并将Impersonate属性设置为false。

    ```xml
    <identity impersonate="false" />
    ```

-	找到`<authentication>`元素并将Mode属性更改为Forms。 另外，添加以下`<forms>`元素作为`<authentication>`元素的子元素，并设置loginUrl，name，timeout和path属性，如下所示：

	```xml
	<authentication mode="Forms">
		<forms loginUrl="logon.aspx" name="sqlAuthCookie" timeout="60" path="/"></forms>
	</authentication> 
	```
	
-   在`<authentication>`元素之后直接添加以下`<authorization>`元素。

	```xml
	<authorization> 
	<deny users="?" />
	</authorization> 
	```

这将拒绝未经身份验证的用户访问报表服务器的权限。 先前建立的`<authentication>`元素的loginUrl属性会将未经身份验证的请求重定向到Logon.aspx页面。


## 第4步：生成机器密钥

使用Forms身份验证要求所有报表服务器进程都可以访问身份验证cookie。 这涉及配置机器密钥和解密算法 - 对于之前已设置SSRS以在横向扩展环境中工作的人来说，这是一个熟悉的步骤。

在RSReportServer.config文件中的`<Configuration>`下生成并添加`<MachineKey>`。

```xml
<MachineKey ValidationKey="[YOUR KEY]" DecryptionKey="[YOUR KEY]" Validation="AES" Decryption="AES" />
```

**检查外标签的属性，它应该是Pascal Casing，如上例**

**不需要```<system.web>```条目 **

您应该使用特定于您部署的验证密钥，有几个工具可以生成密钥，例如Internet Information Services Manager（IIS）

## 第5步：配置PassThroughCookies

新门户网站和报告服务器使用内部soap API进行通信以进行某些操作。 当需要将其他cookie从门户传递到服务器时，PassThroughCookies属性仍然可用。 更多详情：https：//msdn.microsoft.com/en-us/library/ms345241.aspx
在rsreportserver.config文件中添加以下```<UI>```

```xml
<UI>
   <CustomAuthenticationUI>
      <PassThroughCookies>
         <PassThroughCookie>sqlAuthCookie</PassThroughCookie>
      </PassThroughCookies>
   </CustomAuthenticationUI>
</UI>
```

# 自动配置示例

如果您是默认安装的Power BI Report Server（该脚本仅对Power BI Report Server有效，对于SSRS，您需要遵循手动步骤），所有步骤都在PowerShell脚本中自动执行
```
.\Configure.ps1
```
*此配置不适用于生产，您应该生成自己的不同于示例的强名称密钥和自己的身份验证密钥*

# 行为守则
该项目采用了[Microsoft开源代码行为准则](https://opensource.microsoft.com/codeofconduct/).
有关更多信息，请参阅[行为准则常见问题](https://opensource.microsoft.com/codeofconduct/faq/)或有任何其他问题或意见请联系[opencode@microsoft.com](mailto：opencode@microsoft.com)。