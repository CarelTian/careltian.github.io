# RF学习总结


# Robot  Framework简介

RPA框架是机器人过程自动化(RPA)的开源库和工具的集合，它被设计为与Robot Framework和Python一起使用。目标是为软件机器人开发人员提供良好的文档和积极维护的核心库。相比较于纯python实现，它的使用逻辑清晰，使用方法简单，可扩展性高。但是在某些特殊场景，Robot Framework具有局限性，比如并发执行，直接接管浏览器而不需要重新打开。

以下Robot  Framework简称RF。RF的应用分为两种，一个是为公司解放劳动力，自动化处理简单但又繁琐的操作。二是为个人使用，现电商抢购活动还有秒杀抢单我怀疑完全可以用RF实现的，只要网速够快，并且成功破解了验证码是没有人能抢过它的。

# RF准备工作

## 环境

| 项目           | 版本   |
| -------------- | ------ |
| python         | 3.7.9  |
| rpaframework   | 22.2.3 |
| robotframework | 5.0.1  |
| selenium       | 4.5.0  |

## 开发工具

RF实体是一个.robot文件，pycharm的插件有点问题，而且不能识别里面的Keyword。因此使用Vscode进行开发。

需要准备的插件如下所示

![图一](../images/RF_1.png)

# RF结构

RF分为四部分**Settings,  Variables,  Test Cases,  Keywords**

## \*\*\* Setting \*\*\*

- Documentation	用于机器人的描述和介绍，对执行并没有什么作用
- Library    用于导入各种库，每个库会有上百个Keyword

Library的官方文档（[https://rpaframework.org/index.html](https://rpaframework.org/index.html)）

## \*\*\* Variable \*\*\*

用于程序初始化的变量

- @ 用作创建列表
- &amp; 用作创建字典
- $ 用作创建字符串或数字

## \*\*\* Test  Cases \*\*\*

与Task作用一样，用于写总流程

## \*\*\*Keywords \*\*\*

由多个自定义的Keyword组成，表示执行的步骤。

定义的Keyword下行加[Arguments]    ${} 可作为函数的参数。

```
循环表示方法
FOR		${}		IN		@{}
		do  something	${}
END
判断表示方法
IF	condititon
	do()
ELSE IF	 condition
	do()
ELSE
	do()
END
异常处理
TRY
	Some Keyword
EXCEPT    ValueError: *    type=GLOB    AS   ${error}
	Error Handler 1    ${error}
EXCEPT    [Ee]rror \\d&#43;    (Invalid|Bad) usage    type=REGEXP    AS    ${error}
	Error Handler 2    ${error}
EXCEPT    AS    ${error}
	Error Handler 3    ${error}
END

```



# Python扩展

导入库时使用的测试库的名称与实现它的模块或类的名称相同。

## Library

**from robot.api.deco import library**

RF导入python的库就像直接实例化一个类，运行构造参数，如下图所示。

![图二](../images/RF_2.png)

@library装饰器
配置实现为类的库的一种简单方法是使用robot.api.deco.library类装饰器。它允许配置库的作用域、版本、自定义参数转换器、文档格式和监听器，可选参数scope、version、converter、doc_format和监听器。当使用这些参数时，它们会自动设置匹配的ROBOT_LIBRARY_SCOPE、ROBOT_LIBRARY_VERSION、ROBOT_LIBRARY_CONVERTERS、ROBOT_LIBRARY_DOC_FORMAT和ROBOT_LIBRARY_LISTENER属性

## Keyword

**from robot.api.deco import keyword**

默认情况下，一个python类或模块下的所有函数被认为是Keyword。如果在设置中使用下图设置，默认不配置为keyword。函数的前缀可以使用@keyword开启。

![图三](../images/RF_3.png)

或者直接使用@not_keyword禁用RF。

@keyword(name=None,tag=(),type=())-&gt;Any

可以修改把参数放在name里面

# 常用的库

## RPA.Browswer.Selenium			

 auto_close=${*FALSE*}    //执行完不自动关闭		 

- 打开网站		&lt;u&gt;Open Available Browser&lt;/u&gt;			url
- 输入内容         &lt;u&gt;Input Text&lt;/u&gt;          locator       text       clear=True
- 下拉框选择		&lt;u&gt;Select From List By Value&lt;/u&gt;			locator			values
- 单选按钮		&lt;u&gt;Select Radio Button&lt;/u&gt;			group_name			value
- 点击元素		&lt;u&gt;Click Element&lt;/u&gt;			locator			

  - id:example

  - name:example

  - xpath://div[@id=&#34;example&#34;]

  - css:div#example
- 直接提交页面的唯一表单         &lt;u&gt;Submit Form&lt;/u&gt;


 -  截图          &lt;u&gt;Screenshot&lt;/u&gt;         locator          output

- 等待元素出现          &lt;u&gt;Wait Until Page Contains Element&lt;/u&gt;             locator              timeout=None

## RPA.Excel.Files

- 打开excel		&lt;u&gt;Open Workbook&lt;/u&gt;		path		
- 读取返回表格         &lt;u&gt;Read Worksheet As Table&lt;/u&gt;      name=None     header=False     start=None
- 创建excel        &lt;u&gt;Create Workbook&lt;/u&gt;      path     fmt=xlsx       sheet_name=None
- 设置表格值      &lt;u&gt;Set Cell Value&lt;/u&gt;        row        column      value      
- 获取表格值并返回      &lt;u&gt;Get Cell Value&lt;/u&gt;        row      column      name=active sheet

# 参考资料

1. Robot Framework User Guide（[http://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html](http://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html)）
2. RPA Documentation, Training Courses, Certificates | Robocorp([https://robocorp.com/docs](https://robocorp.com/docs))
3. Keyword libraries([https://robocorp.com/docs/libraries](https://robocorp.com/docs/libraries))
4. XPATH定位的用法([https://www.cnblogs.com/aiyiless/p/16111340.html](https://www.cnblogs.com/aiyiless/p/16111340.html))

---

> 作者: ZergFlood  
> URL: https://careltian.github.io/posts/doc/rf/  

