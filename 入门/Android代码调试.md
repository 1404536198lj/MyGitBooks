# Android 代码调试

1.打印方法调用栈
>有时候方法会有多层调用，为了能够查看出现问题的方法是被哪些方法调用了，通常会打印调用栈。
```java
package lijiao.android.com.dosometest;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;

public class MainActivity extends AppCompatActivity {
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        try {
            PrintStackTraceTest.A();
        }
        catch (Exception e){
            e.printStackTrace();
        }
    }
}
```
>方法调用栈是从包含try catch 的方法体打印到该方法体抛出异常的方法
```java
package lijiao.android.com.dosometest;

/**
 * Created by lijiao on 17-9-19.
 */

public class PrintStackTraceTest {
    public static void A() throws Exception {
        B();
    }

    public static void B() throws Exception {
        C();
    }

    public static void C() throws Exception {
        last();

    }
    public static void last() throws Exception {
        throw new Exception();
    }
}
```
>调用栈打印的内容为：
```shell
09-19 15:26:36.804 11548 11548 W System.err: 	at lijiao.android.com.dosometest.PrintStackTraceTest.last(PrintStackTraceTest.java:21)
09-19 15:26:36.804 11548 11548 W System.err: 	at lijiao.android.com.dosometest.PrintStackTraceTest.C(PrintStackTraceTest.java:17)
09-19 15:26:36.804 11548 11548 W System.err: 	at lijiao.android.com.dosometest.PrintStackTraceTest.B(PrintStackTraceTest.java:13)
09-19 15:26:36.804 11548 11548 W System.err: 	at lijiao.android.com.dosometest.PrintStackTraceTest.A(PrintStackTraceTest.java:9)
```
