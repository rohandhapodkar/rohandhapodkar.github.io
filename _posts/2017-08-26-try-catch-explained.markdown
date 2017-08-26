---
layout: post
title:  "Try with resource expained"
date:   2017-08-26 12:56:04 +0530
categories: try with resource, AutoCloseable
---
Have you ever used more than one resources in try block without using those as `AutoCloseable` resources ? If your answer is yes, then it's worth reading this post.

This post is going to answer below questions
- Did you think you handled all the edge cases correctly ? 
- What if Exception is thrown while initializing resources ?
- What if Exception is thrown within try block ? 
- What if Exception is thrown while closing resources ?
- Are you closing all the resources correctly ? 
- Are you throwing appropriate Exception back to the caller ?
- Did you recorded all Exceptions ?

Let's just look at the below source code which will run the query and copy the result set to File. Here Connection and Writer instances are used as `AutoCloseable` resource. `Statement` can also be initialized as `AutoCloseable` instances but not included here for simplicity.

{% highlight java %}
import java.io.IOException;
import java.io.Writer;
import java.sql.Connection;
import java.sql.SQLException;

public abstract class ResultSetHelper {

	public void copyResultSetToFile(String query, String fileName) throws SQLException, IOException, ValidationException {

		try (Connection con = this.getConnection(); 
				Writer writer = this.getFileWriter(fileName)) {
			this.copyResultSetToStream(con, query, writer);
		}
	}

	public abstract void copyResultSetToStream(Connection con, String query, Writer writer) 
			throws SQLException, IOException, ValidationException ;

	public abstract Connection getConnection() throws SQLException ;

	public abstract Writer getFileWriter(String fileName) throws IOException ;

}

class ValidationException extends Exception {
	
}

{% endhighlight %}


<a name="test_scenarios"/>
With above source code, Exception can be generated from multiple methods. Below are possible code paths.

- **Happy path**. No exceptions and ResultSet copied successfully to file. This is what developers always expect.
- `SQLException` can be thrown while initializing exception in `getConnection()`.
- `IOException` can be thrown while initializing Writer inside `getFileWriter()`.
- `IOException`, `SQLException`, `ValidationException` or any `RuntimeException` can be thrown from `copyResultSetToStream()` method. If ValidationException is thrown, it should be propagated back to caller.
- `IOException` can be thrown while closing the stream at the end of try block
- `SQLException` is possible while closing Connection at the end of try block

If any Exception is thrown, all resources should be closed. This is the most trickiest part when more than one resources are not used as `AutoCloseable` resource.
Even most trickiest part of this Exception handling is throwing back the Appropriate exception.

eg. *if ValidationException or any RuntimeException is thrown, then same Exception should be raised back to the caller and any Exception raised during closing down the resource should be suppressed or in short Exception which was raised first, should be reported to caller*

 Try to realize Exception handling yourself and you will realize depth and complexity of it.

If you are using try with resource, compiler will handle this complexity and will generate necessary exception handling on behalf of you.

#### Suppressed Exception
With Java 5, when try with resource is introduced, `suppressedExceptions` was silently added to `Throwable`. With the help of Suppressed Exception, any Exception generated during closing down resource, are suppressed and recorded against main Exception.

Let's assume `ValidationException` is thrown from `copyResultSetToFile()` method and at the same time, while closing both Connection and Writer instances, `SQLException` and `IOException` exception thrown respectively. `ValidationException` will be thrown back to caller and SQLException and IOException's are recorded as suppressed Exception. Your Exception stack trace will look like something similar to below one. As mentioned on line 9 and 11, suppressed Exception will be logged.

{% highlight java linenos %}

ValidationException: ValidationException in copyResultSetToFile
	at ResultSetHelperTest.lambda$18(ResultSetHelperTest.java:215)
	-
	-
	-
	at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.runTests(RemoteTestRunner.java:678)
	at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.run(RemoteTestRunner.java:382)
	at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.main(RemoteTestRunner.java:192)
	Suppressed: java.io.IOException: Exception while closing writer
		... 31 more
	Suppressed: java.sql.SQLException: Exception while closing connection
		... 31 more

