title: 理解java中的回调
date: 2015-03-12 22:08:19
categories: java
tags: 设计模式
---
在《java编程思想》一书中时不时的提到设计模式，书中也出现了回调，在学习的时候经常听到这个词，也有必要真正去弄明白什么叫**回调**？
**回调**用一句话解释就是，被调用方在被调用时也会调用对方。形象一点说"If you call me, i will call back"。举个生活中简单的例子：有一位老板很忙，他没有时间盯着员工干活，然后他告诉自己的雇员，干完当前这些事情后，告诉他干活的结果。
接下来的代码演示了这个过程：
```
/**
 * 回调接口,指明用什么方式跟老板汇报，所以老板应该实现这个接口
 */
public interface CallBackInterface {
	public void callback();
}
```
<!--more-->
接下来是老板类
```
 public class Boss implements CallBackInterface {

	private Employee employee;
	public Boss(Employee employee) {
		this.employee=employee;
	}
	//简单得实现callback
	@Override
	public void callback() {
		System.out.println("老板，做完了");
	}
	//发命令让员工做事
	public void doaction(){
		//将CallBackInterface接口告诉了employee
		employee.doSomeThing(this);
	}

	public static void main(String[] args) {
		Employee employee=new Employee();
		Boss boss=new Boss(employee);
		boss.doaction();
	}
}
```

员工类，
```
public class Employee {
	//简单模拟做事的方法，做完执行回调
	public void doSomeThing(CallBackInterface cb){
		System.out.println("干活。。");
		System.out.println("干完了！");
		cb.callback();
	}
}
```

测试：
```
        Employee employee=new Employee();
		Boss boss=new Boss(employee);
		boss.doaction();
```

关于回调的作用网上的这段话至少让我有了一定的认识：我们在用框架的时候，一般直接调用框架提供的API就可以了，但回调不同，当框架不能满足需求，我们想让框架来调用自己的类方法，怎么做呢？总不至于去修改框架吧。许多优秀的框架提几乎都供了相关的接口，我们只需要实现相关接口，即可完成了注册，然后在合适的时候让框架来调用我们自己的类，还记不记得我们在使用Struts时，当我们编写Action时，就需要继承Action类，然后实现execute()方法，在execute()方法中写咱们自己的业务逻辑代码，完成对用户请求的处理。由此可以猜测，框架和容器中会提供大量的回调接口，以满足个性化的定制。

如struts2里Action的一个代理类DefaultActionInvocation中的invoke方法
```
    public String invoke() throws Exception {
        String profileKey = "invoke: ";
        try {
            UtilTimerStack.push(profileKey);
            if (executed) {
                throw new IllegalStateException("Action has already executed");
            }
            //interceptors是包含所有拦截器的集合
            if (interceptors.hasNext()) {
                final InterceptorMapping interceptor
                = interceptors.next();
                String interceptorMsg = "interceptor: " + interceptor.getName();
                UtilTimerStack.push(interceptorMsg);
                try {
                		//调用拦截器的拦截方法
                        resultCode = interceptor.getInterceptor().
                        intercept(DefaultActionInvocation.this);
                            }
                finally {
                    UtilTimerStack.pop(interceptorMsg);
                }
            } else {
                resultCode = invokeActionOnly();
            }
            if (!executed) {
            //此处代码略去
            }
            return resultCode;
        }
        finally {
            UtilTimerStack.pop(profileKey);
        }
    }
```

下面是ModelDrivenInterceptor拦截器的intercept方法
```
    public String intercept(ActionInvocation invocation)
    throws Exception {
        Object action = invocation.getAction();
        if (action instanceof ModelDriven) {
            ModelDriven modelDriven = (ModelDriven) action;
            ValueStack stack = invocation.getStack();
            Object model = modelDriven.getModel();
            if (model !=  null) {
            	stack.push(model);
            }
            if (refreshModelBeforeResult) {
                invocation.addPreResultListener(
                new RefreshModelBeforeResult(modelDriven, model));
            }
        }
        //明显此处是回调
        return invocation.invoke();
    }
```

这两段代码连起来看就很明确，struts2处理一个请求的时候会调用DefaultActionInvocation的invoke方法，这个方法会执行一个拦截器的intercept方法，由于调用这个方法的时候传入了invocation自身，使得拦截器里可以回调invocation的invoke的方法，也正是反复的回调使得interceptors集合里的所有拦截器都得到执行。