Would you like to enable your JUnit runner to be able to handle parameterised tests? It shouldn't be too hard. Just follow my instructions:

  1. Instantiate `ParameterisedTestClassRunner` in your runner class like this:
```
public class MyRunner extends BlockJUnit4ClassRunner {
   private ParameterisedTestClassRunner parameterisedRunner = new ParameterisedTestClassRunner();
   ...
```
  1. Since we execute each parameterset of a parametrised method as a separate test, you need to list these methods separately. Like this:
```
    @Override
    protected List<FrameworkMethod> computeTestMethods() {
        return parameterisedRunner.computeTestMethods(getTestClass().getAnnotatedMethods(Test.class), false);
    }
```
  1. In your `runChild` method delegate to the parameterised runner:
```
    @Override
    protected void runChild(FrameworkMethod method, RunNotifier notifier) {
        if (method.getAnnotation(Ignore.class) != null) {
            Description ignoredMethod = parameterisedRunner.describeParameterisedMethod(method);
            for (Description child : ignoredMethod.getChildren()) {
                notifier.fireTestIgnored(child);
            }
            return;
        }

        if (!parameterisedRunner.runParameterisedTest(method, methodBlock(method), notifier))
            super.runChild(method, notifier);
    }
```
  1. Now we need to add your parameterised methods to the list of all test methods in `getDescription` method:
```
    @Override
    public Description getDescription() {
        Description description = Description.createSuiteDescription(getName(), getTestClass().getAnnotations());
        List<FrameworkMethod> resultMethods = parameterisedRunner.computeTestMethods(getTestClass().getAnnotatedMethods(Test.class), true);

        for (FrameworkMethod method : resultMethods)
            description.addChild(describeMethod(method));

        return description;
    }

    private Description describeMethod(FrameworkMethod method) {
        Description child = parameterisedRunner.describeParameterisedMethod(method);

        if (child == null)
            child = describeChild(method);

        return child;
    }   
```
  1. The last thing is overwriting the `methodInvoker` method to execute parameterised tests separately from other ones:
```
    @Override
    protected Statement methodInvoker(FrameworkMethod method, Object test) {
        Statement methodInvoker = parameterisedRunner.parameterisedMethodInvoker(method, test);
        if (methodInvoker == null)
            methodInvoker = super.methodInvoker(method, test);

        return methodInvoker;
    }
```

With possibly some little additions your runner should be now able to handle parameterised tests. For examples see source code of `JUnitParamsRunner` or `TumblerRunner` class of http://tumbler-glass.googlecode.com