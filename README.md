# SpirentAPI
Spirent测试仪当前仅提供了TCL语言的接口，Python的接口需要收费且不稳定，因此使用Python封装了TCL的接口，为大家的自动化测试提供方便。

## SpirentAPI的初始化
本段描述了SpirentAPI的实现原理，并引用部分代码作为示例。

### 把Spirent TestCenter Application文件夹设置为TCL的搜索路径
Python调用TCL，并且把Application设置为TCL的查找路径
``` python
    def __init__(self):
        self.tclsh = Tkinter.Tcl()
        self.stcpkg = 'C:/Program Files (x86)/Spirent Communications/Spirent TestCenter 4.64/Spirent TestCenter Application'
        self.tclsh.eval("set auto_path [ linsert $auto_path 0 %s ]" %(self.stcpkg))
        print self.tclsh.eval("puts $auto_path")
        self.tclsh.eval("package require SpirentTestCenter")
```

### 组装TCL命令
由于TCL命令参数较多，专门写了一个函数用于组装TCL命令
``` python
    def build_cmd(*args):
        cmd = ''
        for arg in args:
            cmd = cmd+str(arg)+' '
        return cmd
```

### 对常用的TCL命令封装
一些常用的命令使用频率较高，进行封装，减少冗余代码
``` python
    # stc create
    def stc_create(self,*args):
        cmd = build_cmd('stc::create', *args)
        return self.tclsh.eval(cmd)
    # return: project_name
    def stc_create_project(self):
        return self.stc_create('project')
```
## SpirentAPI执行过程梳理
对Spirent测试仪的执行过程进行梳理，对其执行的过程有个大概的了解
### 确定使用测试仪的匡、槽、口等信息