{% endhighlight %}


#### Difference between JDK and Eclipse Compiler
Even though java is *compile once and run everywhere*, different compiler generates different bytecodes.
Eclipse compiler generates [132 method bytes](#Eclipse_byte_codes) where as Oracle 1.8.0_131 JDK compiler generates [202 method bytes](#JDK_byte_codes). 

Here I tried to decode byte codes generated by Eclipse. Probably you will also do it similar way if you want to handle Exception yourself.
![Flow Diagram]({{ site.url }}/assets/try_with_resource_flow.png)

Refer to [ResultSetHelperTest.java](https://github.com/rohandhapodkar/suggested-java-best-practices/blob/master/src/test/java/test/java/trywithreousrce/ResultSetHelperTest.java) for Mockito based test cases for [scenarios described earlier](#test_scenarios)


Eclipse bytecodes:
<a name="Eclipse_byte_codes"/>
{% highlight java %}
  public void copyResultSetToFile(java.lang.String, java.lang.String) throws java.sql.SQLException, java.io.IOException;
    Code:
       0: aconst_null
       1: astore_3
       2: aconst_null
       3: astore        4
       5: aload_0
       6: invokevirtual #21                 // Method getConnection:()Ljava/sql/Connection;
       9: astore        5
      11: aload_0
      12: aload_2
      13: invokevirtual #25                 // Method getFileWriter:(Ljava/lang/String;)Ljava/io/Writer;
      16: astore        6
      18: aload_0
      19: aload         5
      21: aload_1
      22: aload         6
      24: invokevirtual #29                 // Method copyResultSetToStream:(Ljava/sql/Connection;Ljava/lang/String;Ljava/io/Writer;)V
      27: aload         6
      29: ifnull        53
      32: aload         6
      34: invokevirtual #33                 // Method java/io/Writer.close:()V
      37: goto          53
      40: astore_3
      41: aload         6
      43: ifnull        51
      46: aload         6
      48: invokevirtual #33                 // Method java/io/Writer.close:()V
      51: aload_3
      52: athrow
      53: aload         5
      55: ifnull        132
      58: aload         5
      60: invokeinterface #38,  1           // InterfaceMethod java/sql/Connection.close:()V
      65: goto          132
      68: astore        4
      70: aload_3
      71: ifnonnull     80
      74: aload         4
      76: astore_3
      77: goto          92
      80: aload_3
      81: aload         4
      83: if_acmpeq     92
      86: aload_3
      87: aload         4
      89: invokevirtual #41                 // Method java/lang/Throwable.addSuppressed:(Ljava/lang/Throwable;)V
      92: aload         5
      94: ifnull        104
      97: aload         5
      99: invokeinterface #38,  1           // InterfaceMethod java/sql/Connection.close:()V
     104: aload_3
     105: athrow
     106: astore        4
     108: aload_3
     109: ifnonnull     118
     112: aload         4
     114: astore_3
     115: goto          130
     118: aload_3
     119: aload         4
     121: if_acmpeq     130
     124: aload_3
     125: aload         4
     127: invokevirtual #41                 // Method java/lang/Throwable.addSuppressed:(Ljava/lang/Throwable;)V
     130: aload_3
     131: athrow
     132: return
    Exception table:
       from    to  target type
          18    27    40   any
          11    53    68   any
           5   106   106   any
    LineNumberTable:
      line 10: 0
      line 11: 11
      line 12: 18
      line 13: 27
      line 14: 132
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
          0     133     0  this   LResultSetCopier;
          0     133     1 query   Ljava/lang/String;
          0     133     2 fileName   Ljava/lang/String;
         11      93     5   con   Ljava/sql/Connection;
         18      33     6 writer   Ljava/io/Writer;

{% endhighlight %}

Oracle 1.8.0_131 JDK compiler bytecodes:
<a name="JDK_byte_codes"/>
{% highlight java %}
  public void copyResultSetToFile(java.lang.String, java.lang.String) throws java.sql.SQLException, java.io.IOException, ValidationException;
    Code:
       0: aload_0
       1: invokevirtual #2                  // Method getConnection:()Ljava/sql/Connection;
       4: astore_3
       5: aconst_null
       6: astore        4
       8: aload_0
       9: aload_2
      10: invokevirtual #3                  // Method getFileWriter:(Ljava/lang/String;)Ljava/io/Writer;
      13: astore        5
      15: aconst_null
      16: astore        6
      18: aload_0
      19: aload_3
      20: aload_1
      21: aload         5
      23: invokevirtual #4                  // Method copyResultSetToStream:(Ljava/sql/Connection;Ljava/lang/String;Ljava/io/Writer;)V
      26: aload         5
      28: ifnull        113
      31: aload         6
      33: ifnull        56
      36: aload         5
      38: invokevirtual #5                  // Method java/io/Writer.close:()V
      41: goto          113
      44: astore        7
      46: aload         6
      48: aload         7
      50: invokevirtual #7                  // Method java/lang/Throwable.addSuppressed:(Ljava/lang/Throwable;)V
      53: goto          113
      56: aload         5
      58: invokevirtual #5                  // Method java/io/Writer.close:()V
      61: goto          113
      64: astore        7
      66: aload         7
      68: astore        6
      70: aload         7
      72: athrow
      73: astore        8
      75: aload         5
      77: ifnull        110
      80: aload         6
      82: ifnull        105
      85: aload         5
      87: invokevirtual #5                  // Method java/io/Writer.close:()V
      90: goto          110
      93: astore        9
      95: aload         6
      97: aload         9
      99: invokevirtual #7                  // Method java/lang/Throwable.addSuppressed:(Ljava/lang/Throwable;)V
     102: goto          110
     105: aload         5
     107: invokevirtual #5                  // Method java/io/Writer.close:()V
     110: aload         8
     112: athrow
     113: aload_3
     114: ifnull        202
     117: aload         4
     119: ifnull        143
     122: aload_3
     123: invokeinterface #8,  1            // InterfaceMethod java/sql/Connection.close:()V
     128: goto          202
     131: astore        5
     133: aload         4
     135: aload         5
     137: invokevirtual #7                  // Method java/lang/Throwable.addSuppressed:(Ljava/lang/Throwable;)V
     140: goto          202
     143: aload_3
     144: invokeinterface #8,  1            // InterfaceMethod java/sql/Connection.close:()V
     149: goto          202
     152: astore        5
     154: aload         5
     156: astore        4
     158: aload         5
     160: athrow
     161: astore        10
     163: aload_3
     164: ifnull        199
     167: aload         4
     169: ifnull        193
     172: aload_3
     173: invokeinterface #8,  1            // InterfaceMethod java/sql/Connection.close:()V
     178: goto          199
     181: astore        11
     183: aload         4
     185: aload         11
     187: invokevirtual #7                  // Method java/lang/Throwable.addSuppressed:(Ljava/lang/Throwable;)V
     190: goto          199
     193: aload_3
     194: invokeinterface #8,  1            // InterfaceMethod java/sql/Connection.close:()V
     199: aload         10
     201: athrow
     202: return
    Exception table:
       from    to  target type
          36    41    44   Class java/lang/Throwable
          18    26    64   Class java/lang/Throwable
          18    26    73   any
          85    90    93   Class java/lang/Throwable
          64    75    73   any
         122   128   131   Class java/lang/Throwable
           8   113   152   Class java/lang/Throwable
           8   113   161   any
         172   178   181   Class java/lang/Throwable
         152   163   161   any
    LineNumberTable:
      line 10: 0
      line 11: 8
      line 10: 15
      line 12: 18
      line 13: 26
      line 10: 64
      line 13: 73
      line 10: 152
      line 13: 161
      line 14: 202

{% endhighlight %}