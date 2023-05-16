# mac上运行bat

我们使用windows时经常会用到批处理文件。双击下.bat文件。大量的工作就会替你完成。这功能够酸爽。mac下是否有同样的功能那？答案是肯定的。接下来就会介绍mac系统如何执行批处理。

### 方法/步骤

新建一个文本文件。命名为test.command。注意一定要以command为结尾。

![](<../.gitbook/assets/image (152).png>)



打开『终端』程序。给test.command加上可执行权限。命令为chmod +x test.command.

![](<../.gitbook/assets/image (133).png>)

用编辑器打开文本文件。输入可以在终端执行的指令。例如ls

![](<../.gitbook/assets/image (128).png>)

将文件保存到桌面上。然后双击打开。效果如图所示：

![](<../.gitbook/assets/image (71).png>)

根据自己的需求。编写你自己的批处理文件把。这里有一个问题。就是批处理执行完了。弹出的那个命令行不会自动关闭。为了解决这个问题。在批处理文件的最后加上代码：osascript -e 'tell application"Terminal" to close (every window whose name contains".command")' \&exit

![](<../.gitbook/assets/image (279).png>)

现在就和windows的bat文件体验完全相同。值得提醒的是在mac系统下你还可以编写shell脚本。shell脚本的语法在网上就可以找到。相信您可以做出非常使用的批处理文件。![mac系统如何执行批处理](https://exp-picture.cdn.bcebos.com/e3d059e833e0397282f14459b58630486043564a.jpg?x-bce-process=image%2Fresize%2Cm\_lfit%2Cw\_500%2Climit\_1)

### 注意事项

* 一定记得给文件可执行权限。否则脚本服务执行
* mac系统中的删除功能谨慎使用。因为恢复起来比较难
