---
title: 【Java进阶】利用APT优雅的实现统一日志格式
summary:
    enabled: '1'
    format: short
---

#### 统一日志格式的几种方式

无论是搭建日志平台还是进行大数据分析，统一日志格式都是一个重要的前提条件。假设要统一成下面的日志格式，

> 日志格式：[{系统}|{模块}]{描述}[param1=value1$param2=value2]，例如：[API|Weixin]Weixin send message failed. [senderId=1234$receiverId=5678]

常见的方法有：

- 方法1：每次记录日志时，根据上下文在原始的消息内容前后分别加上合适的[{系统}|{模块}]前缀和参数后缀。
- 方法2：自定义日志类，将{系统}和{模块}作为构造函数的参数传入，并且在所提供的日志接口中自动格式化传入的参数数组。
- 方法3：自定义注解类声明所属的{系统}和{模块}，然后通过AOP的方式，统一在日志中插入[{系统}|{模块}]前缀。
- 方法4：在方法2的基础上，自定义注解类声明所属的{系统}和{模块}，然后通过APT自动生成自定义类型的log成员变量。

方法1依赖于人工来保证统一的日志格式，方法3虽然简化了方法调用，但对性能有一定的影响。方法2是最常见的手段，但每个类都要显示声明log成员变量，略显冗余。方法4兼具方法2和方法3的优点，同时又避免了两者的不足，是一种优雅的实现方式，也是[lombok](https://github.com/rzwitserloot/lombok)所采用的方式。

下面就针对方法4，结合示例代码介绍一下相关技术。

#### APT: 编译期自动生成log成员变量

[APT](http://docs.oracle.com/javase/6/docs/technotes/guides/apt/)的全称是Annotation Processing Tool，诞生于Java 6版本，主要用于在编译期根据不同的注解类生成或者修改代码。APT运行于独立的JVM进程中（编译之前），并且在一次编译过程中可能会被多次调用。

首先，声明一个包含{系统}和{模块}定义的日志注解类。注意@Retention应设置为RetentionPolicy.SOURCE，表示编译后擦除该注解信息。

```java
/**
 * 用于自动生成log成员变量.仅适用于class或enum,不适用于接口.
 */
@Retention(RetentionPolicy.SOURCE)
@Target(ElementType.TYPE)
public @interface Slf4j {

    /**
     * 系统名称.如果为空则取"-Dvlogging.system"系统属性,如果系统属性也为空,则取"Unknown".
     */
    String system() default "";

    /**
     * 模块名称.如果为空则取"-Dvlogging.module"系统属性,如果系统属性也为空,则取"Unknown".
     */
    String module() default "";
}
```

然后，声明一个注解处理类，继承Java默认提供的AbstractProcessor类，其中：

- messager: 用于记录处理日志
- trees: 用于解析Java AST树
- maker: 用于生成Java AST节点
- names: 用于生成Java AST节点名称

```java
public class Slf4jProcessor extends AbstractProcessor {

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        messager = processingEnv.getMessager();
        trees = Trees.instance(processingEnv);
        Context context = ((JavacProcessingEnvironment) processingEnv).getContext();
        maker = TreeMaker.instance(context);
        names = Names.instance(context);
    }

    ...
}
```

在process方法中调用Java Compiler API根据注解信息动态生成log日志成员变量：<br>
`private static final Logger log = LoggerFactory.getLogger(LoggerFactory.Type.SLF4J, annotatedClass.class, system, module);`

```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    // 1 检查类型
    roundEnv.getElementsAnnotatedWith(Slf4j.class).stream().forEach(elm -> {
        if (elm.getKind() != ElementKind.CLASS && elm.getKind() != ElementKind.ENUM) {
            messager.printMessage(Diagnostic.Kind.ERROR, "Only classes or enums can be annotated with " + Slf4j.class.getSimpleName());
            return;
        }

        // 2 检查log成员变量是否已存在
        TypeElement typeElm = (TypeElement) elm;
        if (typeElm.getEnclosedElements().stream()
                .filter(e -> e.getKind() == ElementKind.FIELD && Logger.FIELD_NAME.equals(e.getSimpleName())).count() > 0) {
            messager.printMessage(Diagnostic.Kind.WARNING, MessageFormat.format("A member field named {0} already exists in the annotated class", Logger.FIELD_NAME));
            return;
        }

        // 3 注入log成员变量
        CompilationUnitTree cuTree = trees.getPath(typeElm).getCompilationUnit();
        if (cuTree instanceof JCTree.JCCompilationUnit) {
            JCTree.JCCompilationUnit cu = (JCTree.JCCompilationUnit) cuTree;
            // only process on files which have been compiled from source
            if (cu.sourcefile.getKind() == JavaFileObject.Kind.SOURCE) {
                _findType(cu, typeElm.getQualifiedName().toString()).ifPresent(type -> {
                    Slf4j slf4j = typeElm.getAnnotation(Slf4j.class);
                    String system = slf4j.system();
                    String module = slf4j.module();

                    // 生成private static final Logger log = LoggerFactory.getLogger(LoggerFactory.Type.SLF4J, <annotatedClass>, <system>, <module>);
                    JCTree.JCExpression loggerType = _toExpression(Logger.class.getCanonicalName());
                    JCTree.JCExpression getLoggerMethod = _toExpression(LoggerFactory.class.getCanonicalName() + ".getLogger");
                    JCTree.JCExpression typeArg = _toExpression(LoggerFactory.Type.class.getCanonicalName() + "." + LoggerFactory.Type.SLF4J.name());
                    JCTree.JCExpression nameArg = _toExpression(typeElm.getQualifiedName() + ".class");
                    JCTree.JCExpression systemArg = maker.Literal(system);
                    JCTree.JCExpression moduleArg = maker.Literal(module);
                    JCTree.JCMethodInvocation getLoggerCall = maker.Apply(List.nil(), getLoggerMethod, List.of(typeArg, nameArg, systemArg, moduleArg));
                    JCTree.JCVariableDecl logField = maker.VarDef(
                            maker.Modifiers(Flags.PRIVATE | Flags.STATIC | Flags.FINAL),
                            names.fromString(Logger.FIELD_NAME), loggerType, getLoggerCall);

                    _insertField(type, logField);
                });
            }
        }
    });

    return true;
}
```

##### 集成示例

```java
@Slf4j(system = "Vlogging", module = "Integration")
public class VloggingAnnotated {

    public static void main(String[] args) {
        HashMap<String, String> params = new HashMap<>();
        params.put("foo", "xyz");
        log.info(VloggingAnnotated.class.getCanonicalName(), params);
    }
}
```

由此可见，使用方法4，业务类只要加上自定义注解，然后正常调用日志API，就可以以统一的日志格式记录日志。

##### 输出示例

```
2016-07-10 17:26:45 +0800 [INFO] from VloggingAnnotated in main - [Vlogging|Integration]com.xingren.v.logging.integration.VloggingAnnotated[foo=xyz]
```

#### IntelliJ Plugin: 自动生成PSI Element，消除编译错误

至此，在命令行方式下，方法4已经可以正确运行。但在IDE环境中（比如IntelliJ，Eclipse），由于一般它们都会使用自定义的编译模型，需要额外实现一个插件来根据注解信息动态修改IDE的语法树，以避免编译错误。对于IntelliJ而言，使用的是[PSI模型](http://www.jetbrains.org/intellij/sdk/docs/basics/architectural_overview/psi_elements.html)，相应的插件代码如下：

```java
// 继承com.intellij.psi.augment.PsiAugmentProvider类

@NotNull
@Override
public <Psi extends PsiElement> List<Psi> getAugments(@NotNull PsiElement psiElement, @NotNull Class<Psi> type) {
    final List<Psi> emptyResult = Collections.emptyList();
    // skip processing during index rebuild
    final Project project = psiElement.getProject();
    if (DumbService.isDumb(project)) {
        return emptyResult;
    }
    // Expecting that we are only augmenting an PsiClass
    // Don't filter !isPhysical elements or code auto completion will not work
    if (!(psiElement instanceof PsiExtensibleClass) || !psiElement.isValid()) {
        return emptyResult;
    }
    // filter non-field type
    if (!PsiField.class.isAssignableFrom(type)) {
        return emptyResult;
    }
    final PsiClass psiClass = (PsiClass) psiElement;
    // see AbstractClassProcessor#process()
    PsiAnnotation psiAnnotation = PsiAnnotationUtil.findAnnotation(psiClass, Slf4j.class);
    if (null == psiAnnotation) {
        return emptyResult;
    }
    // check cache first
    if (loggerCache.containsKey(psiClass.getQualifiedName())) {
        return Arrays.asList((Psi) loggerCache.get(psiClass.getQualifiedName()));
    }

    final PsiManager manager = psiClass.getContainingFile().getManager();
    final PsiElementFactory psiElementFactory = JavaPsiFacade.getElementFactory(project);
    PsiType psiLoggerType = psiElementFactory.createTypeFromText(LOGGER_TYPE, psiClass);
    LightFieldBuilder loggerField = new LightFieldBuilder(manager, LOGGER_NAME, psiLoggerType);
    LightModifierList modifierList = (LightModifierList) loggerField.getModifierList();
    modifierList.addModifier(PsiModifier.PRIVATE);
    modifierList.addModifier(PsiModifier.STATIC);
    modifierList.addModifier(PsiModifier.FINAL);
    loggerField.setContainingClass(psiClass);
    loggerField.setNavigationElement(psiAnnotation);

    final String loggerInitializerParameter = String.format(LOGGER_CATEGORY, psiClass.getName());
    final PsiExpression initializer = psiElementFactory.createExpressionFromText(String.format(LOGGER_INITIALIZER, loggerInitializerParameter), psiClass);
    loggerField.setInitializer(initializer);
    // add to cache
    loggerCache.put(psiClass.getQualifiedName(), loggerField);

    return Arrays.asList((Psi) loggerField);
}
```

#### 参考

- [GitHub: lombok](https://github.com/rzwitserloot/lombok)
- [GitHub: lombok-intellij-plugin](https://github.com/mplushnikov/lombok-intellij-plugin)
- [ANNOTATION PROCESSING 101](http://hannesdorfmann.com/annotation-processing/annotationprocessing101)
- [Java Compiler API](https://www.javacodegeeks.com/2015/09/java-compiler-api.html)
- [Creating Your First Plugin](http://www.jetbrains.org/intellij/sdk/docs/basics/getting_started.html)
