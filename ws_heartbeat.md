# websocket心跳重连
## 心跳重连
使用websocket过程中，由于信号不好或网路临时关闭等可能出现的网络断开情况时，websocket连接断开，而浏览器不会执行onclose方法，我们无法知道是否断开，也就无法进行重连。
但若当前发送webscoket数据到后端，一旦请求超时，onclose便会执行，这时便可进行绑好的重连。
---
## 实现
* **websocket实例化**
```javascript
let ws = new WebSocket(url);
ws.onclose = function () {
	//something
};
ws.onerror = function () {
	//something
};
ws.onopen = function () {
	//something
};
ws.onmessage = function () {
	//something
};
```
> * 要websocket连接一直保持，通常在close或error上绑定重连方法
```javascript
//example
ws.onclose = function () {
	reconnect();
};
ws.onerror = function () {
	reconnect();
};
```
* **实现逻辑**

>前端ws.send的时候，就是发送了一个心跳，如果后端收到了，应该向前端返回一个心跳回应，前端拿到了，就说明连接是正常的，就不会重连

1. 什么条件下执行
每隔一段时间给服务器发送一个消息，无回复便执行：
onopen连接上时，开始start计时，在定时范围内，onmessage获取到后端信息就重新倒计时。

2. 判断前端ws断开
当心跳检测send方法执行后，如果ws是断开状态，发送超时后，浏览器ws会自动触发onclose方法，在onclose方法绑定重连事件，如果当前一直断网，重连会time执行一次直到重建连接。

> 由于不同浏览器实现不同，如firefox断网7秒，ie断网10秒后直接执行onclose，不需要心跳检测。导致reconnect可能会执行多次。因此最后个reconnect()方法加上一个锁保证只执行一次

3. 判断后端ws断开
后端在可控情况下断开ws会先发送一个断连消息，随后断开，这时我们便会重连。
异常断开，我们无法感应，但我们发送心跳一定时间后，后端没返回心跳响应消息，前端有没收到任何其他信息，我们就可以认为后端主动断开了。

* **具体实现**
```javascript
let ws;//实例
let lockReconnect = false;//避免重复连接
let heartCheck; //心跳检测
const wsUrl = 'ws:xxx.1.1.1';

function createWebSocket(url) {
	try {
		ws = new Websocket(url);
		initEventHandle();
	} catch (e) {
		reconnect(url);
	}
}

function initEventHandle() {
	ws.onclose = function () {
		reconnect(wsUrl);
	};
	ws.onerror = function () {
		reconnect(wsUrl);
	};
	ws.onopen = function () {
		//心跳检测重置
		heartCheck.reset().start();
	}；
	ws.onmessge = function (event) {
		//获得任何信息，心跳检测重置
		heartCheck.reset().start();
	}
}

function reconnect(url) {
	if(lockReconnect) return;//避免重复连接
	lockReconnect = true;
	//设置延时避免请求过多
	setTimeout(function () {
		createWebSocket(url);
		lockReconnect = false;//重置锁
	},2000);
}

//心跳检测
let heartCheck = {
	timeout: 6000,//60s
	timeoutObj: null,
	serverTimeoutObj: null,
	reset: function () {
		clearTimeout(this.timeout);
		clearTimeout(this.serverTimeout);
		return this;
	},

	start: function () {
		let self = this;
		this.timeoutObj = setTimeout(fuction(){
			//发送心跳，后端收到返回心跳消息说明连接正常
			ws.send("HeartBeat");
			self.severTimeoutObj = setTimeout(function(){
				ws.close();
			},self.timeout);
		},this.timeout);
	}
};

creatWebSocket(wsUrl);
```