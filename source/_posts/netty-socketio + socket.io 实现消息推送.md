---
title: netty-socketio + socket.io 实现消息推送
date: 2018-04-11 17:12:53
---

socket.io 和 netty-socketio 结合实现命名空间的推送时，socket.io的版本必须使用 1.7.4 这个版本。不能使用 2.0.0 以上的版本。

``` bash
// https://mvnrepository.com/artifact/com.corundumstudio.socketio/netty-socketio
compile group: 'com.corundumstudio.socketio', name: 'netty-socketio', version: '1.7.14'

https://cdnjs.cloudflare.com/ajax/libs/socket.io/1.7.4/socket.io.min.js
```

server: 

``` java

    @Test
    public void test() throws InterruptedException {
        Configuration config = new Configuration();
        config.setHostname("localhost");
        config.setPort(8899);

        final SocketIOServer server = new SocketIOServer(config);
        server.start();
        String uid = "1111";
        String namespace = String.format("/%s_%s", "msg", uid);//构建命名空间
        SocketIONamespace chat1namespace = server.addNamespace(namespace); //设置命名空间

        for (int i = 0; i < Integer.MAX_VALUE; i++) {
            Thread.sleep(2000);
            chat1namespace.getBroadcastOperations().sendEvent("message", 1); //每次发送数字一
            System.out.println("room size: " + chat1namespace.getAllClients().size());
        }


        server.stop();
    }
```

client

``` html
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
<title>Insert title here</title>
<!--   <script src="./jquery-1.9.1.js" type="text/javascript"></script> 
  <script type="text/javascript" src="./socket.io/socket.io.js"></script>
    <script src="http://apps.bdimg.com/libs/socket.io/0.9.16/socket.io.min.js" type="text/javascript"></script>
	  <script type="text/javascript" src="./socket.io.js"></script>
	  
	   ok 
	  <script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/1.7.4/socket.io.min.js"></script>
   -->
  
  <script src="http://apps.bdimg.com/libs/jquery/2.1.4/jquery.min.js" type="text/javascript"></script>
 <script type="text/javascript" src="./socket.io.js"></script>

  
  
    <style>
        body { 
            padding:20px;
        }
        #console {  
            overflow: auto; 
        }
        .username-msg {color:orange;}
        .connect-msg {color:green;}
        .disconnect-msg {color:red;}
        .send-msg {color:#888}
    </style>

</head>

<body>
    
    <h4>Netty-socketio Demo Chat</h4>
    
    <br/>

    <div id="console" class="well">
    </div>
    消息总数：<div id="msgnum">0</di>    
</body>

  
         
<script type="text/javascript">

 var socket =  io.connect('http://192.168.1.89:8899/message');

        socket.on('connect', function() {
            output('<span class="connect-msg">Client has connected to the server!</span>');
        });

        socket.on('message', function(data) {//收到消息后，将消息总数加一
            var num = $("#msgnum").html();
            num = parseInt(num) + data;
            $("#msgnum").html(num);
        });
		
		socket.on('test', function(data) {//收到消息后，将消息总数加一
            $("#msgnum").html(data);
        });
        
        socket.on('disconnect', function() {
            output('<span class="disconnect-msg">The client has disconnected!</span>');
        });
        function sendDisconnect() {
            socket.disconnect();
        } 
         
        function output(message) {
            var currentTime = "<span class='time'>" +  new Date() + "</span>";
            var element = $("<div>" + currentTime + " " + message + "</div>");
            $('#console').prepend(element);
        }
        
    </script>
</html>
```

## 参考
1. [socket.io](https://cdnjs.com/libraries/socket.io/1.7.4)
2. [Add support for socket.io-client 2](https://github.com/mrniko/netty-socketio/issues/473)
3. [netty-socketio使用namespace](http://www.cnblogs.com/always-online/p/4133405.html)
4. [netty-socketio实时推送信息](https://blog.csdn.net/Shaun_luotao/article/details/78192711)