使用说明
---

## 引入TestableMock

首先在项目`pom.xml`文件中添加`testable-processor`依赖：

```xml
<dependency>
    <groupId>com.alibaba.testable</groupId>
    <artifactId>testable-processor</artifactId>
    <version>${testable.version}</version>
    <scope>provided</scope>
</dependency>
```

此时项目就获得了在单元测试中随意访问被测类私有字段和方法的能力（需配合注解使用，见下文详述）。

若要开启极速Mock功能，还需在`pom.xml`里加上`testable-maven-plugin`插件。

```xml
<plugin>
    <groupId>com.alibaba.testable</groupId>
    <artifactId>testable-maven-plugin</artifactId>
    <version>${testable.version}</version>
    <executions>
        <execution>
            <id>prepare</id>
            <goals>
                <goal>prepare</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

> PS：其中`${testable.version}`需替换为具体版本号，当前最新版本为`0.2.2-SNAPSHOT`

## 使用TestableMock

`TestableMock`目前能为测试类提供两项增强能力：__直接访问被测类的私有成员__ 和 __极速Mock被测方法中的调用__

### 访问私有成员字段和方法

只需为测试类添加`@EnablePrivateAccess`注解，即可在测试用例中获得以下增强能力：

- 调用被测类的私有方法
- 读取被测类的私有成员
- 修改被测类的私有成员
- 修改被测类的常量成员（使用final修饰的成员）

访问和修改私有、常量成员时，IDE可能会提示语法有误，但编译器将能够正常运行测试。

若不希望看到IDE的语法错误提醒，或是在基于JVM的非Java语言项目里（譬如Kotlin语言），也可以借助`PrivateAccessor`工具类来实现私有成员的访问。

效果见`java-demo`和`kotlin-demo`示例项目中`DemoPrivateAccessService`类的测试用例。

### Mock被测类的任意方法调用

**1. <u>覆写任意类的方法调用</u>**

在测试类里定义一个有`@TestableMock`注解的普通方法，使它与需覆写的方法名称、参数、返回值类型完全一致，然后在其参数列表首位再增加一个类型为该方法原本所属对象类型的参数。

此时被测类中所有对该需覆写方法的调用，将在单元测试运行时，将自动被替换为对上述自定义Mock方法的调用。

**注意**：也可以将需覆写的方法名写到`@TestableMock`注解的`targetMethod`参数里，这样Mock方法自身就可以随意命名了（当遇到重名的待覆写方法时特别有用）。

例如，被测类中有一处`"anything".substring(1, 2)`调用，我们希望在运行测试的时候将它换成一个固定字符串，则只需在测试类定义如下方法：

```java
// 原方法签名为`String substring(int, int)`
// 调用此方法的对象`"anything"`类型为`String`
// 则Mock方法签名在其参数列表首位增加一个类型为`String`的参数（名字随意）
// 此参数可用于获得当时的实际调用者的值和上下文
@TestableMock
private String substring(String self, int i, int j) {
    return "sub_string";
}
```

完整代码示例见`java-demo`和`kotlin-demo`示例项目中的`should_able_to_mock_common_method()`测试用例。(由于Kotlin对String类型进行了魔改，故Kotlin示例中将被测方法在`BlackBox`类里加了一层封装)

**2. <u>覆写被测类自身的成员方法</u>**

有时候，在对某些方法进行测试时，希望将被测类自身的另外一些成员方法Mock掉。

操作方法与前一种情况相同，Mock方法的第一个参数类型需与被测类相同，即可实现对被测类自身（不论是公有或私有）成员方法的覆写。

例如，被测类中有一个签名为`String innerFunc(String)`的私有方法，我们希望在测试的时候将它替换掉，则只需在测试类定义如下方法：

```java
// 被测类型是`DemoMockService`
// 因此在定义Mock方法时，在目标方法参数首位加一个类型为`DemoMockService`的参数（名字随意）
@TestableMock
private String innerFunc(DemoMockService self, String text) {
    return "mock_" + text;
}
```

完整代码示例见`java-demo`和`kotlin-demo`示例项目中的`should_able_to_mock_member_method()`测试用例。

**3. <u>覆写任意类的静态方法</u>**

对于静态方法的Mock与普通方法相同。但需要注意的是，对于静态方法，传入Mock方法的第一个参数实际值始终是`null`。

例如，在被测类中调用了`BlackBox`类型中的静态方法`secretBox()`，改方法签名为`BlackBox secretBox()`，则Mock方法如下：

```java
// 目标静态方法定义在`BlackBox`类型中
// 在定义Mock方法时，在目标方法参数首位加一个类型为`BlackBox`的参数（名字随意）
// 此参数仅用于标识目标类型，实际传入值将始终为`null`
@TestableMock
private BlackBox secretBox(BlackBox ignore) {
    return new BlackBox("not_secret_box");
}
```

完整代码示例见`java-demo`和`kotlin-demo`示例项目中的`should_able_to_mock_static_method()`测试用例。

**4. <u>覆写任意类的new操作</u>**

在测试类里定义一个有`@TestableMock`注解的普通方法，将注解的`targetMethod`参数写为"<init>"，然后使该方法与要被创建类型的构造函数参数、返回值类型完全一致，方法名称随意。

此时被测类中所有用`new`创建指定类的操作（并使用了与Mock方法参数一致的构造函数）将被替换为对该自定义方法的调用。

例如，在被测类中有一处`new BlackBox("something")`调用，希望在测试时将它换掉（通常是换成Mock对象，或换成使用测试参数创建的临时对象），则只需定义如下Mock方法：

```java
// 要覆写的构造函数签名为`BlackBox(String)`
// 无需在Mock方法参数列表增加额外参数，由于使用了`targetMethod`参数，Mock方法的名称随意起
// 此处的`CONSTRUCTOR`为`TestableTool`辅助类提供的常量，值为"<init>"
@TestableMock(targetMethod = CONSTRUCTOR)
private BlackBox createBlackBox(String text) {
    return new BlackBox("mock_" + text);
}
```

完整代码示例见`java-demo`和`kotlin-demo`示例项目中的`should_able_to_mock_new_object()`测试用例。

**5. <u>识别当前测试用例和调用来源</u>**

在Mock方法中可以通过`TestableTool.TEST_CASE`和`TestableTool.SOURCE_METHOD`来识别**当前运行的测试用例名称**和**进入该Mock方法前的被测类方法名称**，从而区分处理不同的调用场景。

完整代码示例见`java-demo`和`kotlin-demo`示例项目中的`should_able_to_get_source_method_name()`和`should_able_to_get_test_case_name()`测试用例。

**6. <u>验证Mock方法被调用的顺序和参数</u>**

在测试用例中可用通过`TestableTool.verify()`方法，配合`with()`、`withInOrder()`、`without()`、`withTimes()`等方法实现对Mock调用情况的验证。

详见`java-demo`和`kotlin-demo`示例项目中的相关测试用例。

