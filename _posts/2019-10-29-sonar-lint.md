---
title: Sonar Lint idea code
subtitle: Sonar lint idea 源码分析
tags: [idea,plugin]
layout: post
---

sonar lint 插件源码分析。



## SonarLint介绍

SonarLint is an IDE extension that helps you detect and fix quality issues as you write code. Like a spell checker, SonarLint squiggles flaws so they can be fixed before committing code.



## 项目结构

![](../img/sonarlint-project.png)

`plugin.xml` :插件的全局配置

#### 结构分析

#### 1. 应用组件

- SonarLintGlobalSettings
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
- SonarLintSubmitter
- AnalysisResultIssues
- ProjectBindingManager
- LiveIssueCache
- IssuePersistence
- SonarLintConsole
- SonarLintProjectSettings
- SonarLintProjectState
- SonarLintJobManager
- IssueMatcher
- IssueProcessor
- SonarLintAnalyzer
- SonarLintProjectNotifications
- ServerIssueUpdater
- UpdateChecker
- SonarLintTaskFactory
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





## 代码结构

![](../img/sonar-2.png)

### 结构分析

- config：
  - global:全局配置，面板展示（qube绑定，规则设置）
  - project：项目文件、目录等exclude配置