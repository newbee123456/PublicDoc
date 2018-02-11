### Gradle 版本升级
> 相信最近在sdp上下载工程文件的同学会发现，工厂直接把 gradle 编译插件升级到3.0.0了；

>      classpath 'com.android.tools.build:gradle:3.0.0'

> 然后你*import* 工程文件并构建的时候会悲催的发现一堆的错误：
1. 要求Android studio 要升级到3.0.0
2. 要求我们本地工程要手动更新Glade文件中的 `gradle-wrapper.properties` 中的网址
        `distributionUrl=https\\\://services.gradle.org/distributions/gradle-4.1-all.zip`
3. ...  （反正就是一堆问题，具体问题跟原工程的依赖有关）

> 那么为什么应用工厂会直接把gradle升级到3.0.0呢？这肯定是跟他的功能特性还有优势有关了。
>升级 gradle build tools 至 3.0.0会加快整个工程构建的速度，相信使用过的同学应该会有所体会，尤其是在本地重新打包的时候。

这也使得gradle 插件的版本 API带来了较大的变化。本文参考网上相关文章结合实际开发，就gradle API 比较大的变化为大家做一些分享。

### 使用新的依赖配置
> Gradle 3.4 引入了新的 Java 库插件配置,Android 插件 3.0.0 正在迁移到这些新依赖项配置。 要迁移您的项目，只需更新您的依赖项以使用新配置。
>先看下新旧配置列表：

>  其中 `模块` 指的是 调用依赖的 模块

| 新配置        | 已弃用配置  |  行为  |
| ------------- |:------------:| :-----|
| implementation      | compile | 依赖项在编译时对`模块`可用，并且仅在运行时对`模块`的消费者可用。对于大型多项目构建，使用 implementation 而不是 api/compile 可以显著缩短构建时间，因为它可以减少构建系统需要重新编译的项目量。大多数应用和测试模块都应使用此配置。 |
| api      | compile      |   依赖项在编译时对`模块`可用，并且在编译时和运行时还对`模块`的消费者可用。 此配置的行为类似于 compile（现在已弃用），一般情况下，您应当仅在库模块中使用它。 应用模块应使用 implementation，除非您想要将其 API 公开给单独的测试模块。 |
| compileOnly | provided    |    依赖项仅在编译时对`模块`可用，并且在编译或运行时对其消费者不可用。 此配置的行为类似于 provided（现在已弃用）。 |
| runtimeOnly | apk    |    依赖项仅在运行时对`模块`及其消费者可用。 此配置的行为类似于 apk（现在已弃用）。 |

为了让各位更好的理解新的配置，接下来，举例说明下 `implementation` 和 `api` 的区别。
假设 现在 有名为A 、B 两个的module 类库,其中 A1 类、B2类分别在A、B

       //A  module
       public class A1 {
           public static String getAString(){
               return "hello";
           }
       }

       //B  module
       public class B2 {
           public String getBString(){
               return A1.getAString();
           }
       }
现 假设 B module中的 build.gradle 依赖 A module

    //B module
    dependencies {
        api project(':A')
    }
然后在主module中的 build.gradle 依赖 B module

    dependencies {
        api project(':B')
    }
那么在主module中，使用`api`依赖配置， A、B module都可以访问到：

    B2 b = new B2();//允许被创建并调用
    b.getBString();
    A1.getAString();//可以被调用（本意是不允许的）
 > 这时候，一些不想要被使用的实现类或方法就泄露了

如果我们在 B module中 的 build.gradle 使用`implementation` 依赖A module

         //B module
         dependencies {
             implementation project(':A')
         }

然后在主module中的 build.gradle 使用`implementation` 依赖 B module

         dependencies {
             implementation project(':B')
         }

 那么在主module中，使用`implementation`依赖配置， A、B module的调用情况：

         B2 b = new B2();//允许被创建并调用
         b.getBString();
         A1.getAString();//报错，找不到该方法

>  如果在 B module中使用的是 `api` 依赖 A module，那么不管主module使用的是`api` 或 `implementation`依赖配置,主module中都可以调用`A1.getAString()`;

>使用这种封箱策略，当 B module `implementation`  A module的接口代码发生改动的时候，Gradle 只会重新编译 B module，不会触发重新编译整个应用，因为无法访问A module。
当有大量的嵌套依赖的时候，这个机制就会明显的提升构建速度。这就是为什么本地重新打包的时候速度杠杠的~

### 使用注解处理器依赖配置
在之前版本的 Android Plugin for Gradle 中，编译类路径中的依赖项将自动添加到处理器类路径中。 也就是说，您可以向编译类路径中添加一个注解处理器，它将按预期工作。 不过，由于向处理器添加了大量不需要的依赖项，这会对性能造成显著影响。

使用新插件时，必须使用 annotationProcessor 依赖项配置将注解处理器添加到处理器类路径中，如下所示：

    dependencies {
        ...
         annotationProcessor 'com.google.dagger:dagger-compiler:<version-number>'
        }
如果 JAR 文件包含以下文件，插件会将依赖项假设为注解处理器：META-INF/services/javax.annotation.processing.Processor。如果插件在编译类路径中检测到注解处理器，您的构建将失败，并且您将看到一条错误消息，其中列出了编译类路径中的每一个注解处理器。

![annotationProcessor](/images/annotationProcessor.png)

 要修复错误，只需更改这些依赖项的配置以使用 annotationProcessor。 如果依赖项包含也需要位于编译类路径中的组件，请再次声明该依赖项并使用 compile 依赖项配置。

    dependencies {
        ...
        compile 'com.jakewharton:butterknife:6.1.0'
        annotationProcessor 'com.jakewharton:butterknife-compiler:6.1.0'
        }


### 停用错误检查
如果编译类路径中的依赖项包含您不需要的注解处理器，您可以将以下代码添加到 build.gradle 文件中，停用错误检查。 请记住，您添加到编译类路径中的注解处理器仍不会添加到处理器类路径中。

    android {
        ...
        defaultConfig {
            ...
            javaCompileOptions {
                annotationProcessorOptions {
                    includeCompileClasspath false
                }
            }
        }
    }

如果您在迁移到新依赖项解析策略时遇到问题

![includeCompileClasspath](/images/includeCompileClasspath.png)

可以设置 `includeCompileClasspath true`，将行为恢复为 Android 插件 2.3 的行为。 不过，不建议将行为恢复为版本 2.3，未来更新中将移除恢复选项。

### 总结
* 当你切换到新的 Android gradle plugin 3.x.x，你应用使用 `implementation` 替换所有的 `compile` 依赖配置。然后尝试编译和测试你的应用。
如果没问题那样最好，如果有问题说明你的依赖或使用的代码现在是私有的或不可访问
* 对于所有需要公开的 API 你应该使用 `api` 依赖配置，测试依赖或不让最终用户使用的依赖使用 `implementation` 依赖配置。
* 工程构建将失败时，如果报`annotationProcessor`相关错误，根据错误信息找到相对应的依赖，使用`annotationProcessor` 对其进行声明。

### 参考文献
[安卓工程依赖新方式：Implementation vs API dependency](https://juejin.im/entry/59476897da2f60006786029f)

[Gradle 依赖配置 api VS implementation](https://www.jianshu.com/p/dd932f951137)

[Gradle官方迁移指南](https://developer.android.com/studio/build/gradle-plugin-3-0-0-migration.html#new_configurations)

