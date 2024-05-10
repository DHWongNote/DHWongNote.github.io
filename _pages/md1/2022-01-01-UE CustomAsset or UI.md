---
title: 《UE自定义资产》
author: DH Wang
date: 1000-01-01
category: TA
layout: post
---

# 简称
Action是指FAssetTypeActions_Base类，下面简称Action，Factory类是UFactory也是简称同上。

# 自定义资源
1、所有资产类型都继承自UObject,一部分是类似UFactory等在Editortime来编辑器这个UObject数据的代码，这其实很好理解。UFactory等代码会在游戏打包后丢掉。另一部分是继承FAssetTypeActions_Base来编辑资产的常规操作和注册模块或获取一些资产原始属性如显示图标等。
2、还有一些其他的可进行拓展如定义该资源的导入导出（需要继承UExporter重新编译引擎），拓展该资源的Detail显示面板（包括UI布局需要继承IDetailCustomization，或一些拖拽事件的逻辑操作如修改某元素自动计算新的UI元素值等可继承FAssetEditorToolkit）。
3、关于资源的分类可在Factory和Action中重载，都重写分类函数优先级是Factory大于Action.新资源默认是Miscellaneous

```
//自定义类型
UCLASS(BlueprintType)
class EXTRAASSETS_API 【NewAsset】 : public UObject
{
	GENERATED_BODY()

public:
	UPROPERTY(EditAnywhere)
	FString myString; 
};

//Factory 
UCLASS(config = Editor)
class UYaksueTestAssetFactory : public UFactory
{
    GENERATED_UCLASS_BODY()
public: 
	/* New assets that don't override this function are automatically placed into the "Miscellaneous" category in the editor */
	//virtual uint32 GetMenuCategories() const override;

	virtual UObject* FactoryCreateNew(UClass* Class, UObject* InParent, FName Name, EObjectFlags Flags, UObject* Context, FFeedbackContext* Warn) override;
	 
};

 
class 【NewAsset】TypeActions : public FAssetTypeActions_Base
{
public:
	FYaksueTestAssetTypeActions(); 
	// Asset的名字
	virtual FText GetName() const override { return FText::FromName(TEXT("【NewAsset】Name")); }
	// Asset图标的颜色
	virtual FColor GetTypeColor() const override { return FColor(0, 255, 255); }
	// Asset的UObject是什么
	virtual UClass *GetSupportedClass() const override { return 【NewAsset】::StaticClass(); }
	// Asset所属的分类
	virtual uint32 GetCategories() override;

	virtual void OpenAssetEditor(const TArray<UObject *> &InObjects, TSharedPtr<class IToolkitHost> EditWithinLevelEditor = TSharedPtr<IToolkitHost>()) override;
private:
	EAssetTypeCategories::Type AssetCategory;
};
```


# 自定义资产面板

如果仅定义默认界面的.uasset仅需Factory和Action可以使用FSimpleAssetEditor::CreateEditor直接创建默认的Editor。
```
void 【NewAsset】TypeActions::OpenAssetEditor(const TArray<UObject*>& InObjects, TSharedPtr<class IToolkitHost> EditWithinLevelEditor)
{
	FSimpleAssetEditor::CreateEditor(EToolkitMode::Standalone, EditWithinLevelEditor, InObjects);
}
```


1、如果需要更细致的封装和编辑可以利用 FAssetEditorToolkit和 IDetailCustomization来进行UI和编辑器窗口的一些操作和布局。IDetailCustomization和FAssetEditorToolkit都可以对显示面板进行绘制重写。区别就是Toolkit类是对自定义资产进行绘制，而IDetailCustomization是可以对整个窗口进行完全控制绘制（可以新开分类绘制提示Lable或者绘制图标等）

2、正常开发中涉及复杂的工具链或者插件或者引擎资产（如Actor类）需要更详细的面板编辑需要拓展Detail,而自定义资产面板则先要使用toolkit类完成资源定义（也可以在定义时完成简单的样式编辑）。 

3、总的来说自定义资产需要在重写Action类的OpenAssetEditor函数中完成初始化操作，并在重载函数RegisterTabSpawners中完成样式的编辑。

# 自定义资源导入导出
自定义资源的导出需要在当前模块中继承UExporter并实现自定义资源的格式操作。

* 参考UTextureExporterTGA
```
UCLASS()
class UNewAssetExporter : public UExporter
{
GENERATED_UCLASS_BODY()

virtual bool SupportsObject(UObject* Object) const override;
virtual bool ExportBinary(UObject* Object, const TCHAR* Type, FArchive& Ar, FFeedbackContext* Warn, int32 FileIndex = 0, uint32 PortFlags = 0) override;
};
// Fill out your copyright notice in the Description page of Project Settings.


#include "NewAssetExporter.h"
#include "NewAsset.h" 
UNewAssetExporter::UNewAssetExporter(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
{
	SupportedClass = UNewAsset::StaticClass();
	PreferredFormatIndex = 0;
	FormatExtension.Add(TEXT("myfile"));
	FormatDescription.Add(TEXT("myfile file"));
} 
bool UNewAssetExporter::SupportsObject(UObject* Object) const
{
	bool bSupportsObject = false;
	if (Super::SupportsObject(Object))
	{
		bSupportsObject = IsValid(Cast<UNewAsset>(Object));
	}
	return bSupportsObject;
} 
bool UNewAssetExporter::ExportBinary(UObject* Object, const TCHAR* Type, FArchive& Ar, FFeedbackContext* Warn, int32 FileIndex, uint32 PortFlags)
{
	UNewAsset* Asset = CastChecked<UNewAsset>(Object);
	if (!IsValid(Asset))
	{
		return false;
	}
	Ar << Asset->IntValue;
	return true;
}

```





# Actor类型细节面板
UE内部默认提供的资产类型则不需要绑定太多操作，只需要在模块注册时在重写StartupModule中调用 RegisterCustomClassLayout 分配Actor参数。
无论是对插件里的Actor还是工程初始模块里的Actor进行重写UI都需要将目标对象（继承自IDetailCustomization的类，具体UI排布和一些事件逻辑都可在里面实现）在加载了的模块中注册分配Layout(通过RegisterCustomClassLayout函数)。

### 总结
总的来说和unity里通过Typeof(Target)扩展Editor也差不多，都是需要一个新的类来写新的UI布局或者逻辑操作（unity里是继承自Editor,这里是通过IDetailCustomization），多的步骤就是ue里需要自己手动实现新Layout的注册（在模块初始化里进行，默认继承AGameModeBase的初始模块没有重写 StartupModule，所以还需要新建模块并在.uproject中添加过后，再通过重写StartupModule实现Layout注册）。

