# SpirentAPI
Spirent测试仪当前仅提供了TCL语言的接口，Python的接口需要收费且不稳定，因此使用Python封装了TCL的接口，为大家的自动化测试提供方便。

## 一、SpirentAPI的初始化
本段描述了SpirentAPI的实现原理，并引用部分代码作为示例。

### 1、把Spirent TestCenter Application文件夹设置为TCL的搜索路径
Python调用TCL，并且把Application设置为TCL的查找路径
``` python
    def __init__(self):
        self.tclsh = Tkinter.Tcl()
        self.stcpkg = 'C:/Program Files (x86)/Spirent Communications/Spirent TestCenter 4.64/Spirent TestCenter Application'
        self.tclsh.eval("set auto_path [ linsert $auto_path 0 %s ]" %(self.stcpkg))
        print self.tclsh.eval("puts $auto_path")
        self.tclsh.eval("package require SpirentTestCenter")
```

### 2、组装TCL命令
由于TCL命令参数较多，专门写了一个函数用于组装TCL命令
``` python
    def build_cmd(*args):
        cmd = ''
        for arg in args:
            cmd = cmd+str(arg)+' '
        return cmd
```

### 3、对常用的TCL命令封装
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
## 二、SpirentAPI执行过程梳理
对Spirent测试仪的执行过程进行梳理，对其执行的过程有个大概的了解
### 1、确定使用测试仪的匡、槽、口等信息
测试仪的管理IP、Slot的编号，端口的信息需要提前确定下来
``` Python
    # Create the root project object
    hProject = stc.stc_create_project()
    # Create ports
    hPortTx = stc.stc_create_port(hProject,ChassisIp,str(Slot),str(TxPort))
    hPortRx = stc.stc_create_port(hProject,ChassisIp,str(Slot),str(RxPort))
```
### 2、配置工程参数信息并应用
配置该工程的速率参数为RATE_BASED，等等
``` Python
    # Configure physical interface.
    hPortTxCopperInterface_tx = stc.stc_create("EthernetCopper","-under",hPortTx)
    hPortTxCopperInterface_rx = stc.stc_create("EthernetCopper","-under",hPortRx)
    
    #ConfigProperties
    ConfigProperties = stc.stc_get(hProject,"-children-port")
    cmd = '{ -generator.generatorConfig.schedulingMode RATE_BASED -generator.generatorConfig.durationMode CONTINUOUS }'
    for port in ConfigProperties.split():
        stc.stc_perform("ConfigProperties","-objectlist",port,"-propertylist",cmd)
    #应用当前的配置
    stc.stc_perform("attachPorts")
    stc.stc_apply()
```
### 3、配置收发端口及其相关IP、MAC等参数
需要配置发端的IP、MAC、网关等参数
``` Python
   # Retrieve the generator and analyzer objects.
    hGenerator = stc.stc_get(hPortTx,"-children-Generator")
    hAnalyzer = stc.stc_get(hPortRx,"-children-Analyzer") 
    # Retrieve the generator and analyzer objects.
    hStreamBlock = stc.stc_create_streamblock(hPortTx,TxParas["srcMac"],TxParas["dstMac"],TxParas["sourceAddr"],TxParas["destAddr"],TxParas["gateway"],TxParas["speed"])
    hStreamBlockRx = stc.stc_create_streamblock(hPortRx,RxParas["srcMac"],RxParas["dstMac"],RxParas["sourceAddr"],RxParas["destAddr"],RxParas["gateway"],RxParas["speed"])
```
stc_create_streamblock函数的具体配置为
``` Python
    # return: streamblock name
    def stc_create_streamblock(self,port_name,srcMac,dstMac,sourceAddr,destAddr,gateway,speed):
        hStreamBlock = self.stc_create('streamBlock -under',port_name,\
                               '-insertSig','true',\
							   '-frameConfig','""',\
							   '-frameLengthMode', 'FIXED',\
							   '-maxFrameLength', '1200',\
							   '-FixedFrameLength', '256',\
							   '-LoadUnit','KILOBITS_PER_SECOND',\
							   '-Load',speed)
	#Adding headers
        hEthernet = self.stc_create('ethernet:EthernetII -under',hStreamBlock,\
                               '-srcMac',srcMac,\
							   '-dstMac',dstMac)
        #Adding ipv4					   
        hE = self.stc_create('ipv4:IPv4 -under',hStreamBlock,\
                               '-sourceAddr',sourceAddr,\
							   '-destAddr',destAddr,\
                               '-gateway',gateway )
        return  hStreamBlock
```
### 4、收发端口速率、流量信息实时记录
实际测试中，需要对收发端口的速率、流量信息、收发帧数量等信息实时记录，以便对测试结果进行判断
``` Python
    # Subscribe to realtime results.
    hAnalyzerConfig = stc.stc_get(hAnalyzer,"-children-AnalyzerConfig")    
    subscribeRx = {"Parent":hProject, \
                "ConfigType": "Analyzer", \
                "resulttype": "AnalyzerPortResults",  \
                "filenameprefix": "Rx_Port_Results"}
    
    subscribeTx = {"Parent":hProject, \
            "ConfigType": "Generator", \
            "resulttype": "GeneratorPortResults",  \
            "filenameprefix": "Tx_Port_Results"}    
    TxResults = stc.stc_config_subscribe(**subscribeTx)
    RxResults = stc.stc_config_subscribe(**subscribeRx) 
    # Apply configuration. 
    stc.stc_apply()
    # Save the configuration as an XML file. Can be imported into the GUI.
    stc.stc_perform("SaveAsXml")
```
### 5、开始进行打流
首先接收端开启，然后再开启发送端，确保没有流量丢失
``` Python
    # Start the analyzer and generator.
    stc.stc_perform("AnalyzerStart","-AnalyzerList",hAnalyzer)
    stc.stc_perform("GeneratorStart","-GeneratorList",hGenerator)
```
### 6、获取实时速率
打流启动后，可以获取当前实时速率
``` Python
    #获取实时速率
    hAnalyzerResults = stc.stc_get(hAnalyzer,"-children-AnalyzerPortResults")
    hGeneratorResults = stc.stc_get(hGenerator,"-children-GeneratorPortResults")
    sleep(5)   
    RxRate = stc.stc_get(hAnalyzerResults,"-SigFrameRate")
    TxRate = stc.stc_get(hGeneratorResults,"-GeneratorSigFrameRate")
```
### 7、停止打流并删除工程
测试完成后要停止打流并删除工程，不耽误下次的自动化执行
``` Python
    # Stop the generator. 
    stc.stc_perform("GeneratorStop","-GeneratorList",hGenerator)
    sleep(1)
    stc.stc_perform("AnalyzerStop","-AnalyzerList",hAnalyzer)
    # Disconnect from chassis, release ports, and reset configuration.
    stc.stc_perform("ChassisDisconnectAll")
    stc.stc_perform("ResetConfig")
    stc.stc_delete(hProject)  
```

## 末尾
核心是调用TCL的接口，外面再用Python包装了一层，基本的功能示例上都有了，参考即可。其他更复杂的功能可以找Sprient的接口文档参考。未尽事项可以加微信 `15321254618` 一起学习。





