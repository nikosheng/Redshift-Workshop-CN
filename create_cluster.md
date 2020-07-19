#创建集群
--
在本实验中，您将启动一个新的Redshift集群，建立连接并配置JDBC客户端工具。

##内容索引
- 开始使用
- 前提条件
- CloudFormation
- 配置客户端
- 运行例子

##开始使用
确定并捕获以下信息，然后登录到AWS控制台。如果您不熟悉AWS，则可以创建一个帐户。

- [AWS_Account_Id] 
- [AWS_User_Name]
- [AWS_密码]

确定最接近您的[AWS区域名称]和[AWS区域ID]，然后将控制台切换到该区域。

##前提条件
- 尽管Amazon Redshift确实提供了基于Web的查询编辑器来执行简单的查询，这些编辑器将在3分钟内完成，但是对于这些实验，建议您安装第三方工具。我们将使用SQL Workbench / J。
- 一旦安装了第三方工具，您将需要JDBC或ODBC驱动程序。 Amazon Redshift提供了JDBC和ODBC驱动程序供下载。请参阅Amazon Redshift文档网站上的使用SQL客户端工具连接到Amazon Redshift集群。

##CloudFormation

[创建群集](https://console.aws.amazon.com/cloudformation/home?#/stacks/new?stackName=ImmersionLab1&templateURL=https://s3-us-west-2.amazonaws.com/redshift-immersionday-labs/lab1.yaml)

##配置客户端

- 启动SQL Workbench / J并导航到[文件|管理驱动程序]。
- 选择“ Amazon Redshift”，并将驱动程序库位置设置为您下载Redshift JDBC驱动程序的位置。点击确定


- 导航到[文件| [连接窗口]创建新的连接配置文件并修改以下设置，完成后单击“测试连接”按钮。
	- 名称-“ LabConnection”
	- 驱动程序-Amazon Redshift（com.amazon.redshift.jdbc.Driver）
	- URL-通过导航到“群集列表”，选择您的群集，单击“属性”并复制位于“连接”详细信息中的端点来找到此URL。
	- Username - [Master user name]
	- Password - [Master user password]
	- Autocommit - Enabled

	
##运行例子
运行以下查询以列出redshift集群中的用户

```
select * from pg_user
```