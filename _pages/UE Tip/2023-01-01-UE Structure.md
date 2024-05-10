---
title: 【UE Tip】Structure
author: DH Wang
date:  1000-01-01
category: GAME
layout: post
---

# 模块结构
工程初始模块（默认模块）如需添加其他模块需要修改ProName.uproject中的模块引入
```
{
	"FileVersion": 3,
	"EngineAssociation": "4.27",
	"Category": "",
	"Description": "",
	"Modules": [
		{
			"Name": "Unreal_Playground",
			"Type": "Runtime",
			"LoadingPhase": "Default",
			"AdditionalDependencies": [
				"Engine",
				"UMG"
			]
		}
        //默认工程C++文件夹只有一个即项目名，添加如下模块即多了个文件夹BlogpostModule
        ///
        //BlogpostModule结构：如下下
        // private
        // public
        // BlogpostModule.Build.cs
        ///
        ,
		{
			"Name": "BlogpostModule",
			"Type": "Runtime",
			"LoadingPhase": "Default"
		}
	],
```

同理插件管理也是一样，如需在ue中新建插件子文件夹，则需添加子模块。每个模块都是由如下构成：

//private
内部文件，同默认工程文件

//public
内部文件，同默认工程文件

//ModuleName.Build.cs
```
using UnrealBuildTool; 
public class ModuleName : ModuleRules
{
	public ModuleName(ReadOnlyTargetRules Target) : base(Target)
	{
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

		//Public module names that this module uses.
		//In case you would like to add various classes that you're going to use in your game
		//you should add the core,coreuobject and engine dependencies.
		PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine","PropertyEditor","Slate","SlateCore"});
 
		//The path for the header files
		PublicIncludePaths.AddRange(new string[] {"ModuleName/Public"});
 
		//The path for the source files
		PrivateIncludePaths.AddRange(new string[] {"ModuleName/Private"});
	}
}
```
示例可参考自定义资产并修改细节面板的完整插件


# GamePlay?
在介绍GamePlay的时候，更多的重点是在于介绍各对象的职责和关联，所以更多是用类图来描述结构，反而对源码进行剖析的机会不多

# GEngine
```
GEngine->GetSmallFont()
```

# UObject

# UGameEngine
## UGameInstance
### FWorldContext
## UWorld

# AGameModeBase
## AGameMode


## ULevel


## AActor
### AInfo
#### AGameState
#### AWorldSettings
#### APlayerState 




## UActorComponent
### USceneComponent
#### UPrimitiveComponent




# UExporter

# FColor
