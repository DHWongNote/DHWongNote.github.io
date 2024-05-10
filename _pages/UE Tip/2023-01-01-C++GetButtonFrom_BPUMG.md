---
title: 【UE Tip】UMG
author: DH Wang
date:  1000-01-01
category: GAME
layout: post
---


# 大致流程
1、创建UMG控制器（继承自UserWidget的C++父类，因为需要控制新建的子类UMG对象，一般是不修改源码，通过新建项进行拓展编辑） //TestUI
2、创建UMG对象（继承上面创建的父类），界面一般是通过蓝图类来编写（因为容易编写界面，但是很多公司不允许在该蓝图类中使用节点，因为不方便代码管理，这里只是通过蓝图来作为编写UI界面的桥梁）
3、显示UMG对象，可以通过PlayController 或者LevelBluePrint来搭载UI显示//Level BluePrint在ToolBar->BluePrint->LevelBluePrint可以找到
4、绑定注册按钮，因为UMG对象继承自父类控制器所以这里可以根据按钮名字通过虚幻内部反射建立引用关系。

父类.h
``` 
#pragma once 
#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "TestUI.generated.h"
  
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnButtonClickedEventMantra);
UCLASS()
class UNREAL_PLAYGROUND_API UTestUI : public UUserWidget
{
	GENERATED_BODY()
public: 
	virtual void NativeConstruct() override;
	UPROPERTY(Meta = (BindWidget))
	class UButton* Btn_Test;
	UFUNCTION()
		void OnBtn_Test();
};
```
父类.cpp
```
#include "TestUI.h"  
#include "Components/ContentWidget.h"
#include "Components/Button.h" 
void UTestUI::NativeConstruct()
{
	Super::NativeConstruct();
	if (Btn_Test != nullptr)
	{
		// 绑定按钮点击事件
		Btn_Test->OnClicked.AddDynamic(this, &UTestUI::OnBtn_Test);
	}

	UE_LOG(LogTemp, Warning, TEXT("NativeConstruct"));
}

void UTestUI::OnBtn_Test()
{
	UE_LOG(LogTemp, Warning, TEXT("btn clicked"));
}
```
UMGBP
```
图标编辑器->Put Button->Rename Button->Complie
```

LevelBP
```
Beginplay->CreateWidgetBluePrintWidget->AddToViewport


//也可以使用代码进行绑定绘制操作
#include "Blueprint/UserWidget.h" 
UUserWidget* CurrentWidget = CreateWidget<UUserWidget>(GetWorld(), NewWidgetClass); // NewWidgetClass is TSubclassOf<UUserWidget>
CurrentWidget->AddToViewport();
CurrentWidget->RemoveFromParent();// CurrentWidget->RemoveFromViewport();
```



# 注意

反射是通过名称进行查找的，所以绑定时需要注意命名