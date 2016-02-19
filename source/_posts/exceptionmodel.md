title: Java中IO的异常处理
date: 2015-05-08 21:18:39
categories: java
tags: 异常处理
---
 1. 平时都不太注意异常的处理，随意的try-catch一下，就像这样
 ```
 		try {
			return 10 / i;
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
		}

 ```
 真正在项目中如果就这样异常的信息只打印在了控制台，log上是没有的，一般都会这样
  <!--more-->
 ```
 		try {
			return 10 / i;
		} catch (Exception ex) {
			throw new Exception("exception");
		} finally {
		}
 ```
 try-catch后重新抛出一个异常（当然最好自己定义一个异常类，抛出自己定义的异常），一层一层抛，最后可以统一处理异常。
 2. io中的异常处理：
在涉及java的io时会遇到各种异常的try-catch典型的写法是这样
```
		InputStream input = null;
		OutputStream output = null;
		byte[] buffer=new byte[1024];
		try {
			input = new FileInputStream("XXX");
			output = new FileOutputStream("YYY");
			while(input.read(buffer) != -1) {
				output.write(buffer);
			}
		} catch (IOException e) {
		} finally {
			if (input != null) {
				try {
					input.close();
				} catch (IOException e) {
				}
			}
			if (output != null) {
				try {
					output.close();
				} catch (IOException e) {
				}
			}
		}
```
由于要处理各种异常，代码变的特别丑陋，这里异常处理模板就派上用场了
模板1：抽象类
```
public abstract class InputStreamProcessingTemplate {
	public void process(String fileName) {
		InputStream input = null;
		try {
			input = new FileInputStream(fileName);
			doProcess(input);
		} catch (IOException e) {
		} finally {
			if (input != null) {
				try {
					input.close();
				} catch (IOException e) {
				}
			}
		}
	}

	public abstract void doProcess(InputStream input)
	throws IOException;
}
```
接下来如何使用
```
new InputStreamProcessingTemplate(){
        public void doProcess(InputStream input)
        throws IOException{
            int inChar = input.read();
            while(inChar !- -1){
                //do something with the chars...
            }
        }
    }.process("someFile.txt");
```
模板2：接口方式
先统一声明一个处理具体业务代码的接口,流怎么使用都在这里实现
```
public interface InputStreamProcessor {
	public void process(InputStream input) throws IOException;
}
```
处理模板
```
public class InputStreamProcessingTemplate2 {
	public static void process(String fileName,
	InputStreamProcessor processor){
		InputStream input = null;
		try {
			input = new FileInputStream(fileName);
			processor.process(input);
		} catch (IOException e) {
		} finally {
			if (input != null) {
				try {
					input.close();
				} catch (IOException e) {
				}
			}
		}
	}
}
```
具体使用传入匿名内部类
```
InputStreamProcessingTemplate2.process("e://1.txt",
            new InputStreamProcessor() {
			@Override
			public void process(InputStream input)
			throws IOException {
				int data = input.read();
				while (data != -1) {
					System.out.println((char) data);
					data = input.read();
				}
			}
		});
```
这样一来我们使得代码就变得比较整洁,当然java这样一来我们使得代码就变得比较整洁，当然从Java7开始，一种新的被称作“try-with-resource”的异常处理机制被引入进来，也可以像这样使用，它会确保流在使用完被正确关闭。
```
try(FileInputStream input = new FileInputStream("file.txt")) {
	        int data = input.read();
	        while(data != -1){
	            System.out.print((char) data);
	            data = input.read();
	        }
	    } catch (IOException e) {
			e.printStackTrace();
		}
```