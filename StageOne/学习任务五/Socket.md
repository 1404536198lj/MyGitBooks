# Socket
&emsp;&emsp;Unix系统中支持进程间通信（IPC），IPC的接口设计得就像是文件IO操作接口。在Unix中，一个进程会有一套可以进行读取写入的IO描述符。IO描述符可以说是文件、设备或者是通信通道（socket套接字）。一个文件描述符由三部分组成：创建（打开socket）、读取写入数据（接收和发送到socket）、销毁（关闭socket）。
<br>&emsp;&emsp;消息的目的地址是使用socket地址来表示，**一个socket地址是由网络地址和端口号组成的通信标识符。**
<br>&emsp;&emsp;进程间通信操作需要客户端和服务器端分别有一个socket对象。当一个消息发出后，这个消息在发送端的socket中处于排队状态，直到下层的网络协议将这些消息发送出去；当消息到达接收端的socket后，也会处于排队状态，直到接收端的进程对这条消息进行了接收处理。
<br>&emsp;&emsp;对于socket编程可以有两种通信协议可以选择：数据报通信和流通信。数据报通信就是UDP，UDP是一种无连接的协议，这意味着我们每次发送数据报时，需要同时发送本机的socket描述符和接收端的描述符。流通信即TCP，TCP是一种基于连接的协议，在通信之前，必须在通信的一对儿socket之间建立连接，其中的一个socket作为服务器进行监听请求，另一个作为客户端进行连接请求，一旦两个socket建立好了连接，他们可以单向或者双向进行数据传输。
<br>&emsp;&emsp;TCP还是UDP的选择。
<br>&emsp;&emsp;UDP报头有64KB的限制，发送的数据不一定会被接收端按顺序接收；而TCP一旦socket建立了连接，它们之间的通信如同IO流，没有大小限制，接收端收到的包和发送端的顺序一致。因此TCP适合远程登录和文件传输这类的网络服务，因为这些需要传输的数据大小不确定。UDP相比TCP轻量一些，用于实现实时性较高或者丢包不重要的一些服务，例如微信消息的接收发送，在局域网内，UDP的丢包率相对较低。

## 第一章 基于UDP的Socket通信Demon
### 1.1 客户端
客户端通过将数据放到byte数组中：
```java
byte[] Data = “你想要发送的文本”.getBytes();
```
并通过一个数据包对象将数据封装：
```java
DatagramPacket packetS  = new DatagramPacket(Data,Data.length,address,port);
```
最后通过DataGramSocket对象将该数据包发送：
```java
client = new DatagramSocket();
client.send(packetS);
```
（1）设置网络连接权限
```xml
在<application></application>前添加
<uses-permission android:name="android.permission.INTERNET" />
```

(2)Demon

```java
package com.ipc.lijiao.udpcilent;

import android.os.Handler;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.EditText;

import java.io.IOException;
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.net.SocketException;
import java.net.UnknownHostException;

public class MainActivity extends AppCompatActivity {

    EditText et;
    Button sd;
    int port = 8800;
    String host = "127.0.0.1";
    DatagramSocket client;
    Handler mhandler = new Handler();
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        et = (EditText)findViewById(R.id.et);
        sd = (Button)findViewById(R.id.sd);

        sd.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                String content = et.getText().toString();

                Send send = new Send(content);
                send.start();
            }
        });
    }

    public  class  Send  extends Thread{
        String content;
        InetAddress address;
        private  Send(String content)
        {
            this.content = content;
        }
        public  void run()
        {
            try {
                 address = InetAddress.getByName(host);
            } catch (UnknownHostException e) {
                e.printStackTrace();
            }
            byte[] Data = content.getBytes();
            DatagramPacket packetS  = new DatagramPacket(Data,Data.length,address,port);
            try {
                client = new DatagramSocket();
            } catch (SocketException e) {
                e.printStackTrace();
            }
            try {
                client.send(packetS);
            } catch (IOException e) {
                e.printStackTrace();
            }
            byte[] DataSend = new byte[300];
//DatagramPacket 需要新建对象，创建方式如果为packetS = new DatagramPacket(DataSend,DataSend.length);packetS仍然会指向原对象。
            DatagramPacket packetR = new DatagramPacket(DataSend,DataSend.length);

            try {
                client.receive(packetR);
            } catch (IOException e) {
                e.printStackTrace();
            }
            content = new String(packetR.getData(),0,packetR.getLength());
            mhandler.post(new Runnable() {
                @Override
                public void run() {
                    et.setText(content);
                }
            });
            client.close();
        }

    }

}
```
### 1.2 服务器端
服务器端首先需要设置一个数据接收容器：
```java
byte[] Data = new byte[300];
```
设置接收数据包：
```java
DatagramPacket packetR = new DatagramPacket(Data,Data.length);
```
通过DataGramSocket对象接收数据到DatagramPacket对象中：
```java
server = new DatagramSocket(8800);
server.receive(packetR);
```
读取数据：
```java
content = new String(packetR.getData(),0,packetR.getLength());
```
（1）设置网络连接权限
```xml
在<application></application>前添加
<uses-permission android:name="android.permission.INTERNET" />
```

（2）Demon

