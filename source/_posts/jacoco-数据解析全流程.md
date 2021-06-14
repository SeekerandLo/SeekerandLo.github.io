---
title: 'jacoco: 数据解析全流程'
date: 2021-06-12 19:57:32
tags: 
  - jacoco
  - 覆盖率
categories: 服务端覆盖率
---

最近在做代码覆盖率相关工作开发，本文对 java 的覆盖率工具 jacoco 解析 exec 数据的流程做一个简单介绍，是通过阅读源码和 debug 的方式来学习。本文将通过 javaagent 的形式使用 jacoco。

<!-- more -->

## jacoco

> JaCoCo is a free code coverage library for Java, which has been created by the EclEmma team based on the lessons learned from using and integration existing libraries for many years.

相关资料指路：
1. [jacoco 官网](https://www.eclemma.org/jacoco/index.html)
2. [本文使用的测试项目](https://github.com/SeekerandLo/blog-projects/tree/main/jacoco-demo)


## 项目准备

通过 javaagent 方式使用 jacoco，首先需要转变 javaagent.jar、一个被测试项目、一个使用 jacoco api 生成覆盖率报告的项目。被测试项目这里使用一个简单的 spring boot 服务，提供一个 Controller 和一个 Service。

### aserver：被测试项目

配置被测试项目的 JVM 参数，如下：
```
-javaagent=jacocoagent.jar,includes=com.*.*,classdumpdir=[某路径],output=tcpserver,address=127.0.0.1,port=6300
```
配置完成后，启动被测试项目。之后就可以拉取到 exec 文件了。
被测试项目部分代码如下：

```java
// controller
@RestController
@RequestMapping("aserver/")
public class AController {
  @Autowired
  AService aService;
  @GetMapping("add.json")
  public ResponseEntity<Integer> add(@RequestParam Integer p1, @RequestParam Integer p2) {
  return ResponseEntity.ok(aService.add(p1, p2));
  }
  @GetMapping("if-test.json")
  public ResponseEntity<Boolean> ifTest(@RequestParam Boolean flag) {
  return ResponseEntity.ok(aService.ifTest(flag));
  }
}

// service
@Service
public class AService {
  public Integer add(Integer p1, Integer p2){
  System.out.println(p1);
  System.out.println(p2);
  return p1 + p2;
  }
  public Boolean ifTest(Boolean flag){
  if(flag){
    System.out.println("flag is true");
  }else {
    System.out.println("flag is false");
  }
  return flag;
  }
}
```

### jacoco-analyzer

直接使用的 jacoco 官网提供 api example：
- [ExecutionDataClient.java](https://www.jacoco.org/jacoco/trunk/doc/examples/java/ExecutionDataClient.java)
- [ReportGenerator.java](https://www.jacoco.org/jacoco/trunk/doc/examples/java/ReportGenerator.java)

## jacoco 数据解析过程

在刚接触 jacoco 时，一直有几个疑问：exec 文件是如何被利用的？为什么必须要使用被测试服务的源码和编译出来的 class 文件？
本文也是主要针对这几个问题去寻找答案。

根据 jacoco 的需要准备材料，如下：

![资源文件](./resources.PNG)

- used-classes：被测试项目的 class 文件
- used-java：被测试项目的源码文件
- jacoco-client.exec：拉取到 exec 文件

下面就根据官网例子提供 [ReportGenerator.java](https://www.jacoco.org/jacoco/trunk/doc/examples/java/ReportGenerator.java) 去看看解析过程吧！

### ReportGenerator.java

```java
public class ReportGenerator {
  private final String title;
  private final File executionDataFile;
  private final File classesDirectory;
  private final File sourceDirectory;
  private final File reportDirectory;

  private ExecFileLoader execFileLoader;

  public ReportGenerator(final File projectDirectory) {
  this.title = projectDirectory.getName();
  this.executionDataFile = new File(projectDirectory, "/test-site/jacoco-client.exec");
  this.classesDirectory = new File(projectDirectory, "/test-site/used-classes");
  this.sourceDirectory = new File(projectDirectory, "/test-site/used-java");
  this.reportDirectory = new File(projectDirectory, "/test-site/coveragereport");
  }


  public void create() throws IOException {
  // 加载 exec 文件
  loadExecutionData();
  // 解析数据
  final IBundleCoverage bundleCoverage = analyzeStructure();
  // 创建报告
  createReport(bundleCoverage);
  }

  private void createReport(final IBundleCoverage bundleCoverage)
    throws IOException {

  final HTMLFormatter htmlFormatter = new HTMLFormatter();
  final IReportVisitor visitor = htmlFormatter
    .createVisitor(new FileMultiReportOutput(reportDirectory));

  visitor.visitInfo(execFileLoader.getSessionInfoStore().getInfos(),
    execFileLoader.getExecutionDataStore().getContents());

  visitor.visitBundle(bundleCoverage,
    new DirectorySourceFileLocator(sourceDirectory, "utf-8", 4));

  visitor.visitEnd();

  }

  private void loadExecutionData() throws IOException {
  execFileLoader = new ExecFileLoader();
  execFileLoader.load(executionDataFile);
  }

  private IBundleCoverage analyzeStructure() throws IOException {
  final CoverageBuilder coverageBuilder = new CoverageBuilder();
  final Analyzer analyzer = new Analyzer(
    execFileLoader.getExecutionDataStore(), coverageBuilder);

  analyzer.analyzeAll(classesDirectory);

  return coverageBuilder.getBundle(title);
  }

  public static void main(final String[] args) throws IOException {
  String path = System.getProperty("user.dir");
  final ReportGenerator generator = new ReportGenerator(
    new File(path));
  generator.create();
  }
}
```

通过主函数可以看到首先调用了 `generator.create()` 方法，该方法中将报告的生成分成了 3 步，后续我也根据这 3 步来分别分析。


#### loadExecutionData：加载 exec 文件
```java
// private ExecFileLoader execFileLoader;
// this.executionDataFile = new File(projectDirectory, "/test-site/jacoco-client.exec");

private void loadExecutionData() throws IOException {
  execFileLoader = new ExecFileLoader();
  execFileLoader.load(executionDataFile);
}
```
很简洁的代码，首先创建一个 exec 文件加载器，然后加载它。
```java
// ExecFileLoader.java

public ExecFileLoader() {
  sessionInfos = new SessionInfoStore();
  executionData = new ExecutionDataStore();
}

private ISessionInfoVisitor sessionInfoVisitor = null;

private IExecutionDataVisitor executionDataVisitor = null;

public void setSessionInfoVisitor(final ISessionInfoVisitor visitor) {
  this.sessionInfoVisitor = visitor;
}

public void setExecutionDataVisitor(final IExecutionDataVisitor visitor) {
  this.executionDataVisitor = visitor;
}

public void load(final File file) throws IOException {
  final InputStream stream = new FileInputStream(file);
  try {
  // 1. 创建一个 InputStream，然后传给 load 方法（看来是要读文件了）
  load(stream);
  } finally {
  stream.close();
  }
}

public void load(final InputStream stream) throws IOException {
  // 2. 创建一个 ExecutionDataReader（一个 exec 文件专用的 Reader？）
  final ExecutionDataReader reader = new ExecutionDataReader(
  new BufferedInputStream(stream));
  // 3. 给 reader set 一个 ExecutionDataStore（看方法名字是一个 Visitor，怎么用？）
  reader.setExecutionDataVisitor(executionData);
  // 4. 给 reader set 一个 SessionInfoStore（什么是 SessionInfo？）
  reader.setSessionInfoVisitor(sessionInfos);
  reader.read();
}
```

再进一层，去看看是怎么读取的

```java
// ExcutionDataReader.java
protected final CompactDataInput in;

// 1.构造函数
public ExecutionDataReader(final InputStream input) {
  // CompactDataInput：压缩数据输入（这里先不深入，理解为 jacoco 自己写的 exec 文件，自己就有版本解析）
  this.in = new CompactDataInput(input);
}

// 2.实际的读方法（读完后，会改变什么呢？）
public boolean read()
  throws IOException, IncompatibleExecDataVersionException {
  byte type;
  do {
  int i = in.read();
  if (i == -1) {
  return false; // EOF
  }
  type = (byte) i;
  if (firstBlock && type != ExecutionDataWriter.BLOCK_HEADER) {
  throw new IOException("Invalid execution data file.");
  }
  firstBlock = false;
  } while (readBlock(type));
  return true;
}

// 3.读ing...
protected boolean readBlock(final byte blocktype) throws IOException {
  // 根据读到的 type 再调用不同的方法，type 分三个
  // ExecutionDataWriter.BLOCK_HEADER: 0x01
  // ExecutionDataWriter.BLOCK_SESSIONINFO: 0x10
  // ExecutionDataWriter.BLOCK_EXECUTIONDATA: 0x11
  switch (blocktype) {
  case ExecutionDataWriter.BLOCK_HEADER:
  readHeader();
  return true;
  case ExecutionDataWriter.BLOCK_SESSIONINFO:
  readSessionInfo();
  return true;
  case ExecutionDataWriter.BLOCK_EXECUTIONDATA:
  // 先关注 ExecutionData
  readExecutionData();
  return true;
  default:
  throw new IOException(
  format("Unknown block type %x.", Byte.valueOf(blocktype)));
  }
}

// 4.读 ExecutionData
private void readExecutionData() throws IOException {
  if (executionDataVisitor == null) {
  throw new IOException("No execution data visitor.");
  }
  final long id = in.readLong();
  final String name = in.readUTF();
  final boolean[] probes = in.readBooleanArray();
  // 到这里为止，把 exec 文件中的数据读出来了，详情请看下图
  executionDataVisitor
  .visitClassExecution(new ExecutionData(id, name, probes));
}
```
下图中可以看到，exec 文件读取时是按类读取的，每个类有三个属性：id、name、probes（探针），其中 probes 存放的是一个 boolean 数组，再结合 jacoco 的插桩知识，可以得出 **probes 是指令的覆盖情况，true 代表指令被执行，false 代表指令未执行**。

<img src="./exec-load.PNG" style="zoom:80%;"/>

数据读取之后，将 id、name、probes 三个属性封装进 jacoco 中使用的 ExecutionData，之后又调用了 executionDataVisitor（executionDataVisitor 是 ExecutionDataStore 的对象） 的 visitClassExecution 方法。

```java
// ExecutionDataStore.java

private final Map<Long, ExecutionData> entries = new HashMap<Long, ExecutionData>();

private final Set<String> names = new HashSet<String>();

public void put(final ExecutionData data) throws IllegalStateException {
  final Long id = Long.valueOf(data.getId());
  final ExecutionData entry = entries.get(id);
  if (entry == null) {
  entries.put(id, data);
  names.add(data.getName());
  } else {
  entry.merge(data);
  }
}

// 1. 将 exectionData 存储起来，ExecutionDataStore 中有两个存储的结构：
// 一个 HashMap：以 executionData 的 id 为 key，executionData 本身为 value 存储。
// 一个 Set：存储 executionData 的 name，name 就是全类名。 
public void visitClassExecution(final ExecutionData data) {
  put(data);
}
```

至此，loadExecutionData 步骤执行完毕，**最后拥有了一个 executionDataStore 对象，在其之中，存放着每个类的指令覆盖情况**。

<img src="./executionDataStore.PNG" style="zoom:90%;"/>

#### analyzeStructure：分析结构
```java
//  private final File classesDirectory;
//  this.classesDirectory = new File(projectDirectory, "/test-site/used-classes");

//  private final String title;
//  this.title = projectDirectory.getName();

private IBundleCoverage analyzeStructure() throws IOException {
  // 1.创建一个 CoverageBuilder（什么是 CoverageBuilder？）
  final CoverageBuilder coverageBuilder = new CoverageBuilder();
  // 2.创建一个 Analyzer（什么是 Analyzer？）
  final Analyzer analyzer = new Analyzer(
    execFileLoader.getExecutionDataStore(), coverageBuilder);
  // 3.解析
  analyzer.analyzeAll(classesDirectory);
  // 4.返回
  return coverageBuilder.getBundle(title);
}
```
上述代码结构也很清晰，首先创建了一个 CoverageBuilder 对象，然后创建了一个 Analyzer 对象，然后解析，然后返回。

- CoverageBuilder：完全解析之后的覆盖率数据就存在这里，按类分好，之后可以通过类名等获取相应内容。
- Analyzer：解析器...

简单看下两个类的构造函数：

```java
// CoverageBuilder.java
public CoverageBuilder() {
  this.classes = new HashMap<String, IClassCoverage>();
  this.sourcefiles = new HashMap<String, ISourceFileCoverage>();
}

// Analyzer.java
private final ExecutionDataStore executionData;
private final ICoverageVisitor coverageVisitor;
private final StringPool stringPool;

public Analyzer(final ExecutionDataStore executionData,
    final ICoverageVisitor coverageVisitor) {
  this.executionData = executionData;
  this.coverageVisitor = coverageVisitor;
  this.stringPool = new StringPool();
}
```

正式进入分析数据的过程：`analyzer.analyzeAll(classesDirectory)`，传入进方法的是一个 File 对象，它本身又是一个文件夹。

```java
// Analyzer.java

public int analyzeAll(final File file) throws IOException {
  int count = 0;
  // 1.判断是否是文件夹
  if (file.isDirectory()) {
    for (final File f : file.listFiles()) {
      // 2.子 File 对象进行递归调用
      count += analyzeAll(f);
    }
  } else {
    // 3.如果不是文件夹了，那就是 class 文件了，开始进入正题...
    final InputStream in = new FileInputStream(file);
    try {
      // 4.再调用下同类中的 analyzeAll 方法，传入的参数是一个流对象和 class 文件的绝对路径，详情请看下方的代码
      count += analyzeAll(in, file.getPath());
    } finally {
      in.close();
    }
  }
  return count;
}

// 第二个 analyzeAll 方法
public int analyzeAll(final InputStream input, final String location)
    throws IOException {
  final ContentTypeDetector detector;
  try {
    // 1.创建了一个“内容类型检测器”，它能知道将要分析的文件的类型是什么，不展开讲
    detector = new ContentTypeDetector(input);
  } catch (final IOException e) {
    throw analyzerError(location, e);
  }
  switch (detector.getType()) {
  // 2.文件的类型是 class，之后的分析将会进入 analyzeClass 方法
  case ContentTypeDetector.CLASSFILE:
    // 3.调用同类中的 analyzeClass 方法，传入的参数有流对象和对应 class 的绝对路径，详情请继续看下方代码
    analyzeClass(detector.getInputStream(), location);
    return 1;
  case ContentTypeDetector.ZIPFILE:
    return analyzeZip(detector.getInputStream(), location);
  case ContentTypeDetector.GZFILE:
    return analyzeGzip(detector.getInputStream(), location);
  case ContentTypeDetector.PACK200FILE:
    return analyzePack200(detector.getInputStream(), location);
  default:
    return 0;
  }
}

// analyzeClass
public void analyzeClass(final InputStream input, final String location)
    throws IOException {
  final byte[] buffer;
  try {
    // 1.将流读成了字节数组
    buffer = InputStreams.readFully(input);
  } catch (final IOException e) {
    throw analyzerError(location, e);
  }
  // 2.继续调用同类中的 analyzeClass 方法，传入的参数是字节数组和绝对路径
  analyzeClass(buffer, location);
}

// 第二个 analyzeClass 方法
public void analyzeClass(final byte[] buffer, final String location)
    throws IOException {
  try {
    // 1.继续调用同类中的 analyzeClass 方法，传入的参数是字节数组
    analyzeClass(buffer);
  } catch (final RuntimeException cause) {
    throw analyzerError(location, cause);
  }
}

// 第三个 analyzeClass 方法
private void analyzeClass(final byte[] source) {
  final long classId = CRC64.classId(source);
  // 1.创建 ClassReader
  final ClassReader reader = InstrSupport.classReaderFor(source);
  if ((reader.getAccess() & Opcodes.ACC_MODULE) != 0) {
    return;
  }
  if ((reader.getAccess() & Opcodes.ACC_SYNTHETIC) != 0) {
    return;
  }
  // 2.创建访问者
  final ClassVisitor visitor = createAnalyzingVisitor(classId,
      reader.getClassName());
  // 3.“开始访问”
  reader.accept(visitor, 0);
}
```
经过上面几个 `analyzeAll` 和 `analyzeClass` 方法可以看出，jacoco 是将对各个类“各自击破”，将要去执行实际的分析过程时，类文件已经变成了字节数组。这最后一个 `analyzeClass` 方法会去执行实际的分析过程，它首先创建了一个 ClassReader（asm 包中的工具，用来读取和分析类文件），后面分析利用了[访问者模式](https://www.runoob.com/design-pattern/visitor-pattern.html)，jacoco 创建自己的 visitor 实现自己的逻辑。

> asm 包中的 ClassVisitor.java 和 MethodVisitor.java 有很多“钩子”可以重写，jacoco 这里主要重写了 visitEnd 方法，来实现对 Class 和 Method 的访问。

访问流程还是通过阅读代码来跟进，在阅读源码之前有几个问题需要注意下：

1. 经过 loadExecutionData 方法，exec 文件中的指令覆盖情况已经被记录到 executionDataStore 对象中了，但是只知道指令的覆盖情况，怎么知道行的情况、分支的情况、类的情况？换句话说 jacoco 怎么将指令的覆盖情况合并为更具体的覆盖情况？

下面继续阅读代码，首先回到 jacoco 创建 visitor 的方法。

```java
// Analyzer.java
private ClassVisitor createAnalyzingVisitor(final long classid,
    final String className) {
  // 1.classId 是通过解析字节数组获取到的，应该与 exec 文件中解析出来的 classId 相同
  // 从 executionData（上文提到，这是一个 ExecutionDataStore 类，按类分别存放类的 ExecutionData）
  final ExecutionData data = executionData.get(classid);
  final boolean[] probes;
  final boolean noMatch;
  if (data == null) {
    // 2.如果从 executionData 对象中获取不到当前类的信息，probes（指令覆盖情况）置为 null
    // noMatch 是通过 executionData 中存放 className 的 set 来判断的
    probes = null;
    noMatch = executionData.contains(className);
  } else {
    // 3.如果 executionData 对象中能获取到类的信息，就正常赋值
    probes = data.getProbes();
    noMatch = false;
  }
  // 4.创建一个 ClassCoverageImpl 对象（ClassCoverageImpl 这是干什么的？）
  final ClassCoverageImpl coverage = new ClassCoverageImpl(className,
      classid, noMatch);
  // 5.创建一个 ClassAnalyzer，这就是解析器了，还将刚才创建的对象都传入了，并且重写了一个 visitEnd 方法（ClassAnalyzer 继承自 ClassProbesVisitor，ClassProbesVisitor 又继承自 ClassVisitor，所以这里可以重写 visitEnd 方法）
  // coverage：ClassCoverageImpl 对象，不详讲，先记得这是每个类的各维度覆盖率计数器，覆盖率数据可以通过这个类获取
  // probes：一个类的指令覆盖情况
  // stringPool：暂不清楚用处
  final ClassAnalyzer analyzer = new ClassAnalyzer(coverage, probes,
      stringPool) {
    // （后续的逻辑会执行到这里，当访问完一个类会调用到这个方法，后文详细介绍）
    @Override
    public void visitEnd() {
      super.visitEnd();
      coverageVisitor.visitCoverage(coverage);
    }
  };
  // 6.创建一个 ClassProbesAdpater，该类也继承自 ClassVisitor
  // 这里先了解，ClassProbesAdapter 也是一个 ClassVisitor
  return new ClassProbesAdapter(analyzer, false);
}
```

到目前为止，出现的类已经比较多了，又是适配器又是访问者，jacoco 这里的水还是很深的，但是我先不关注其他的地方，还是以数据分析过程为主题去看，后面出一个 jacoco 数据分析相关类的关系图。话不多说，终于该进入到 ClassReader 了。

（ClassAnalyzer、ClassProbesAdpater、ClassVisitor的关系，先欠着）

ClassReader 中的 accept 方法接受一个 visitor（需要继承 asm 中的 ClassVisitor，ClassVisitor 是一个抽象类），visitor 可以通过重写 ClassVisitor 的方法来实现一些逻辑，ClassVisitor 中定义的方法有：

<img src="./ClassVisitor.PNG" style="zoom:80%;"/>

ClassVisitor 中具有两个属性，api 和 ClassVisitor 对象，并且在它的 visitXXX 方法中，都会判断 ClassVisitor 对象是不是 null，如果不是 null 则调用其相应的方法。如：

```java
// ClassVisitor.java

/**
 * Visits the end of the class. This method, which is the last one to be called, is used to inform
 * the visitor that all the fields and methods of the class have been visited.
 */
public void visitEnd() {
  if (cv != null) {
    cv.visitEnd();
  }
}

// -----------------------------------------------------------------------------------------------
// ClassProbesAdapter.java

// 这里再看 jacoco 在创建 visitor 时方法
// createAnalyzingVisitor 最后 return new ClassProbesAdapter(analyzer, false);
// ClassProbesAdapter 继承自 ClassVisitor，它的构造方法如下：
public ClassProbesAdapter(final ClassProbesVisitor cv,
    final boolean trackFrames) {
  super(InstrSupport.ASM_API_VERSION, cv);
  // 这里已经将 jacoco 的 ClassAnalyzer 赋值到了，所以当在 ClassReader 中调用 ClassVisitor 的 visitEnd 的方法时，会走到 ClassAnalyzer 重写的 visitEnd 方法中
  this.cv = cv;
  this.trackFrames = trackFrames;
}
```

ClassReader 的 accept 中会执行一些逻辑，这里也不详写，目前只知道在这个方法中可以得到当前类中有几个方法...

```java
// ClassReader.java

// 1.Analyzer 中 reader.accept 首先调用的这个方法
public void accept(final ClassVisitor classVisitor, final int parsingOptions) {
  accept(classVisitor, new Attribute[0], parsingOptions);
}

// 2.实际的逻辑
public void accept(final ClassVisitor classVisitor, final Attribute[] attributePrototypes, final int parsingOptions) {
  // ...很多逻辑...跳过...

  // 3.methodsCount 是这个类中方法的数量
  int methodsCount = readUnsignedShort(currentOffset);
  currentOffset += 2;
  while (methodsCount-- > 0) {
    // 4.处理 method
    currentOffset = readMethod(classVisitor, context, currentOffset);
  }
  // Visit the end of the class.
  classVisitor.visitEnd();
}

```
通过下图可以看到，在 AController 中 methodCounts 是 3（这里有一个疑问，在 AController 中明明我只写两个方法，为什么 methodCounts 会是 3 呢...这里发现每个类都有一个名字是 `<init>` 的方法，这里先不考虑）

<img src="./methodsCount.PNG" style="zoom:80%;"/>

readMethod 方法的注释是这样写的：“Reads a JVMS method_info structure and makes the given visitor visit it.”，意思就是“读取JVMS方法信息结构，并让给定的访问者访问它”，在这个方法中将会创建一个 MethodVisitor，以此来访问方法。

```java
// ClassReader.java

private int readMethod(final ClassVisitor classVisitor, final Context context, final int methodInfoOffset) {
  // ...很多逻辑...

  // 1.传入到这个方法中的类访问者的类型是 jacoco 中的 ClassProbesAdapter
  // 所以执行的 visitMethod 的方法也是 ClassProbesAdapter 中重写的 visitMethod 方法
  MethodVisitor methodVisitor =
      classVisitor.visitMethod(
          context.currentMethodAccessFlags,
          context.currentMethodName,
          context.currentMethodDescriptor,
          signatureIndex == 0 ? null : readUtf(signatureIndex, charBuffer),
          exceptions);
  
  // ...
  methodVisitor.visitEnd();
  return currentOffset;
}

// -----------------------------------------------------------------------------------------------

// ClassProbesAdapter.java

@Override
public final MethodVisitor visitMethod(final int access, final String name,
    final String desc, final String signature,
    final String[] exceptions) {
  final MethodProbesVisitor methodProbes;
  // 1.此处的 cv 是 ClassAnalyzer 对象，在 Analyzer 类中 createAnalyzingVisitor 中有记录
  // 这里调用的方法的逻辑在下面介绍，可先看下面 ClassAnalyzer.java 的 visitMethod 方法
  final MethodProbesVisitor mv = cv.visitMethod(access, name, desc,
      signature, exceptions);
  if (mv == null) {
    methodProbes = EMPTY_METHOD_PROBES_VISITOR;
  } else {
    methodProbes = mv;
  }
  return new MethodSanitizer(null, access, name, desc, signature,
      exceptions) {
    @Override
    public void visitEnd() {
      super.visitEnd();
      LabelFlowAnalyzer.markLabels(this);
      final MethodProbesAdapter probesAdapter = new MethodProbesAdapter(
          methodProbes, ClassProbesAdapter.this);
      if (trackFrames) {
        final AnalyzerAdapter analyzer = new AnalyzerAdapter(
            ClassProbesAdapter.this.name, access, name, desc,
            probesAdapter);
        probesAdapter.setAnalyzer(analyzer);
        methodProbes.accept(this, analyzer);
      } else {
        methodProbes.accept(this, probesAdapter);
      }
    }
  };
}

// -----------------------------------------------------------------------------------------------

// ClassAnalyzer.java
private final ClassCoverageImpl coverage;
private final boolean[] probes;
private final StringPool stringPool;

@Override
public MethodProbesVisitor visitMethod(final int access, final String name,
    final String desc, final String signature,
    final String[] exceptions) {
  // 1.coverage：创建 ClassAnzlyzer 时已经将 coverage 传进来了，是一个 ClassCoverageImpl 对象
  // name 是类的名字
  InstrSupport.assertNotInstrumented(name, coverage.getName());

  // 2.probes；在创建 ClassAnalyzer 时已经将类的 probes 传进来了
  final InstructionsBuilder builder = new InstructionsBuilder(probes);

  // 3.MethodAnalyzer 继承自 MethodProbesVisitor，MethodProbesVisitor 继承自 MethodVisitor
  return new MethodAnalyzer(builder) {
    @Override
    public void accept(final MethodNode methodNode,
        final MethodVisitor methodVisitor) {
      super.accept(methodNode, methodVisitor);
      addMethodCoverage(stringPool.get(name), stringPool.get(desc),
          stringPool.get(signature), builder, methodNode);
    }
  };
}

```

经过上述代码后可以知道 ClassReader 中 readMethod 方法会调用传入的 classVisitor 对象的 visitMethod 方法来获取一个 MethodVisitor 对象，在 readMethod 方法中获取到 MethodVisitor 对象实际是一个 MethodSanitizer 对象，它重写了 visitEnd 方法。在其重写的 visitEnd 方法中它又会执行 `methodProbes.accept()` 的方法，这个 methodProbes 对象是一个 MethodAnalyzer 的实例，所以当执行 methodProbes.accept 时，会执行 MethodAnalyzer 中重写的 accept 方法。

下面继续回到 readMethod 方法中，当执行到最后时，调用了 methodVisitor.visitEnd 方法，调用链如下：

`MethodSanitizer.visitEnd` -> `MethodAnalyzer.accept`

```java
// MethodAnalyzer.java 

// 当 ClassReader 中 readMethod 中的 methodVisitor.visitEnd 执行时，会执行下方的方法。
// 看这部分代码的逻辑，是获取当前方法的指令集合，并继续利用访问者模式，各种类型的指令集合进行操作
@Override
public void accept(final MethodNode methodNode,
    final MethodVisitor methodVisitor) {
  methodVisitor.visitCode();
  for (final TryCatchBlockNode n : methodNode.tryCatchBlocks) {
    n.accept(methodVisitor);
  }
  for (final AbstractInsnNode i : methodNode.instructions) {
    currentNode = i;
    // 1. 访问者模式，传入 methodVisitor 是 MethodProbesAdapter 对象
    i.accept(methodVisitor);
  }
  methodVisitor.visitEnd();
}
```

在 MethodAnalyzer 中 accept 方法可以看到，这里将指令分别处理了。会首先调用 AbstractInsnNode 的 accept 方法，各种类型的 AbstractInsnNode 的 accept 方法又有不同，下面看下 AbstractInsnNode 的种类有哪些：

<img src="./AbstractInsnNode.PNG" style="zoom:80%;"/>

它们中的 accept 方法的逻辑举例：

```java
// MethodInsnNode.java
@Override
public void accept(final MethodVisitor methodVisitor) {
  methodVisitor.visitMethodInsn(opcode, owner, name, desc, itf);
  acceptAnnotations(methodVisitor);
}

// -----------------------------------------------------------------------------------------------

// LabelNode.java
@Override
public void accept(final MethodVisitor methodVisitor) {
  methodVisitor.visitLabel(getLabel());
}

// -----------------------------------------------------------------------------------------------

// InsnNode.java
@Override
public void accept(final MethodVisitor methodVisitor) {
  methodVisitor.visitInsn(opcode);
  acceptAnnotations(methodVisitor);
}
```

发现各种类型的指令首先还是会调用 MethodVisitor 对象的 visitXXX 方法，那么就看下这里将会调用的实际逻辑吧

```java
// MethodAnalyzer.java
// 省略了部分，在这个类中，会所有种类的指令都编写了方法，下面是部分内容

  private final InstructionsBuilder builder;

  @Override
  public void visitLabel(final Label label) {
    // 1.可以看到，这里都会调用 builder 的方法，builder 是一个 InstructionsBuilder 对象
    builder.addLabel(label);
  }

  @Override
  public void visitLineNumber(final int line, final Label start) {
    builder.setCurrentLine(line);
  }

  @Override
  public void visitInsn(final int opcode) {
    // 2.这里大部分指令都会调用 addInstruction 方法，这里主要关注这个方法
    builder.addInstruction(currentNode);
  }

  // 3.同时还注意到有如下的几个方法，它们会传入指令的 id，然后调用 builder 的 addProbe 方法
  // 这里是不是就是我要找到指令与覆盖情况结合的逻辑呢？
  @Override
  public void visitProbe(final int probeId) {
    builder.addProbe(probeId, 0);
    builder.noSuccessor();
  }

  @Override
  public void visitJumpInsnWithProbe(final int opcode, final Label label,
      final int probeId, final IFrame frame) {
    builder.addInstruction(currentNode);
    builder.addProbe(probeId, 1);
  }

  @Override
  public void visitInsnWithProbe(final int opcode, final int probeId) {
    builder.addInstruction(currentNode);
    builder.addProbe(probeId, 0);
  }

```

执行到这里可以知道，jacoco 在解析类时，会先解析类中的方法，在解析方法时又是从组成方法的指令出发。在遍历 methodNode.instructions 时发现，当在解析一个方法时，前面两个指令分别是 LabelNode 和 LineNumberNode。（LineNumberNode 可以将 InstructionBuilder 中的 currentLine 置为当前的行）

```java
// InstructionBuilder.java

private final boolean[] probes;

private int currentLine;

private Instruction currentInsn;

private final Map<AbstractInsnNode, Instruction> instructions;

private final List<Label> currentLabel;

// ...省略部分属性

void addInstruction(final AbstractInsnNode node) {
  // 1. 当在访问每个指令的时候，当进入一个新的行时，currentLine 会被赋值为新的行号
  final Instruction insn = new Instruction(currentLine);
  final int labelCount = currentLabel.size();
  if (labelCount > 0) {
    for (int i = labelCount; --i >= 0;) {
      LabelInfo.setInstruction(currentLabel.get(i), insn);
    }
    currentLabel.clear();
  }
  if (currentInsn != null) {
    currentInsn.addBranch(insn, 0);
  }
  currentInsn = insn;
  instructions.put(node, insn);
}

// 2.可以看到这里是从 probes 中获取指令覆盖情况，以此来判断是否执行！
void addProbe(final int probeId, final int branch) {
  final boolean executed = probes != null && probes[probeId];
  // 3.请看下方的 Instruction 的 addBranch 方法
  currentInsn.addBranch(executed, branch);
}

// -------------------------------------------------------------------------------------

// Instruction.java

// Instruction 是 jacoco 中用来描述每个指令的覆盖情况的对象
// 在这之前，jacoco 只有一个 boolean 数组来记录指令的覆盖情况，但是还没有关联到具体的指令上
private final int line;
private int branches;

// 指令是否被执行记录于此，是一个 BitSet
private final BitSet coveredBranches;
private Instruction predecessor;
private int predecessorBranch;

// 4. 
// executed：是否被执行了
// branch: 是否是分支
public void addBranch(final boolean executed, final int branch) {
  branches++;
  if (executed) {
    propagateExecutedBranch(this, branch);
  }
}

private static void propagateExecutedBranch(Instruction insn, int branch) {
  while (insn != null) {
    if (!insn.coveredBranches.isEmpty()) {
      insn.coveredBranches.set(branch);
      break;
    }
    insn.coveredBranches.set(branch);
    branch = insn.predecessorBranch;
    insn = insn.predecessor;
  }
}

// 5. 这里是指令计数器，会返回指令的覆盖情况
// 如果 coveredBranches 是空的，就会返回一个 ICounter 对象，这个对象的 missed 是 1，covered 是 0
// 如果 coveredBranches 不是空的，还是返回一个 ICounter 对象，这个对象的 missed 是 0，covered 是 1
public ICounter getInstructionCounter() {
  return coveredBranches.isEmpty() ? CounterImpl.COUNTER_1_0
      : CounterImpl.COUNTER_0_1;
}
```

在经过了 Method.accept 之后，类中的一个方法的每个指令就与 exec 文件中的覆盖情况结合了。然后会继续执行 ClassAnalyzer 的 addMethodCoverage 方法。

```java
// ClassAnalyzer.java

// name 是方法名字，desc 是返回值，signature 是入参和返回值
private void addMethodCoverage(final String name, final String desc,
    final String signature, final InstructionsBuilder icc,
    final MethodNode methodNode) {
  // 1.icc 是 InstructionBuilder，其中记录着每个指令的覆盖情况
  final MethodCoverageCalculator mcc = new MethodCoverageCalculator(
      icc.getInstructions());
  filter.filter(methodNode, this, mcc);
  final MethodCoverageImpl mc = new MethodCoverageImpl(name, desc,
      signature);
  // 2.实际的计算逻辑，详情看下方截图
  mcc.calculate(mc);
  if (mc.containsCode()) {
    // Only consider methods that actually contain code
    coverage.addMethod(mc);
  }
}
```

下图是展示 name、desc、signature：

<img src="./name-desc-signature.PNG" style="zoom:80%;"/>

下图是展示 calculate 的实际逻辑，下图的 coverage 对象是 Method 级别的对象，是记录整个 Method 的覆盖情况的。再往下调用的 `coverage.increment` 方法其实做的是将指令级别的数据累加到方法上，如将方法中所有指令的 missed 相加，就是方法的 missed，将所有执行的 covered 相加，就是方法的 covered。

<img src="./calculate.PNG" style="zoom:80%;"/>

上图中最后调用的 `coverage.incrementMethodCounter()` 也是对方法覆盖情况的一个总结，具体逻辑如下图：

<img src="./incrementMethodCounter.PNG" style="zoom:80%;"/>

到这里为止，整个分析的流程就快要结束了，这里是针对方法级别的数据累加，当执行完 calculate 方法后，又会执行一个 `coverage.addMethod(mc)`，这里的 coverage 对象是类级别的对象，再往下执行就是将方法级别的数据累加到类上。

<img src="./coverage-addMethod.PNG" style="zoom:80%;"/>

如此如此，这般这般之后，发现 jacoco 中每个类的覆盖率数据是由指令的数据累加来的，这篇文章中最后也只得出了这个结论，在跟随源码的过程中了解到了数据走向，但是对 class 文件中的指令和 exec 文件中指令的覆盖情况结合的逻辑还不是特别清晰，这里就先粗略的理解。同时在跟进源码时发现，还需要对工具的数据结构和设计模式加强理解，比如 Instruction 处应该是使用了链表；访问者模式；适配器模式等等。之后在工作中如果有需要再来阅读代码，这篇文章的目的是先了解 jacoco 分析数据的思路。📚