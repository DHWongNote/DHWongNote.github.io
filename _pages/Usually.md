
## WinApp 
### 搜索
	Listry

### Code
	Vscode
	Rider













### WinU

F11 全屏

Ctrl+p 浏览器 (打印/导出PDF) 

### Visual Assistant快捷键
alt+o：换 .h 和 .cpp

alt+shift+o：全局找文件

alt+shift+s：全局找符号

alt+shift+f：全局找当前光标符号

f8：ctrl+shift+f 之后 f8，查找下一个当前光标符号的引用，这个不是 vax 的功能，是 vs 自带的，但也勉强算上吧

alt+g：找定义

alt+shift+g：找继承关系

alt+shift+q：添加 include

alt+shift+r：重构

ctrl+k,ctrl+s：包裹代码

alt+m：当前文件方法

vassistx.gotomember：我改成了 alt+shift+m，作为 alt+m 的补充，当前文件局部变量

vassistx.smartselectextend，我替换了默认的 ctrl+w：选择当前文字，可以继续扩展

vassistx.smartselectshrink 我改成了 ctrl+shift+w：上面一个操作的逆操作


还有一个就是如果VS没有安装在固态硬盘，把VA_X.dll的目录mklink /j 符号链接到ssd上面会快很多。


### VSCODE

1.Ctrl + Shift + P 调出主命令框，输入 Markdown，应该会匹配到几项 Markdown相关命令

vscode选中多个数字如何自增: 插件：Increment Selection： ctrl+alt+I

Ctrl+D ：选择多个鼠标点，搜索内容


### DOS

强关进程
toolbag.exe   —12444 Console  19,144 K

C : \Users \Administrator>taskkill /f /t /im toolbag.exe

通配符		路径操作符		        UNIX 用/ DOS用\						
			?		                仅一个字符						
			 /s		                输出目录及其所有子目录下所有文件						
			 || 	                	 当第一条命令执行错误时，后面才执行						
			  &&	                	 第一条命令执行失败后面不会执行						
			 &		                组合命令 			 ，当第一个命令执行失败，后面的命令会继续执行			
			 >或者>>                		  文件重定向			 echo 我是内容 >d:\1.txt  写入文本到指定文件（如果文件存在则替换）			
			*		                所有匹配字符						
Normal		dir		                列出当前目录下的目录和文件  /ad 只输出其子目录	 找到D盘里面含有c这个字母的文件夹，我们可以输入：dir f:\ | find "c" /s 输出目录及其所有子目录下所有文件			

			md		                    创建目录						
			rd		                    删除目录						
			cd\		                    回到根目录						
			cd		                    切换当前目录						
			del		                    删除文件						
			copy                        xcopy		拷贝文件						
			ren		                    重命名						
			move	                	移动						
			mklink	                	创建符号链接						
			save	                	将虚拟磁盘序列化到一个文件中						
			load	                	从一个文件中反序列化为虚拟磁盘						
			cls		                    清屏						
Pro			Start                       可以打开盘、文件、文件夹、网址、程序			 start D: 打开D盘			
            start /max D:\              以最大化打开D盘			
            start D:\Sheet.pdf          打开D盘下面的Sheet.pdf文件			
            start www.baidu.com         打开百度网站" 如果打开文件时，路径或者文件名称有空格，需要将该文件名称用””引起来。
            md D:\”picture 1”           在D盘下新建一个名叫picture 1的文件夹
            start """" ""picture 1""    打开有空格的文件夹"			
			 Call		                可以用来调用代码 call echo.bat   如果echo.bat文件在相同的文件夹下，可以直接调用相对路径，否则应该调用绝对路径。						
			 echo		                显示文本   显示消息，或者启用或关闭命令回显。			@echo off:关闭文本的显示			
											
											
											
											
			WIFI    netsh wlan show profiles			        显示电脑连接过的wifi名称		 
                    netsh wlan show profile name="WD" key=clear 查看机器上名字为WD的密码			 
			防火墙  netsh firewall set opmode mod=disable       关闭	 
					netsh firewall set opmode mod=enable        开启		 
			隐藏    attrib +h 1.txt 			                隐藏		 
					attrib -h 1.txt			                    显示			 
			防火墙  netsh firewall set opmode mod=disable       关闭	 
					netsh firewall set opmode mod=enable        开启			
											
											
											
											


### MarkDown

```顶顶顶顶顶顶顶顶顶顶顶顶顶顶顶
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

    // My trigger component
    UPROPERTY()
    UPrimitiveComponent* Trigger;

    AMyActor()
    {
        Trigger = CreateDefaultSubobject<USphereComponent>(TEXT("TriggerCollider"));

        // Both colliders need to have this set to true for events to fire
        Trigger.bGenerateOverlapEvents = true;

        // Set the collision mode for the collider
        // This mode will only enable the collider for raycasts, sweeps, and overlaps
        Trigger.SetCollisionEnabled(ECollisionEnabled::QueryOnly);
    }

    virtual void NotifyActorBeginOverlap(AActor* Other) override;

    virtual void NotifyActorEndOverlap(AActor* Other) override;
};
```

 