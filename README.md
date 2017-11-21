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



用户上线模块要点

客户端

```
var from = $.cookie('user');
var to = "all";
//通知已经上线的用户，谁来了
socket.emit('online',{user:from});
//该客户端自己也要接收谁上线的消息，有可能这个cookie是自己，要判断一下
socket.on("online",function (data) {
  if(data.user!==from) {
    var sys = '<div>系统提示('+now()+'):'+'用户'+'data.user'+"上线了"+"</div>";
  } else {
    var sys  = '<div>系统提示('+now()+'):你进入了聊天室！</div>' ;
  }
  //刷新一下用户列表
  flushUsers(data.users);
  //显示正在对谁说话
  showSayTo();
})
```



```
//向连接的客户端建立监听
socket.on("online",function (data) {
  socket.name = data.user;
  if(!users[data.user]){
    users[data.user] = data.user;
  }
  io.sockets.emit('online',{user:data.user,users:users});
})
```

注意：我们把存储用户名的操作放到用户上线的事件里（即 `socket.on('online')` 里），而不是用户登录时把直接用户名存储到数组。这是因为 **“用户上线添加用户名，用户下线删除用户名”** 这是一个对称的操作。假如用户登录时把用户名存储到数组，用户下线时从数组中删除用户名，那么当用户刷新聊天室页面或者关闭然后重新打开页面时（都会触发一个下线并上线的动作），users 数组中就会只删除而不再重新添加该用户，即右侧的 “在线用户” 列表也不会显示该用户名了



用户通话模块

用户发送消息的部分

```
$("#say").click(function () {
  var $msg = $("#input_content").html();
  if($msg) return ;
  if(to==='all') {
    $("#contents").append('<div>你(' + now() + ')对 所有人 说：<br/>' + $msg + '</div><br />');
  } else {
    $("#contents").append('<div style="color:#00f" >你(' + now() + ')对 ' + to + ' 说：<br/>' + $msg + '</div><br />');
  }
   //发送发话信息
  socket.emit('say', {from: from, to: to, msg: $msg});
  //清空输入框并获得焦点
  $("#input_content").html("").focus();
})
```

用户需要接受消息

```
socket.on('say',function(data) {
  //对所有人说
  if(data.to=='all') {
     $("#contents").append('<div>' + data.from + '(' + now() + ')对 所有人 说：<br/>' + data.msg + '</div><br />');
  }
  //对我私聊
  if(data.to==from) {
     $("#contents").append('<div style="color:#00f" >' + data.from + '(' + now() + ')对 说：<br/>' + data.msg + '</div><br />');
  }
})
```

服务端需要处理

```
//有人发话
socket.on('say',function (data) {
  if(data.to=='all'){
    socket.broadcast.emit('say',data);
  } else {
    //向特定的用户发送
    //clients存储所有连接对象的数组
    var clients = io.sockets.clients();
    //遍历找到用户
    clinets.forEach(function (client) {
      if(client.name==data.to){
      //触发该用户端的say事件，该用户将接收到消息
        client.emit('say',data);
      }
    })
    
  }
})
```

用户下线模块

服务端的处理

```
//有人下线的话
socket.on('disconnect',function () {
  if(users[socket.name]) {
    delete users[socket.name];
    socket.broadcast.emit('offline',{users: users,user:socket.name});
  }
})
```

客户端

```
socket.on('offline', function (data) {
  //显示系统消息
  var sys = '<div style="color:#f00">系统(' + now() + '):' + '用户 ' + data.user + ' 下线了！</div>';
  $("#contents").append(sys + "<br/>");
  //刷新用户在线列表
  flushUsers(data.users);
  //如果正对某人聊天，该人却下线了
  if (data.user == to) {
  
  }
  //显示正在对谁说话
  showSayTo();
});
```

