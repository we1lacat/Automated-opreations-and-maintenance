jenkins

 是自动化构建、测试的重要工具，Java开发——需要maven+jdk一同工作

安装

部署jenkins服务器

初始化配置

插件引入

UI Configure、

自由式入门

有git repo任务的构建

凭证配置

master/branch的一些区别

日志的查看

开始构建

初识maven任务

初识docker镜像构建

dockerhub

私有仓库

pipline的概念

插件限制和脚本式的灵活

jenkinsfile

```
pipline{
	agent any 
	tools{
	maven "maven-3.9"
	jdk "jdk21"
	}
	paameter{
	striong(namme:'mynanme',defaultValue:'hello',descrption:'something ')
	}
	satges{

		stage{

				step{
					script{
					
					}
					expression

			}
		}
	}
}
```

环境变量

 适用脚本使用

全局可用的变量

凭据引用插件

参数

 适用于表达式使用

函数

输入参数-局部变量/全局变量

 提供构建中的可输入参数，版本，工件等的选则

pipline构建流程

多分支pipline

自动创建流水线

凭据

 系统凭据适用于Jenkins job

 全局凭据适用于所有

项目凭据限于一个pipline

 基本的种类

微服务

 共享库的概念

自动触发构建

jenkins提交到git

触发器ignore

 避免循环构建