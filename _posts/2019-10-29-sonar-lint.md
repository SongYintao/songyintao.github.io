---
title: Sonar Lint idea code
subtitle: Sonar lint idea 源码分析
tags: [idea,plugin]
layout: post
---

sonar lint 插件源码分析。

[idea plugin 插件开发文档](http://www.jetbrains.org/intellij/sdk/docs/basics/getting_started.html)

使用基于gradle的开发方式。

[具体步骤](http://www.jetbrains.org/intellij/sdk/docs/tutorials/build_system/prerequisites.html)

#  IDEA plugin开发指南



# Plugin Components

Components are the fundamental concept of plugin integration. There are three kinds of components:

- **Application level components** are created and initialized when your IDE starts up. They can be acquired from the [Application](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/core-api/src/com/intellij/openapi/application/Application.java) instance by using the `getComponent(Class)` method.
- **Project level components** are created for each [`Project`](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/core-api/src/com/intellij/openapi/project/Project.java) instance in the IDE. (Please note that components may be created even for unopened projects.) They can be acquired from the `Project` instance by using the `getComponent(Class)` method.
- **Module level components** are created for each [`Module`](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/core-api/src/com/intellij/openapi/module/Module.java) inside every project loaded in the IDE. Module level components can be acquired from a `Module` instance with the `getComponent(Class)` method.

**Every component should have interface and implementation classes specified in the configuration file. The interface class will be used for retrieving the component from other components, and the implementation class will be used for component instantiation.**

**Note that two components of the same level ([Application](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/core-api/src/com/intellij/openapi/application/Application.java), [Project](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/core-api/src/com/intellij/openapi/project/Project.java) or [Module](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/core-api/src/com/intellij/openapi/module/Module.java)) cannot have the same interface class.** The same class may be specified for both interface and Implementation.

Each component has a unique name which is used for its externalization and other internal needs. The name of a component is returned by its `getComponentName()` method.



## Components naming notation

It is recommended to name components in the form `<plugin_name>.<component_name>`.



## Application level components

An application component that has no dependencies should have a constructor with no parameters which will be used for its instantiation. If an application component depends on other application components, it can specify these components as constructor parameters. The *IntelliJ Platform* will ensure that the components are instantiated in the correct order to satisfy the dependencies.

Application level components must be registered in the `<application-components>` section of the plugin.xml file (see [Plugin Configuration File](http://www.jetbrains.org/intellij/sdk/docs/basics/plugin_structure/plugin_configuration_file.html)).



## Project level components

Optionally, a project level component’s implementation class may implement the [ProjectComponent](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/core-api/src/com/intellij/openapi/components/ProjectComponent.java) interface.

The constructor of a project level component can have a parameter of the [Project](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/core-api/src/com/intellij/openapi/project/Project.java) type, if it needs the project instance. It can also specify other application-level or project-level components as parameters, if it depends on those components.

Project level components must be registered in the `<project-components>` section of the `plugin.xml` file (see [Plugin Configuration File](http://www.jetbrains.org/intellij/sdk/docs/basics/plugin_structure/plugin_configuration_file.html)).



## Module level components

Optionally, a module level component’s implementation class may implement the [ModuleComponent](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/projectModel-api/src/com/intellij/openapi/module/ModuleComponent.java) interface.

The constructor of a module level component can have a parameter of the Module type, if it needs the module instance. It can also specify other application level, project level or module level components as parameters, if it depends on those components.

Module level components must be registered in the `<module-components>` section of the `plugin.xml` file (see [Plugin Configuration File](http://www.jetbrains.org/intellij/sdk/docs/basics/plugin_structure/plugin_configuration_file.html)).





## Persisting the state of components

The state of every component will be automatically saved and loaded if the component’s class implements the [JDOMExternalizable](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/util/src/com/intellij/openapi/util/JDOMExternalizable.java)(deprecated) or [PersistentStateComponent](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/projectModel-api/src/com/intellij/openapi/components/PersistentStateComponent.java) interface.

When the component’s class implements the [PersistentStateComponent](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/projectModel-api/src/com/intellij/openapi/components/PersistentStateComponent.java) interface, the component state is saved in an XML file that you can specify using the [@State](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/projectModel-api/src/com/intellij/openapi/components/State.java) and [@Storage](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/projectModel-api/src/com/intellij/openapi/components/Storage.java) annotations in your Java code.

When the component’s class implements the [JDOMExternalizable](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/util/src/com/intellij/openapi/util/JDOMExternalizable.java) interface, the components save their state in the following files:

- Project level components save their state to the project (`.ipr`) file.

  However, if the workspace option in the `plugin.xml` file is set to `true`, the component saves its configuration to the workspace (`.iws`) file instead.

- Module level components save their state to the module (`.iml`) file.

For more information and samples, refer to [Persisting State of Components](http://www.jetbrains.org/intellij/sdk/docs/basics/persisting_state_of_components.html).





## Defaults

The defaults (a component’s predefined settings) should be placed in the `<component_name>.xml` file. Place this file in the plugin’s classpath in the folder corresponding to the default package. The `readExternal()` method will be called on the `<component>` root tag.

If a component has defaults, the `readExternal()` method is called twice:

- The first time for defaults
- The second time for saved configuration





## Plugin components lifecycle

The components are loaded in the following order:

- Creation - constructor is invoked.
- Initialization - the `initComponent` method is invoked (if the component implements the [BaseComponent](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/core-api/src/com/intellij/openapi/components/BaseComponent.java) interface).
- Configuration - the `readExternal` method is invoked (if the component implements [JDOMExternalizable](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/util/src/com/intellij/openapi/util/JDOMExternalizable.java) interface), or the `loadState`method is invoked (if the component implements [PersistentStateComponent](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/projectModel-api/src/com/intellij/openapi/components/PersistentStateComponent.java) and has non-default persisted state).
- For module components, the `moduleAdded` method of the [ModuleComponent](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/projectModel-api/src/com/intellij/openapi/module/ModuleComponent.java) interface is invoked to notify that a module has been added to the project.
- For project components, the `projectOpened` method of the [ProjectComponent](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/core-api/src/com/intellij/openapi/components/ProjectComponent.java) interface is invoked to notify that a project has been loaded.

The components are unloaded in the following order:

- Saving configuration - the `writeExternal` method is invoked (if the component implements the [JDOMExternalizable](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/util/src/com/intellij/openapi/util/JDOMExternalizable.java) interface), or the `getState` method is invoked (if the component implements PersistentStateComponent).
- Disposal - the `disposeComponent` method is invoked.

Note that you should not request any other components using the `getComponent()` method in the constructor of your component, otherwise you’ll get an assertion. If you need access to other components when initializing your component, you can specify them as constructor parameters or access them in the `initComponent` method.





# Plugin Services

A *service* is a plugin component loaded on demand when your plugin calls the `getService` method of the [ServiceManager](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/core-api/src/com/intellij/openapi/components/ServiceManager.java) class.

The *IntelliJ Platform* ensures tha**t only one instance of a service is loaded even though the service is called several times.** A service must have an implementation class which is used for service instantiation. A service may also have an interface class which is used to obtain the service instance and provides API of the service. The interface and implementation classes are specified in the `plugin.xml` file.

The *IntelliJ Platform* offers three types of services: *application level* services, *project level* services and *module level* services.



## How to Declare a Service?

To declare a service, you can use the following extension points in the IntelliJ Platform:

- `applicationService`: designed to declare an application level service.
- `projectService`: designed to declare a project level service.
- `moduleService`: designed to declare a module level service.

**To declare a service:**

1. In your project, open the context menu of the destination package and click *New* (or press Alt+Insert).
2. In the *New* menu, choose *Plugin DevKit* and click *Application Service*, *Project Service* or *Module Service* depending on the type of service you need to use.
3. In the dialog box that opens, you can specify service interface and implementation, or just a service class if you uncheck *Separate interface from implementation* check box.

The IDE will generate new Java interface and class (or just a class if you unchecked *Separate interface from implementation* check box) and register the new service in `plugin.xml` file.

Declaring a service via *New* context menu is available since version **2017.3**.

To clarify the service declaration procedure, consider the following fragment of the `plugin.xml` file:

<extensions defaultExtensionNs="com.intellij">
  <!-- Declare the application level service -->
  <applicationService serviceInterface="Mypackage.MyApplicationService" 
                      serviceImplementation="Mypackage.MyApplicationServiceImpl" />

  <!-- Declare the project level service -->
  <projectService serviceInterface="Mypackage.MyProjectService" 
                  serviceImplementation="Mypackage.MyProjectServiceImpl" />
</extensions>

If `serviceInterface` isn’t specified, it’s supposed to have the same value as `serviceImplementation`.



## Retrieving a service

To instantiate your service, in Java code, use the following syntax:

```java
MyApplicationService applicationService =ServiceManager.getService(MyApplicationService.class); 

MyProjectService projectService = ServiceManager.getService(project,MyProjectService.class);  MyModuleService moduleService =ModuleServiceManager.getService(module,MyModuleService.class); 
```



### Sample Plugin

This section allows you to download and install a sample plugin illustrating how to create and use a plugin service. This plugin has a project component implementing a service that counts the number of currently opened projects in the IDE. If this number exceeds the maximum allowed number of simultaneously opened projects, the plugin returns an error message and closes the most recently opened project.

**To install and run the sample plugin**

- Download the included sample plugin project located [here](https://github.com/JetBrains/intellij-sdk-docs/tree/master/code_samples/max_opened_projects).
- Start *IntelliJ IDEA*, on the starting page, click *Open Project*, and then use the *Open Project* dialog box to open the project *max_opened_projects*.
- On the main menu, choose *Run | Run* or press Shift+F10.
- If necessary, change the [Run/Debug Configurations](https://www.jetbrains.com/help/idea/run-debug-configuration-plugin.html).





## SonarLint介绍

SonarLint is an IDE extension that helps you detect and fix quality issues as you write code. Like a spell checker, SonarLint squiggles flaws so they can be fixed before committing code.



## 项目结构

![](../img/sonarlint-project.png)

`plugin.xml` :插件的全局配置

#### 结构分析

#### 1. 应用组件

- SonarLintGlobalSettings

全局设置，至关重要，哪些rule需要设置、哪些不需要。

规则配置找他就好。



- SonarApplication
- SonarLintAppUtils
- SonarLintEngineManager
- GlobalLogOutput
- SonarLintActions
- SonarLintTelemetryImpl
- TelemetryManagerProvider

```xml
<application-components>
        <component>
            <implementation-class>org.sonarlint.intellij.config.global.SonarLintGlobalSettings</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.SonarApplication</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.util.SonarLintAppUtils</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.core.SonarLintEngineManager</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.util.GlobalLogOutput</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.core.SonarLintEngineFactory</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.util.SonarLintActions</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.telemetry.SonarLintTelemetryImpl</implementation-class>
            <interface-class>org.sonarlint.intellij.telemetry.SonarLintTelemetry</interface-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.telemetry.TelemetryManagerProvider</implementation-class>
        </component>
    </application-components>
```



#### 2. 项目组件

- SonarQubeEventNotifications

SonarQube事件通知

- SonarLintSubmitter

submit Analysis job。分析相关的文件，回调显示。

- AnalysisResultIssues
- ProjectBindingManager
- LiveIssueCache
- IssuePersistence
- SonarLintConsole
- SonarLintProjectSettings
- SonarLintProjectState
- SonarLintJobManager

分析Job管理器。生成SonarLintJob创建新的SonarLintTask进行工作。再根据task进行调度、执行。

SonarLintTask：核心部分

```java
@Override
public void run(ProgressIndicator indicator) {
  AccumulatorIssueListener listener = new AccumulatorIssueListener();
  sonarApplication.registerExternalAnnotator();

  try {
    checkCanceled(indicator, myProject);
		//分析核心
    List<AnalysisResults> results = analyze(myProject, indicator, listener);

		//返回问题issues
    List<Issue> issues = listener.getIssues();

    //所有监测不通过的文件
    List<ClientInputFile> allFailedAnalysisFiles = results.stream()
      .flatMap(r -> r.failedAnalysisFiles().stream())
      .collect(Collectors.toList());

    processor.process(job, indicator, issues, allFailedAnalysisFiles);
  } catch (CanceledException e1) {
    console.info("Analysis canceled");
  } catch (Throwable e) {
    handleError(e, indicator);
  } finally {
    myProject.getMessageBus().syncPublisher(TaskListener.SONARLINT_TASK_TOPIC).ended(job);
  }
}




private List<AnalysisResults> analyze(Project project, ProgressIndicator indicator, AccumulatorIssueListener listener) {
    //核心组件SonarLint分析器
    SonarLintAnalyzer analyzer = SonarLintUtils.get(project, SonarLintAnalyzer.class);

    indicator.setIndeterminate(true);
    int numModules = job.filesPerModule().keySet().size();
    String suffix = "";
    if (numModules > 1) {
      suffix = String.format(" in %d modules", numModules);
    }

    int numFiles = job.allFiles().size();
    if (numFiles > 1) {
      indicator.setText("Running SonarLint Analysis for " + numFiles + " files" + suffix);
    } else {
      indicator.setText("Running SonarLint Analysis for '" + getFileName(job.allFiles().iterator().next()) + "'");
    }

    LOGGER.info(indicator.getText());

    ProgressMonitor progressMonitor = new TaskProgressMonitor(indicator);
    List<AnalysisResults> results = new LinkedList<>();

    for (Map.Entry<Module, Collection<VirtualFile>> e : job.filesPerModule().entrySet()) {
      //核心部分，分析模块里面的文件
      results.add(analyzer.analyzeModule(e.getKey(), e.getValue(), listener, progressMonitor));
      checkCanceled(indicator, myProject);
    }
    indicator.startNonCancelableSection();
    return results;
  }
```



- IssueMatcher
- IssueProcessor
- SonarLintAnalyzer

核心的分析组件，主要分析工作靠它。

```java
//主要的核心代码逻辑
public AnalysisResults analyzeModule(Module module, Collection<VirtualFile> filesToAnalyze, IssueListener listener, ProgressMonitor progressMonitor) {
  // Configure plugin properties. Nothing might be done if there is no configurator available for the extensions loaded in runtime.
  long start = System.currentTimeMillis();
  
  Map<String, String> pluginProps = new HashMap<>();
  //分析配置扩展点获取
  AnalysisConfigurator[] analysisConfigurators = AnalysisConfigurator.EP_NAME.getExtensions();
  
  if (analysisConfigurators.length > 0) {
    for (AnalysisConfigurator config : analysisConfigurators) {
      console.debug("Configuring analysis with " + config.getClass().getName());
      pluginProps.putAll(config.configure(module));
    }
  } else {
    console.info("No analysis configurator found");
  }

  // configure files
  VirtualFileTestPredicate testPredicate = SonarLintUtils.get(module, VirtualFileTestPredicate.class);
  List<ClientInputFile> inputFiles = getInputFiles(module, testPredicate, filesToAnalyze);

  // Analyze

  try {
    SonarLintFacade facade = projectBindingManager.getFacade(true);

    String what;
    if (filesToAnalyze.size() == 1) {
      what = "'" + filesToAnalyze.iterator().next().getName() + "'";
    } else {
      what = filesToAnalyze.size() + " files";
    }

    console.info("Analysing " + what + "...");
    AnalysisResults result = facade.startAnalysis(inputFiles, listener, pluginProps, progressMonitor);
    console.debug("Done in " + (System.currentTimeMillis() - start) + "ms\n");
    if (result.languagePerFile().size() == 1 && result.failedAnalysisFiles().isEmpty()) {
      telemetry.analysisDoneOnSingleFile(result.languagePerFile().values().iterator().next(), (int) (System.currentTimeMillis() - start));
    } else {
      telemetry.analysisDoneOnMultipleFiles();
    }
    return result;
  } catch (InvalidBindingException e) {
    // should not happen, as analysis should not have been submitted in this case.
    throw new IllegalStateException(e);
  }
}
```

**SonarLintFacade**抽象类：封装了分析的细节



```java
  public synchronized AnalysisResults startAnalysis(List<ClientInputFile> inputFiles, IssueListener issueListener,
    Map<String, String> additionalProps, ProgressMonitor progressMonitor) {
    Path baseDir = Paths.get(project.getBasePath());
    Path workDir = baseDir.resolve(Project.DIRECTORY_STORE_FOLDER).resolve("sonarlint").toAbsolutePath();
    Map<String, String> props = new HashMap<>();
    props.putAll(additionalProps);
    props.putAll(projectSettings.getAdditionalProperties());
    return analyze(baseDir, workDir, inputFiles, props, issueListener, progressMonitor);
  }

//Standalone 实现

  private final StandaloneSonarLintEngine sonarlint;
@Override
  protected AnalysisResults analyze(Path baseDir, Path workDir, Collection<ClientInputFile> inputFiles, Map<String, String> props,
    IssueListener issueListener, ProgressMonitor progressMonitor) {
    
    //重要：获取需要分析的规则、跳过的规则
    List<RuleKey> excluded = globalSettings.getExcludedRules().stream().map(RuleKey::parse).collect(Collectors.toList());
    List<RuleKey> included = globalSettings.getIncludedRules().stream().map(RuleKey::parse).collect(Collectors.toList());

    StandaloneAnalysisConfiguration config = StandaloneAnalysisConfiguration.builder()
      .setBaseDir(baseDir)
      .addInputFiles(inputFiles)
      .putAllExtraProperties(props)
      .addExcludedRules(excluded)
      .addIncludedRules(included)
      .build();
    console.debug("Starting analysis with configuration:\n" + config.toString());
    
    //最终通过实现的SonarLint引擎进行具体的分析
    return sonarlint.analyze(config, issueListener, new ProjectLogOutput(console, projectSettings), progressMonitor);
  }


```

StandaloneSonarLintEngine：



```java
  @Override
  public AnalysisResults analyze(StandaloneAnalysisConfiguration configuration, IssueListener issueListener, @Nullable LogOutput logOutput, @Nullable ProgressMonitor monitor) {
    requireNonNull(configuration);
    requireNonNull(issueListener);
    setLogging(logOutput);
    rwl.readLock().lock();
    try {
      //实际的内核是调用：sonarlint-core jar里面方法
      return globalContainer.analyze(configuration, issueListener, new ProgressWrapper(monitor));
    } catch (RuntimeException e) {
      throw SonarLintWrappedException.wrap(e);
    } finally {
      rwl.readLock().unlock();
    }
  }

```



**sonarlint-core**：StandaloneGlobalContainer

```java
  public AnalysisResults analyze(StandaloneAnalysisConfiguration configuration, IssueListener issueListener, ProgressWrapper progress) {
    AnalysisContainer analysisContainer = new AnalysisContainer(globalExtensionContainer, progress);
    analysisContainer.add(configuration);
    analysisContainer.add(issueListener);
    analysisContainer.add(rules);
    // TODO configuration should be set directly with Strings
    Set<String> excludedRules = configuration.excludedRules().stream().map(RuleKey::toString).collect(Collectors.toSet());
    Set<String> includedRules = configuration.includedRules().stream()
      .map(RuleKey::toString)
      .filter(r -> !excludedRules.contains(r))
      .collect(Collectors.toSet());
    analysisContainer.add(standaloneActiveRules.filtered(excludedRules, includedRules));
    analysisContainer.add(SensorsExecutor.class);
    DefaultAnalysisResult defaultAnalysisResult = new DefaultAnalysisResult();
    analysisContainer.add(defaultAnalysisResult);
    analysisContainer.execute();
    return defaultAnalysisResult;
  }
```



- SonarLintProjectNotifications
- ServerIssueUpdater
- UpdateChecker
- SonarLintTaskFactory

根据SonarLintJob创建相关的SonarLintTask

- LocalFileExclusions

```xml
 <project-components>
        <component>
            <implementation-class>org.sonarlint.intellij.core.SonarQubeEventNotifications</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.trigger.SonarLintSubmitter</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.issue.AnalysisResultIssues</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.editor.CodeAnalyzerRestarter</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.core.ProjectBindingManager</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.issue.persistence.LiveIssueCache</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.issue.persistence.IssuePersistence</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.ui.SonarLintConsole</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.config.project.SonarLintProjectSettings</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.config.project.SonarLintProjectState</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.analysis.SonarLintJobManager</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.issue.IssueMatcher</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.issue.IssueManager</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.issue.IssueProcessor</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.analysis.SonarLintStatus</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.trigger.EditorOpenTrigger</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.trigger.EditorChangeTrigger</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.analysis.SonarLintAnalyzer</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.core.SonarLintProjectNotifications</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.core.ServerIssueUpdater</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.core.UpdateChecker</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.analysis.SonarLintTaskFactory</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.analysis.LocalFileExclusions</implementation-class>
        </component>
    </project-components>
```



#### 3. Action 用户触发操作



```xml
 <actions>
        <!-- Some actions are defined in SonarLintActions so that they aren't registered in ActionManager -->
        <!--idea上面AnalyzeMenu-->
        <group id="SonarLint.AnalyzeMenu" text="SonarLint" popup="false" icon="/images/ico-sonarlint-13.png">
            <separator/>
            <!-- This group is programmatically added to AnalyzeMenu if it exists -->
        </group>
        <!--鼠标右击「Analyze with SonarLint」，选择之后触发相关操作-->
        <action id="SonarLint.AnalyzeFiles"
                class="org.sonarlint.intellij.actions.SonarAnalyzeFilesAction"
                text="Analyze with SonarLint"
                description="Run SonarLint analysis on selected file(s)"
                icon="SonarLintIcons.SONARLINT">
            <keyboard-shortcut first-keystroke="shift ctrl S" keymap="$default"/>
            <!--触发该操作的位置-->
            <add-to-group group-id="EditorPopupMenu" anchor="last"/>
            <add-to-group group-id="SonarLint.AnalyzeMenu"/>
        </action>

        <!--项目菜单右击弹出，Exclude from SonarLint analysis-->
        <group id="SonarLint.ProjectViewPopupMenu" text="SonarLint" popup="true" icon="/images/ico-sonarlint-13.png">
            <reference ref="SonarLint.AnalyzeFiles"/>
            <action id="SonarLint.Exclude"
                    class="org.sonarlint.intellij.actions.ExcludeFileAction"
                    text="Exclude from SonarLint analysis"
                    description="Adds resources to the list exclusions in the SonarLint project settings"
                    icon="SonarLintIcons.SONARLINT">
            </action>
            <add-to-group group-id="ProjectViewPopupMenu"/>
        </group>

        <action id="SonarLint.AnalyzeChangedFiles"
                class="org.sonarlint.intellij.actions.SonarAnalyzeChangedFilesAction"
                text="Analyze VCS Changed Files with SonarLint"
                description="Run a SonarLint analysis on VCS changed files"
                icon="SonarLintIcons.SONARLINT">
            <add-to-group group-id="SonarLint.AnalyzeMenu"/>
        </action>

        <action id="SonarLint.AnalyzeAllFiles"
                class="org.sonarlint.intellij.actions.SonarAnalyzeAllFilesAction"
                text="Analyze All Files with SonarLint"
                description="Run a SonarLint analysis on all files in the project"
                icon="SonarLintIcons.SONARLINT">
            <add-to-group group-id="SonarLint.AnalyzeMenu"/>
        </action>

        <action id="SonarLint.toolwindow.Cancel"
                class="org.sonarlint.intellij.actions.SonarCancelAction"
                text="Cancel SonarLint Analysis"
                description="Cancel the SonarLint analysis running"
                icon="SonarLintIcons.SUSPEND">
        </action>

        <action id="SonarLint.toolwindow.Analyzers"
                class="org.sonarlint.intellij.actions.SonarShowCodeAnalyzers"
                text="Show more information"
                description="Show more information"
                icon="SonarLintIcons.INFO">
        </action>


        <action id="SonarLint.toolwindow.Configure"
                class="org.sonarlint.intellij.actions.SonarConfigureProject"
                text="Configure SonarLint"
                description="Configure SonarLint"
                icon="SonarLintIcons.TOOLS">
        </action>

    </actions>
```



#### 4. 模块组件



```xml
    <module-components>
        <component>
            <implementation-class>org.sonarlint.intellij.analysis.VirtualFileTestPredicate</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.core.ModuleBindingManager</implementation-class>
        </component>
        <component>
            <implementation-class>org.sonarlint.intellij.config.module.SonarLintModuleSettings</implementation-class>
        </component>
    </module-components>

   
```

#### 5. 扩展

- 配置
- 服务

```xml
 <extensions defaultExtensionNs="com.intellij">
        <codeInsight.linkHandler prefix="#sonarissue/" handlerClass="org.sonarlint.intellij.editor.SonarLinkHandler"/>
        <toolWindow id="SonarLint" anchor="bottom" icon="/images/ico-sonarlint-13.png" factoryClass="org.sonarlint.intellij.ui.SonarLintToolWindowFactory"/>
        <projectConfigurable order="AFTER SonarLintApp" displayName="SonarLint Project Settings" instance="org.sonarlint.intellij.config.project.SonarLintProjectConfigurable"
                             nonDefaultProject="true"/>
        <applicationConfigurable id="SonarLintApp" displayName="SonarLint General Settings" instance="org.sonarlint.intellij.config.global.SonarLintGlobalConfigurable"/>
        <moduleConfigurable id="SonarLint" displayName="SonarLint" instance="org.sonarlint.intellij.config.module.SonarLintModuleConfigurable"/>
        <colorSettingsPage implementation="org.sonarlint.intellij.config.SonarLintColorSettingsPage"/>
        <checkinHandlerFactory implementation="org.sonarlint.intellij.trigger.SonarLintCheckinHandlerFactory"/>
        <additionalTextAttributes scheme="Default" file="colorSchemes/SonarLintDefault.xml"/>
        <additionalTextAttributes scheme="Darcula" file="colorSchemes/SonarLintDarcula.xml"/>
        <editorActionHandler action="EditorEscape" implementationClass="org.sonarlint.intellij.editor.EscapeHandler"/>
        <projectService serviceImplementation="org.sonarlint.intellij.editor.SonarLintHighlighting"/>
        <projectService serviceImplementation="org.sonarlint.intellij.actions.IssuesViewTabOpener"/>
    </extensions>

    <extensionPoints>
        <extensionPoint name="AnalysisConfiguration" interface="org.sonarlint.intellij.analysis.AnalysisConfigurator"/>
    </extensionPoints>
```



# Plugin Extensions and Extension Points

The *IntelliJ Platform* **provides the concept of *extensions* and *extension points*** that <u>allows a plugin to interact with other plugins or with the IDE itself.</u>

## Extension points

**If you want your plugin to allow other plugins to extend its functionality, in the plugin, you must declare one or several *extension points*. Each extension point defines a class or an interface that is allowed to access this point.**

## Extensions

If you want your plugin to extend the functionality of other plugins or the *IntelliJ Platform*, you must declare one or several *extensions*.

## How to declare extensions and extension points

You can declare extensions and extension points in the plugin configuration file `plugin.xml`, within the `<extensions>` and `<extensionPoints>`sections, respectively.

**To declare an extension point**

In the `<extensionPoints>` section, insert a child element `<extensionPoint>` that **defines the extension point name and the name of a bean class or an interface that is allowed to extend the plugin functionality in the `name`, `beanClass` and `interface` attributes, respectively.**

To clarify this procedure, consider the following sample section of the plugin.xml file:

```xml
<extensionPoints>   
  <extensionPoint name="MyExtensionPoint1" beanClass="MyPlugin.MyBeanClass1"> 
  <extensionPoint name="MyExtensionPoint2" interface="MyPlugin.MyInterface"> </extensionPoints> 
```



- The `interface` attribute **sets an interface the plugin that contributes to the extension point must implement**.
- The `beanClass` attribute sets a bean class that specifies one or several properties annotated with the [@Attribute](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/util/src/com/intellij/util/xmlb/annotations/Attribute.java) annotation.

The plugin that contributes to the extension point will read those properties from the `plugin.xml` file.

To clarify this, consider the following sample `MyBeanClass1` bean class used in the above `plugin.xml` file:

```java
public class MyBeanClass1 extends AbstractExtensionPointBean {   
  @Attribute("key")   
  public String key;    
  @Attribute("implementationClass")   
  public String implementationClass;    
  public String getKey() 
  {     
    return key;   
  }    
  public String getClass() {
    return implementationClass;   
  } 
}
```

To declare an extension designed to access the `MyExtensionPoint1` extension point, your `plugin.xml` file must contain the `<MyExtensionPoint1>`tag with the `key` and `implementationClass` attributes set to appropriate values (see sample below).

**To declare an extension**

Auto-completion is available for all these steps.

1. For the

   ```text
   <extensions>
   ```

   element, set the

   ```text
   defaultExtensionNs
   ```

   attribute to one of the following values:

   - `com.intellij`, if your plugin extends the IntelliJ Platform core functionality.
   - `{ID of a plugin}`, if your plugin extends a functionality of another plugin.

2. Add a new child element to the `<extensions>` element. The child element name must match the name of the extension point you want the extension to access.

3. Depending on the type of the extension point, do one of the following:

   - If the extension point was declared using the `interface` attribute, for newly added child element, set the `implementation` attribute to the name of the class that implements the specified interface.
   - If the extension point was declared using the `beanClass` attribute, for newly added child element, set all attributes annotated with the [@Attribute](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/util/src/com/intellij/util/xmlb/annotations/Attribute.java) annotations in the specified bean class.

To clarify this procedure, consider the following sample section of the `plugin.xml` file that defines two extensions designed to access the `appStarter` and `applicationConfigurable` extension points in the *IntelliJ Platform* and one extension to access the `MyExtensionPoint1` extension point in a test plugin:



```xml
<!-- Declare extensions to access extension points in the IntelliJ Platform.
     These extension points have been declared using the "interface" attribute.
 -->
  <extensions defaultExtensionNs="com.intellij">
    <appStarter implementation="MyTestPackage.MyTestExtension1" />
    <applicationConfigurable implementation="MyTestPackage.MyTestExtension2" />
  </extensions>

<!-- Declare extensions to access extension points in a custom plugin
     The MyExtensionPoint1 extension point has been declared using *beanClass* attribute.
-->
  <extensions defaultExtensionNs="MyPluginID">
     <MyExtensionPoint1 key="keyValue" implementationClass="MyTestPackage.MyClassImpl"></MyExtensionPoint1>
  </extensions>
```



### Extension default properties

The following properties are available always:

- `id` - unique ID
- `order` - allows to order all defined extensions using `first`, `last` or `before|after [id]` respectively
- `os` - allows to restrict extension to given OS, e.g., `os="windows"` registers the extension on Windows only

### Extension properties code insight

Several tooling features are available to help configuring bean class extension points in `plugin.xml`.

Property names matching the following list will resolve to FQN:

- `implementation`
- `className`
- `serviceInterface` / `serviceImplementation`
- ending with `Class` (case-sensitive)

A required parent type can be specified in the extension point declaration via nested `<with>`:

```xml
<extensionPoint name="myExtension" beanClass="MyExtensionBean">   
  <with attribute="psiElementClass" implements="com.intellij.psi.PsiElement"/> </extensionPoint> 
```

Property name `language` will automatically resolve to all present `Language` IDs.

Specifying `@org.jetbrains.annotations.Nls` verifies capitalization of UI text properties according to given `capitalization` value (2019.2 and later).



## How to get the extension points list?

To get a list of extension points available in the *IntelliJ Platform* core, consult the `<extensionPoints>` section of the following XML configuration files:

- [`LangExtensionPoints.xml`](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/platform-resources/src/META-INF/LangExtensionPoints.xml)
- [`PlatformExtensionPoints.xml`](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/platform-resources/src/META-INF/PlatformExtensionPoints.xml)
- [`VcsExtensionPoints.xml`](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/platform-resources/src/META-INF/VcsExtensionPoints.xml)

# Plugin Actions

The *IntelliJ Platform* provides the concept of *actions*. An action is a class, derived from the [`AnAction`](https://upsource.jetbrains.com/idea-ce/file/idea-ce-e97504227f5f68c58cd623c8f317a134b6d440b5/platform/editor-ui-api/src/com/intellij/openapi/actionSystem/AnAction.java) class, whose `actionPerformed` method is called when the menu item or toolbar button is selected.

The system of actions allows plugins to add their own items to IDEA menus and toolbars. Actions are organized into groups, which, in turn, can contain other groups. A group of actions can form a toolbar or a menu. Subgroups of the group can form submenus of a menu. You can find detailed information on how to create and register your actions in the [IntelliJ Platform Action System](http://www.jetbrains.org/intellij/sdk/docs/basics/action_system.html).



## 代码结构

![](../img/sonar-2.png)

### 结构分析

- config：
  - global:全局配置，面板展示（qube绑定，规则设置）
  - project：项目文件、目录等exclude配置



