```java
package com.ipc.lijiao.udpserver;

import android.os.Handler;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.widget.TextView;

import java.io.IOException;
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.SocketException;

public class MainActivity extends AppCompatActivity {

    int Max_SIZE = 300;
    TextView show;
    Handler mhandler;
    String content;
    DatagramSocket server;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        show = (TextView)findViewById(R.id.show);
        mhandler = new Handler();
        Server ImpServer = new Server();
        Thread server = new Thread(ImpServer);
        server.start();
    }

    public class  Server implements   Runnable{


        @Override
        public void run() {
            byte[] Data = new byte[Max_SIZE];
            DatagramPacket packetR = new DatagramPacket(Data,Data.length);
            try {
                 server = new DatagramSocket(8800);
            } catch (SocketException e) {
                e.printStackTrace();
            }
            while (true)
            {
                try {
                    server.receive(packetR);
                } catch (IOException e) {
                    e.printStackTrace();
                }
                content = new String(packetR.getData(),0,packetR.getLength());
                mhandler.post(new Runnable() {
                    @Override
                    public void run() {
                        show.setText(content);
                    }
                });
                byte[] DataS = ("Servier："+content).getBytes();
                DatagramPacket packetSend = new DatagramPacket(DataS,DataS.length,packetR.getAddress(),packetR.getPort());
                try {
                    server.send(packetSend);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

        }
    }

    public void onDestroy(){
        super.onDestroy();
        server.close();
    }
}

```
## 第二章 基于TCP的Socket
#### 1.2.1 客户端
客户端通过服务端的host和port创建一个Socket对象：
```java
client = new Socket(host,port);
```
向服务端发送消息使用BufferedWriter对象：
```java
BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(client.getOutputStream()));
writer.write(show.getText().toString());
writer.flush();
client.shutdownOutput();
```
使用client.shutdownOutput()单向关闭写入流（如果不关闭写入流，则数据不会真正写到服务端），如果使用writer.close()会将socket对象一并关闭。

<br>demon

```java
package com.ipc.lijiao.ipcclient;

import android.os.Handler;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.EditText;

import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.io.OutputStreamWriter;
import java.io.PrintWriter;
import java.io.Reader;
import java.io.Writer;
import java.net.ServerSocket;
import java.net.Socket;

public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    EditText show ;
    Button socketB ;
    Handler mhandler = new Handler();
    Socket client;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        show = (EditText)findViewById(R.id.show);
        socketB = (Button)findViewById(R.id.socketB);
        socketB.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        switch(v.getId())
        {
            case R.id.socketB:
            {
                new Thread()
                {
                    public void run()
                    {
                       // String host = "172.17.100.77";
                        String host = "127.0.0.1";
                        int port = 8919;
                        try
                        {
                                client = new Socket(host,port);
                                BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(client.getOutputStream()));
                                writer.write(show.getText().toString());
                                writer.flush();
                                client.shutdownOutput();

                                BufferedReader reader = new BufferedReader(new InputStreamReader(client.getInputStream()));
                                String content = "";

                                while(true)
                                {
                                    String line = null;
                                    line = reader.readLine();
                                    if(line == null)
                                        break;
                                    content = content + line;
                                }
                                client.shutdownInput();

                                final String context = content;
                                mhandler.post(new Runnable() {
                                    @Override
                                    public void run() {
                                        show.setText(context);
                                    }
                                });
                            client.close();
                        }catch(IOException e)
                        {
                            e.printStackTrace();
                        }
                    }
                }.start();
            }
        }
    }

    public void onDestroy() {
        super.onDestroy();
        mhandler.removeCallbacksAndMessages(null);
        mhandler = null;
    }
}
```

#### 1.2.2 服务端
服务端会创建ServerSocket对象，并通过该对象的accept（）方法监听端口：
```java
ServerSocket server = new ServerSocket(port);
Socket socket = server.accept();
```
socket对象通过BufferReader来接收信息，并通过ReadLine读取信息：
```java
String line = null;
line = reader.readLine();
if(line == null)
    break;
```

Demon

```java
package com.ipc.lijiao.ipcserver;

import android.os.Handler;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.widget.TextView;

import java.io.BufferedInputStream;
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.io.Reader;
import java.io.Writer;
import java.net.ServerSocket;
import java.net.Socket;

import javax.security.auth.Destroyable;

public class MainActivity extends AppCompatActivity {
    TextView show;
    Handler mhandler = new Handler();
    ServerSocket server;
    Socket socket;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        show = (TextView)findViewById(R.id.textView);
        AcceptClient();
    }

    private void AcceptClient()
    {
        new Thread()
        {
            @Override
            public void run()
            {
                    try
                    {
                        int port = 8919;
                        server = new ServerSocket(port);
                        while(true)
                        {
                            socket = server.accept();
                            String content = "";
                            BufferedReader reader=new BufferedReader(new InputStreamReader(socket.getInputStream()));
                            while (true)
                            {
                                String line = null;
                                line = reader.readLine();//读取客户端传来的数据
                                if (line == null)
                                    break;
                                content = content + line;
                            }
                            socket.shutdownInput();
                            final String context = content;
                            mhandler.post(new Runnable() {
                                @Override
                                public void run() {
                                    show.setText(context);
                                }
                            });
                            BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
                            writer.write("客户端发送到服务端的消息为："+context);
                            writer.flush();
                            socket.shutdownOutput();
                           
                        }
                    }
                    catch(IOException e)
                    {
                        e.printStackTrace();
                    }
            }
        }.start();
    }

    public void onDestroy() {
        super.onDestroy();
        try {
            socket.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        mhandler.removeCallbacksAndMessages(null);
        mhandler = null;
        try {
            socket.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        try {
            server.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}
```

































