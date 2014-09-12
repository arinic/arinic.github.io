#TV助手虚拟麦克风#
##功能简介##
TV助手虚拟麦克风功能主要是为了解决盒子在没有麦克风的情况下无法进行语音输入的问题，PD及UED目前对该功能的定义如下：

1. 虚拟麦克风与语音搜索使用同一个入口，即目前的语音搜索按钮。
2. 虚拟麦克风功能为被动功能，只有当TV端打开特定的应用(需要语音输入支持，如视频通话)时，手机端的语音按钮对应功能变为虚拟麦克风，否则为默认的语音搜索功能。
3. 支持功能扩展，如加入音频回传功能。


##模块交互##

虚拟麦克风涉及三个模块，手机端的TV助手麦克风模块，盒子端的input\_boost与虚拟音频设备virtual\_audio\_dev，其中input\_boost为virtual\_audio\_dev提供命令数据通道与TV助手打交道。
<img src="http://i.imgur.com/zLFnOuu.png" width=600 style="margin:auto;display:block;border:1px solid black;"  />

技术实现上需要考虑以下细节问题：

1. 多设备支持及踢人策略。多个手机端同时使用该功能，目前不支持混音，使用独占策略，***踢人策略***为先到者独占使用该功能。
2. 盒子端需要保证及时感知移动端退出虚拟麦克风，切换到正常的音频输入逻辑，需要考虑到异常情况，如网络中断。该问题使用心跳解决，当虚拟音频设备被某个手机客户端成功占据后，virtual\_audio\_dev发起心跳。


共用连接由input\_boost,input\_boost需要有需要有客户端概念，不建议完全使用ip来标识，原因这限制了一台设备只能有一条链接，不合理。
<img src="http://i.imgur.com/c93TwaV.png" width=700 style="margin:auto;display:block;border:1px solid black;"  />



##交互协议##
交互协议包括手机端与input\_boost，input\_boost与virtual\_audio\_dev,要求所有协议使用**UTF-8编码**。




- **功能查询请求**

	TV助手连接上input\_boost后，发起功能查询请求，input\_boost返回其支持的功能列表
>
	{
		  "version" : "1.0.1",//input_boost版本
		  "dev_name" : "我的魔盒",
		  "dev_model_name" : "MagicBox",
		  "modules":
					[
						 {
							 "name": "virtual_audio_dev",
                             "version": "1.0.1",
                             "version_code": "2100100001"
							 "features": 1, //按照位标记功能， 0位对应-是否支持虚拟音频输入， 1—>
							 "mic_data_port": 39900, //接收音频数据的接口
							 "mic_data_port_type": 0, //0--> UDP, 1 --> TCP, 2 --> TCP&UDP
						 }
					]；
	}
 其中features字段为一个整型，其二进制由低位到高位，每一位的含义如下：
	<table>
		<tbody>
			<tr><td><em>二进制位</em></td><td><em>含义</em></td></tr>
			<tr><td>第0位</td><td>是否支持虚拟语音输入</td></tr>
			<tr><td>第1位</td><td>是否支持语音输出</td></tr>
		</tbody>
	</table>

- **请求开启语音输入**
	
    TV助手发送给input_boost的命令中数据，input_boost需要定义一个数据包COMMON_CMD，代表发送给第三方模块的。

>
	{
		  "dest_module":"virtual_audio_dev",
		  "module_data":
					 {
						 "oper": 0x1001,
                     }
	}
input\_boost需要将客户端ID号，IP信息，及module_data传递给virtual_audio_dev，由其处理。

input\_boost回应请求结果，数据包COMMON_CMD_RSP

>
	{
		  "dest_module":"virtual_audio_dev",
		  "module_data":
					 {
						 "oper": 0x1001,
                         "res": 0,
						 "dat":""
                     }
	}

<table>
	<tbody>
		<tr><td><em>oper值</em></td><td><em>含义</em></td></tr>
		<tr><td>0x1001</td><td>心跳</td></tr>
		<tr><td>0x1002</td><td>开启虚拟麦克</td></tr>
		<tr><td>0x1003</td><td>关闭虚拟麦克</td></tr>
	</tbody>
</table>

<table>
	<tbody>
		<tr><td><em>res值</em></td><td><em>含义</em></td></tr>
		<tr><td>0</td><td>操作成功</td></tr>
		<tr><td>-1</td><td>操作失败</td></tr>
	</tbody>
</table>




##目前存在的问题##
###input_boost相关###
1. 目前input_boost与TV助手链接是一条TCP链接，这条TCP链接上传递了以下这类数据：鼠标，按键，体感，前台应用变更(盒子到手机)。供第三方模块，如虚拟麦克风模块使用的命令通道使用这条TCP链接是否会有问题？特别是在发送体感数据时。
2. 查询未支持的接口功能，需要明确回应不支持。




