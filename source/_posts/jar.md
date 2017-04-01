title: jar包中内容的读取
date: 2017-04-01 13:42:23
categories: java
tags: tiny4j
---
最近几天有空在折腾[tiny4j](https://github.com/twogoods/tiny4j)里一直想试着实现的类似springboot的可执行jar包。在`component-scan`包扫描这一步碰上点小问题，记录一下。
项目的`Bean`都是基于`component-scan`扫描得到的，要实现这个功能首先根据配置的包名查找的资源路径，这一点可以通过类加载器来得到,它可以在运行时动态的获取加载的类的信息`class.getClass().getResource(packageName);`这个函数返回一个迭代URL。<!--more-->
这时如果我们通过`java -jar XXX.jar`来运行，返回的URL会是类似这样的`/Users/xxx/IdeaProjects/web/target/web.jar!/classes/rest.jar/com/tg`。jar包是一个文件，文件内部的一些信息没办法直接通过文件API得到，我们可以通过`JarFile`来得到

```
JarFile localJarFile = new JarFile(new File("/Users/xxx/IdeaProjects/abc/web/target/web.jar"));
Enumeration<JarEntry> entries = localJarFile.entries();
        while (entries.hasMoreElements()) {
            JarEntry jarEntry = entries.nextElement();
            System.out.println(jarEntry.getName());
        }
```
如果我们要扫描的类在jar包里的jar包里该怎么办呢？如这个URL `/Users/xxx/IdeaProjects/web/target/web.jar!/BOOT-INF/lib/rest.jar/com/tg`,这个形式在可执行Jar里是很正常的，lib目录下是一些依赖的jar。然而上述方法已经得不到这个路径中`rest.jar`里的内容了。可能这个问题不是很常见，网上找了一圈也没找到解决方法，遂跑去看了一下Spring的实现。在Spring的`PathMatchingResourcePatternResolver`看到了关键的一个方法

```
 protected Set<Resource> doFindPathMatchingJarResources(Resource rootDirResource, String subPattern) throws IOException {
        URLConnection con = rootDirResource.getURL().openConnection();
        boolean newJarFile = false;
        JarFile jarFile;
        String jarFileUrl;
        String rootEntryPath;
        if(con instanceof JarURLConnection) {
            JarURLConnection result = (JarURLConnection)con;
            ResourceUtils.useCachesIfNecessary(result);
            jarFile = result.getJarFile();
            jarFileUrl = result.getJarFileURL().toExternalForm();
            JarEntry entries = result.getJarEntry();
            rootEntryPath = entries != null?entries.getName():"";
        } else {
            String result1 = rootDirResource.getURL().getFile();
            int entries1 = result1.indexOf("!/");
            if(entries1 != -1) {
                jarFileUrl = result1.substring(0, entries1);
                rootEntryPath = result1.substring(entries1 + "!/".length());
                jarFile = this.getJarFile(jarFileUrl);
            } else {
                jarFile = new JarFile(result1);
                jarFileUrl = result1;
                rootEntryPath = "";
            }
            newJarFile = true;
        }

        LinkedHashSet entries3;
        try {
            if(logger.isDebugEnabled()) {
                logger.debug("Looking for matching resources in jar file [" + jarFileUrl + "]");
            }
            if(!"".equals(rootEntryPath) && !rootEntryPath.endsWith("/")) {
                rootEntryPath = rootEntryPath + "/";
            }
            LinkedHashSet result2 = new LinkedHashSet(8);
            Enumeration entries2 = jarFile.entries();
            while(entries2.hasMoreElements()) {
                JarEntry entry = (JarEntry)entries2.nextElement();
                String entryPath = entry.getName();
                if(entryPath.startsWith(rootEntryPath)) {
                    String relativePath = entryPath.substring(rootEntryPath.length());
                    if(this.getPathMatcher().match(subPattern, relativePath)) {
                        result2.add(rootDirResource.createRelative(relativePath));
                    }
                }
            }
            entries3 = result2;
        } finally {
            if(newJarFile) {
                jarFile.close();
            }

        }
        return entries3;
    }

```
方法里的一些对jar的操作其实跟最上的那段代码差不多，在简单调试后我发现Spring的这段代码也是无法实现读取第二层jar包的。这是Spring里的实现，并非SpringBoot`@ComponentScan`注解的实现，我们传统的Spring项目还是通过war包的方式部署，虽然war包里也有lib目录，看似确实是有两层jar，但在把war部署到servlet容器的时候，其实容器帮我们做了一次解压操作，于是这个场景下就只有一层jar了，或许这就能解释为什么spring上述实现依旧可以正常运行。
当然，最终我还是在网上找到了读取多层jar的方法:

```
URL url = new URL("jar", null, 0, "file:/Users/xxx/IdeaProjects/web/target/web.jar!/BOOT-INF/lib/rest.jar");
        URLConnection con = url.openConnection();
        if (con instanceof JarURLConnection) {
            JarURLConnection result = (JarURLConnection) con;
            JarInputStream jarInputStream = new JarInputStream(result.getInputStream());
            JarEntry entry;
            while ((entry = jarInputStream.getNextJarEntry()) != null) {
                System.out.println(entry.getName());
            }
        }
```