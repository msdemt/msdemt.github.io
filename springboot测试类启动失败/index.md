# Springboot测试类启动失败


运行springboot 测试类时，启动失败了，错误信息如下

```log
java.lang.IllegalStateException: Unable to find a @SpringBootConfiguration, you need to use @ContextConfiguration or @SpringBootTest(classes=...) with your test

	at org.springframework.util.Assert.state(Assert.java:76)
	at org.springframework.boot.test.context.SpringBootTestContextBootstrapper.getOrFindConfigurationClasses(SpringBootTestContextBootstrapper.java:236)
	at org.springframework.boot.test.context.SpringBootTestContextBootstrapper.processMergedContextConfiguration(SpringBootTestContextBootstrapper.java:152)
	at org.springframework.test.context.support.AbstractTestContextBootstrapper.buildMergedContextConfiguration(AbstractTestContextBootstrapper.java:392)
	at org.springframework.test.context.support.AbstractTestContextBootstrapper.buildDefaultMergedContextConfiguration(AbstractTestContextBootstrapper.java:309)
	at org.springframework.test.context.support.AbstractTestContextBootstrapper.buildMergedContextConfiguration(AbstractTestContextBootstrapper.java:262)
	at org.springframework.test.context.support.AbstractTestContextBootstrapper.buildTestContext(AbstractTestContextBootstrapper.java:107)
	at org.springframework.boot.test.context.SpringBootTestContextBootstrapper.buildTestContext(SpringBootTestContextBootstrapper.java:102)
	at org.springframework.test.context.TestContextManager.<init>(TestContextManager.java:137)
	at org.springframework.test.context.TestContextManager.<init>(TestContextManager.java:122)
	at org.junit.jupiter.engine.execution.ExtensionValuesStore.lambda$getOrComputeIfAbsent$4(ExtensionValuesStore.java:86)
	at org.junit.jupiter.engine.execution.ExtensionValuesStore$MemoizingSupplier.get(ExtensionValuesStore.java:205)
	at org.junit.jupiter.engine.execution.ExtensionValuesStore$StoredValue.evaluate(ExtensionValuesStore.java:182)
	at org.junit.jupiter.engine.execution.ExtensionValuesStore$StoredValue.access$100(ExtensionValuesStore.java:171)
	at org.junit.jupiter.engine.execution.ExtensionValuesStore.getOrComputeIfAbsent(ExtensionValuesStore.java:89)
	at org.junit.jupiter.engine.execution.ExtensionValuesStore.getOrComputeIfAbsent(ExtensionValuesStore.java:93)
	at org.junit.jupiter.engine.execution.NamespaceAwareStore.getOrComputeIfAbsent(NamespaceAwareStore.java:61)
	at org.springframework.test.context.junit.jupiter.SpringExtension.getTestContextManager(SpringExtension.java:294)
	at org.springframework.test.context.junit.jupiter.SpringExtension.beforeAll(SpringExtension.java:113)
	at org.junit.jupiter.engine.descriptor.ClassBasedTestDescriptor.lambda$invokeBeforeAllCallbacks$8(ClassBasedTestDescriptor.java:368)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.jupiter.engine.descriptor.ClassBasedTestDescriptor.invokeBeforeAllCallbacks(ClassBasedTestDescriptor.java:368)
	at org.junit.jupiter.engine.descriptor.ClassBasedTestDescriptor.before(ClassBasedTestDescriptor.java:192)
	at org.junit.jupiter.engine.descriptor.ClassBasedTestDescriptor.before(ClassBasedTestDescriptor.java:78)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$5(NodeTestTask.java:136)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$7(NodeTestTask.java:129)
	at org.junit.platform.engine.support.hierarchical.Node.around(Node.java:137)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8(NodeTestTask.java:127)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively(NodeTestTask.java:126)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.execute(NodeTestTask.java:84)
	at java.util.ArrayList.forEach(ArrayList.java:1257)
	at org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.invokeAll(SameThreadHierarchicalTestExecutorService.java:38)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$5(NodeTestTask.java:143)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$7(NodeTestTask.java:129)
	at org.junit.platform.engine.support.hierarchical.Node.around(Node.java:137)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8(NodeTestTask.java:127)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively(NodeTestTask.java:126)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.execute(NodeTestTask.java:84)
	at org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.submit(SameThreadHierarchicalTestExecutorService.java:32)
	at org.junit.platform.engine.support.hierarchical.HierarchicalTestExecutor.execute(HierarchicalTestExecutor.java:57)
	at org.junit.platform.engine.support.hierarchical.HierarchicalTestEngine.execute(HierarchicalTestEngine.java:51)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:108)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:88)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.lambda$execute$0(EngineExecutionOrchestrator.java:54)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.withInterceptedStreams(EngineExecutionOrchestrator.java:67)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:52)
	at org.junit.platform.launcher.core.DefaultLauncher.execute(DefaultLauncher.java:96)
	at org.junit.platform.launcher.core.DefaultLauncher.execute(DefaultLauncher.java:75)
	at com.intellij.junit5.JUnit5IdeaTestRunner.startRunnerWithArgs(JUnit5IdeaTestRunner.java:71)
	at com.intellij.rt.junit.IdeaTestRunner$Repeater.startRunnerWithArgs(IdeaTestRunner.java:33)
	at com.intellij.rt.junit.JUnitStarter.prepareStreamsAndStart(JUnitStarter.java:220)
	at com.intellij.rt.junit.JUnitStarter.main(JUnitStarter.java:53)
	Suppressed: java.lang.IllegalStateException: Unable to find a @SpringBootConfiguration, you need to use @ContextConfiguration or @SpringBootTest(classes=...) with your test
		at org.springframework.util.Assert.state(Assert.java:76)
		at org.springframework.boot.test.context.SpringBootTestContextBootstrapper.getOrFindConfigurationClasses(SpringBootTestContextBootstrapper.java:236)
		at org.springframework.boot.test.context.SpringBootTestContextBootstrapper.processMergedContextConfiguration(SpringBootTestContextBootstrapper.java:152)
		at org.springframework.test.context.support.AbstractTestContextBootstrapper.buildMergedContextConfiguration(AbstractTestContextBootstrapper.java:392)
		at org.springframework.test.context.support.AbstractTestContextBootstrapper.buildDefaultMergedContextConfiguration(AbstractTestContextBootstrapper.java:309)
		at org.springframework.test.context.support.AbstractTestContextBootstrapper.buildMergedContextConfiguration(AbstractTestContextBootstrapper.java:262)
		at org.springframework.test.context.support.AbstractTestContextBootstrapper.buildTestContext(AbstractTestContextBootstrapper.java:107)
		at org.springframework.boot.test.context.SpringBootTestContextBootstrapper.buildTestContext(SpringBootTestContextBootstrapper.java:102)
		at org.springframework.test.context.TestContextManager.<init>(TestContextManager.java:137)
		at org.springframework.test.context.TestContextManager.<init>(TestContextManager.java:122)
		at org.junit.jupiter.engine.execution.ExtensionValuesStore.lambda$getOrComputeIfAbsent$4(ExtensionValuesStore.java:86)
		at org.junit.jupiter.engine.execution.ExtensionValuesStore$MemoizingSupplier.get(ExtensionValuesStore.java:205)
		at org.junit.jupiter.engine.execution.ExtensionValuesStore$StoredValue.evaluate(ExtensionValuesStore.java:182)
		at org.junit.jupiter.engine.execution.ExtensionValuesStore$StoredValue.access$100(ExtensionValuesStore.java:171)
		at org.junit.jupiter.engine.execution.ExtensionValuesStore.remove(ExtensionValuesStore.java:107)
		at org.junit.jupiter.engine.execution.NamespaceAwareStore.remove(NamespaceAwareStore.java:73)
		at org.springframework.test.context.junit.jupiter.SpringExtension.afterAll(SpringExtension.java:125)
		at org.junit.jupiter.engine.descriptor.ClassBasedTestDescriptor.lambda$invokeAfterAllCallbacks$14(ClassBasedTestDescriptor.java:434)
		at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
		at org.junit.jupiter.engine.descriptor.ClassBasedTestDescriptor.lambda$invokeAfterAllCallbacks$15(ClassBasedTestDescriptor.java:434)
		at java.util.ArrayList.forEach(ArrayList.java:1257)
		at org.junit.jupiter.engine.descriptor.ClassBasedTestDescriptor.invokeAfterAllCallbacks(ClassBasedTestDescriptor.java:434)
		at org.junit.jupiter.engine.descriptor.ClassBasedTestDescriptor.after(ClassBasedTestDescriptor.java:216)
		at org.junit.jupiter.engine.descriptor.ClassBasedTestDescriptor.after(ClassBasedTestDescriptor.java:78)
		at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$6(NodeTestTask.java:149)
		at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
		at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$7(NodeTestTask.java:149)
		... 29 more
```

测试类内容如下:

```java
@SpringBootTest
public class JpaTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    public void testSave(){
        User user = new User();
        user.setName("nike");
        user.setAge(28);
        user.setAddress("Shandong");

        userRepository.save(user);
    }
}
```

解决：在SpringBootTest注解上添加启动类

```java
@SpringBootTest(classes = SpringDataJpaApplication.class)
public class JpaTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    public void testSave(){
        User user = new User();
        user.setName("nike");
        user.setAge(28);
        user.setAddress("Shandong");

        userRepository.save(user);
    }
}
```

问题来了，为什么其他的测试类注解上不加启动类没问题呢？？？

仔细对比了下这个测试类，发现 测试类 和 启动类不在一个包里，所以需要加上启动类路径。  
这个例子中，启动类的路径为：org.msdemt.simple.spring_data_jpa_demo  
而测试类的路径为：org.msdemt.sample.spring_data_jpa_demo.test  

将测试类修改为org.msdemt.simple.spring_data_jpa_demo.test后，springboottest注解上不加启动类也可以运行成功了。

注意：springboot单元测试不必再引入junit包了，只用spring-boot-starter-test就可以。
