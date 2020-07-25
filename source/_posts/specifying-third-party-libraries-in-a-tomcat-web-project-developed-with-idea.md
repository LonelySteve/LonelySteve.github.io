---
title: 在使用 idea 开发的 tomcat Web 项目中指定第三方库
tags:
  - 原创
  - tomcat
  - idea
  - java
date: 2020-07-25 23:20:23
---


在学习 tomcat 开发 Web 项目的过程中，有时需要引入第三方库来简化某些操作，避免自己重复造轮子。和一般的 java 项目引入第三方库的步骤有所不同，如果仅仅只在项目中通过 idea 的 Project Structures 中的 Modules Dependencies 或者 Libraries 引入第三方库是不够的。因为还需要在部署的时候带上依赖才能正常运行。

<!-- more -->

## 环境

- IDEA 2019.3
- java 8
- tomcat 9

## 实例

### 目标

我们希望在当前 Web 工程中引入 Google 的 Gson 库，以支持 json 的序列化和反序列化。

### 解决方案

与通常的 java 项目的依赖添加步骤类似，首先要在 Project Structures 对话框的 Libraries 页面中添加 Gson 库，这里使用 Maven 或者本地 jar 包的方式都是可以的。（下面使用 Maven 方式进行演示）

![使用 Maven 安装 Gson 库](jiang_2020-07-24_07-46-32.png)

依赖安装完成后，就可以自定义一个 Servlet 进行测试：

```java
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import com.google.gson.Gson;

@WebServlet("/")
public class demo extends HttpServlet {
  @Override
  protected void doGet(HttpServletRequest req, HttpServletResponse resp)
      throws ServletException, IOException {
    Person person = new Person("张三", 22, true, "445155199812125318", "13832535364");
    Gson gson = new Gson();
    resp.setContentType("text/html; charset=UTF-8");
    resp.getWriter().write(gson.toJson(person));
  }
}

class Person {
  public String name;
  public int age;
  public boolean sex;
  public String id;
  public String phone;

  public Person(String name, int age, boolean sex, String id, String phone) {
    this.name = name;
    this.age = age;
    this.sex = sex;
    this.id = id;
    this.phone = phone;
  }
}

```

比较坑爹的是，idea 不会有任何明显的错误提示，编译运行都不会产生报错或者警告。但是当访问相应页面的时候就会得到下面的异常页面：

![异常页面](jiang_2020-07-24_08-07-07.png)

被抛出的异常是 `NoClassDefFoundError`，这意味着编译时 Gson 库是存在的，但是在运行时却没能找到。网上对类似问题的解决方法都是简单粗暴地将所依赖的 JAR 包放在 tomcat 的库目录下，且不说这样做能不能成功，这种做法一来不优美，二来不具有可移植性，属于没有办法的办法。

那么真的就没有其它办法了吗？答案是否定的。事实上 Java Web 项目和普通的 Java 项目不同，它拥有一个部署的过程，这一过程会复制必要的的字节码文件和静态文件到 out 目录。如果引入了第三方库，但只进行了通常的依赖安装步骤，那么 idea 在部署时是不会输出依赖的。对于这一问题 idea 也并不是完全没有提示，在 Project Structure 中 Problems 或 Artifacts 页面，打开这两个页面之一就可以看到相关提示和快捷修复入口：

![错误提示2](jiang_2020-07-24_09-37-29.png)

![错误提示](jiang_2020-07-24_08-07-49.png)

▲ *值得吐槽的是，这两个页面的提示实在是过于不起眼，很容易被忽略掉。*

本质上，如果通过上面的方式修复了问题，那么当前模块下的 .idea/artifacts 目录中有一个 artifact 的配置文件会发生改变：

```xml
<component name="ArtifactManager">
  <artifact type="exploded-war" name="demo:war exploded">
    <output-path>$PROJECT_DIR$/out/artifacts/demo_war_exploded</output-path>
    <root id="root">
      <element id="javaee-facet-resources" facet="demo/web/Web" />
      <element id="directory" name="WEB-INF">
        <element id="directory" name="classes">
          <element id="module-output" name="demo" />
        </element>
        <!-- 下面这个 element 元素就是通过上面的快捷修复方式引入进来的 -->
        <element id="directory" name="lib">
          <element id="library" level="project" name="com.google.code.gson:gson:2.8.6" />
        </element>
      </element>
    </root>
  </artifact>
</component>
```

## 依赖作用域 | dependency scope

其实到这里，问题其实应该已经解决了，但是不知道读者是否注意到没有，在修复的过程中，除了第一选项，还有两个选项可供使用，他们的区别在于依赖作用域的不同。

![四种依赖作用域](jiang_2020-07-24_09-49-16.png)

在 idea 中，对于依赖项有四种作用域：

- Compile（编译）：构建、测试和运行时所必需的（默认范围）
- Test（测试）：编译和运行单元测试时所需
- Runtime（运行时）：包含在源和测试源的类路径之中，但仅在运行阶段
- Provided（提供）：用于构建和测试项目
  
| 范围              | 当编译源代码时 | 当运行源代码时 | 当编译测试时 | 当运行测试时 |
| :---------------- | :------------: | :------------: | :----------: | :----------: |
| Compile（编译）   |       +        |       +        |      +       |      +       |
| Test（测试）      |       -        |       -        |      +       |      +       |
| Runtime（运行时） |       -        |       +        |      -       |      +       |
| Provided（提供）  |       +        |       -        |      +       |      +       |

根据上面的表格，如果我们在项目中对依赖项使用 Provided ，运行时是不会使用这个依赖的，但在我测试之后发现对于上面那个例子来说这样做并不会有问题，可能 artifact 的配置文件的优先级更高。

## 参考资料

- https://stackoverflow.com/questions/34413/why-am-i-getting-a-noclassdeffounderror-in-java
- https://www.jetbrains.com/help/idea/working-with-module-dependencies.html#analyze-dependencies
- https://zhuanlan.zhihu.com/p/79932848