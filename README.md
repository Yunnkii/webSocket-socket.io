#### 学习使用 socket.io 完成前后端双向通信

socket.io  文件版本对运行结果有影响

客户端基本用法：

```
<script src = "/socket.io/socket.io.js"></script>
<script>
	var socket = io.connect("http://localhost");
	socket.on("connect",function (data) {
      console.log(data);
      socket.emit("mu other thing",{my:"data"});
	})
</script>
```



我们就创建了一个socket连接，这里需要强调一点，默认情况下，无论我们创建了多少次socket连接，指向都是同一个socket实例，除非我们在connect()的第二个参数中指定"force new socket"强制建立一个新的连接

刚刚创建一个socket的时候，其实这只是一个空的socket，而且并未分配一个id，只有当与服务器连接成功后，才会生成一个id，因此我们在connect中才能得到获取socket的id，有些时候，socket的id是很有用的，有时会用于存储对应socket的用户信息。socket连接成功后，服务端与客户端会定时发送数据包，若某一方长时间未回应，默认连接断开，当我们的网络不稳定或者有其他影响连接的因素存在的时候，则会与服务器断开连接，断开时触发disconnect监听，如果没有声明reconnect 就不再重新连接，如果在创建的时候声明reconnect:true，在客户端会尝试新的连接，会触发reconnecting监听，若连接未成功，回程时多次连接，若连接成功，就会触发reconnect监听，原有的socket已经不在，内部会创建一个新的连接，socket 会得到一个新的id，会再一次触发connect监听。



server

```
var app = require('express')();
var server = require('http').createServer(app);
var socket = require('socket.io').listen(server);

app.get('/',function (req,res) {
  res.sendFile(__dirname+"/idnex.html");
});
io.sockets.on('connection',function (socket) {
  socket.emit('news',{hello:'world'});
  socket.on('my other thing',function (data) {
    console.log(data);
  })
})
```



socket.io提供了三种默认的方法socket.connect，socket.message,socket.disconnect，同时可以由用户自定义很多方法。

服务器端需要区分三种发送信息的方法：

- socket.emit()向建立连接的客户端广播
- socket.broadcast.emit()向没有建立连接的客户端广播
- io.sockets.emit()，向所有客户端广播