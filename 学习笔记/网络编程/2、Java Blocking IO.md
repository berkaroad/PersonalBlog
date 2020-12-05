# Java Blocking IO

使用Socket建立网络连接，ServerSocket作为服务端使用，用于网络监听。

```java
public static void main(String[] args) {
    ServerSocket serverSocket = new ServerSocket();
    serverSocket.bind(new INetSocketAddress("127.0.01", 7001));
    while(true) {
        Socket socket = serverSocket.accept();
        Thread handleThread = new Thread(new RequestHandler(socket));
        handleThread.start();
    }
}

private class RequestHandler implements Runnable {
    private Socket socket;
    public RequestHandler(Socket socket) {
        this.socket = socket;
    }
    
    public void run() {
        //TODO: 读写IO
        InputStream in = socket.getInputStream();
        OutputStream out = socket.getOutputStream();
        
    }
}
```



