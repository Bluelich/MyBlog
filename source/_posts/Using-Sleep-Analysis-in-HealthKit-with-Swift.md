---
title: 使用Swift实现基于HealthKit的睡眠分析功能
subtitle: healthKit+swift=sleep_analysis
date: 2016-07-18 21:03:52
tags:
---

原文:[链接][1]
译者:[Bluelich][2]

如今,睡眠分析的彻底改变已经成为一种趋势。用户比以往更加好奇，他们不仅希望知道自己的睡眠时间，比如说什么时候开始进入睡眠等，他们还想要通过获得聚合数据来了解自己的睡眠趋势。而今，硬件和手机的技术革新，给这个正在日益增长的用户群体带来了新的曙光。

Apple提供了一个非常酷的方式，让你可以以非常安全的方式，通过内置的`健康`应用和用户的健康信息进行交互。你不仅仅可以通过使用`HealthKit`来[构建一个健身App][3]，该框架还允许你对用户睡眠数据进行分析。

在这个教程里，我将会对`HealthKit`进行一个简单介绍，并演示如何构建一个简单的睡眠分析App。<!--More-->

## 介绍

HealthKit框架提供了一个叫做`HealthKit store`的加密数据库结构来保存数据。你可以通过`HKHealthStore`这个类来访问这个数据库。 iPhone和Apple Watch都有他们自己的`HealthKit Store`。 健康数据会在iPhone和Apple Watch上进行同步；需要注意的是在Apple Watch上，一旦可用空间不足，旧的数据就会被删除掉；另外`HealthKit`在iPad上无法使用。

如果你想创建一个基于健康数据的iOS或watchOS应用，HealthKit将会是一个非常强大的工具。 
它被设计为一个管理各个来源的健康数据的工具，根据用户的偏好设置，将这些数据进行聚合。这些基于`HealthKit`的App拥有在`健康`App中各自数据的读写访问权限，还可以将各自的数据进行合并。这些数据不仅包括用户身体状况的基本数据，健身信息，营养状况，还包括用户的睡眠分析数据。

本文的其余部分,我将向您展示如何在iOS上利用`HealthKit`框架读写睡眠分析数据。 同样的方法也适用于watchOS应用程序。 请注意,本教程编写使用Swift 2.0 和 Xcode 7。所以确保你使用的也是Xcode 7，以便继续下面的教程。

在进行下一步之前，你可以先下载这个[启动项目][4]然后解压。这是一个拥有基本功能的App。运行这个项目后，你会看到一个显示时间的计时器UI和一个开始按钮。

## 使用HealthKit框架

我们要实现的效果是，通过点击`Start`和`Stop`按钮来保存和查询用户的数据。要使用`HealthKit`，必须先让你的App获取到`HealthKit`权限，在工程中选中当前项目Target，然后选择Capabilities,打开`HealthKit`的开关
![HealthKit-allow][image-1]
接下来，你讲需要在`ViewController`中创建一个`HKHealthStore`的对象。
代码如下：
`let healthStore = HKHealthStore()`
后面我们就要使用这个对象`healthStore`来访问`HealthKit store`了

就像前面说的那样，HealthKit给予用户权限来掌控自己的健康数据，因此你在对用户的睡眠数据进行分析之前，需要先获得用户的许可。要获得许可，需要先引入`HealthKit`framework，然后更新`viewDidLoad`里的代码。

代码如下：
```swift
	override func viewDidLoad() {
	    super.viewDidLoad()
	    let typestoRead = Set([
	        HKObjectType.categoryTypeForIdentifier(HKCategoryTypeIdentifierSleepAnalysis)!
	        ])
	    let typestoShare = Set([
	        HKObjectType.categoryTypeForIdentifier(HKCategoryTypeIdentifierSleepAnalysis)!
	        ])
	    self.healthStore.requestAuthorizationToShareTypes(typestoShare, readTypes: typestoRead) { (success, error) -> Void in
	        if success == false {
	            NSLog("请求权限失败")
	        }
	    }
	}
```

这段代码将提示用户`Allow`或`Dont Allow`你的权限请求。 通过这个completion block,您可以处理成功或错误，并得到最终结果。 让用户授予App所有请求的权限并不是必要的，所以你必须优雅地处理应用程序中的错误。

但对于测试的目的,你必须选择“Allow”选项给予允许应用程序访问设备的健康数据。

![Health-App-Permission][image-2]



## 写入睡眠分析的数据

首先,我们如何检索睡眠分析数据? 根据Apple的文档,每个睡眠分析样本只能有一个值。 `HealthKit`使用两个或更多的样本的叠加来代表用户在床上和睡眠中的状态。 通过比较这些样本的开始和结束时间,应用程序可以二次统计:

- 用户进入睡眠所用的时间
- 实际睡觉的时间对比在床上的时间的百分比
- 用户在床上醒来的次数
- 进入睡眠和在床上的时间之和

![record\_sleep\_data][image-3]

简而言之,你需要遵循以下方法来吧睡眠分析数据保存到`HealthKit store`中:

1. 我们需要定义2个 `NSDate`  对象，分别对应起始时间和结束时间。
2. 然后用`HKCategoryTypeIdentifierSleepAnalysis`(这是一个`Enum`)创建一个 `HKObjectType`对象 .
3. 我们需要创建一个新的`HKCategorySample`的对象,因为我们需要用这个对象来记录睡眠数据。个人样本代表了用户在床上或者睡着了的时间周期。因此我们要创建2个样本，分别是在床上的样本和睡着了的样本的时间
4. 最后, 我们使用`HKHealthStore`的 `saveObject` 这个类方法，保存数据.

**编者注**: 对于`HKCategorySample`的类型,可以查看 [HealthKit Constants Reference][5]。



下面我们用Swift来实现上面的4个步骤,保存用户的睡眠数据。 请讲该代码片段放到`ViewController`类中。

代码如下：

```swift
func saveSleepAnalysis() {
	// alarmTime 和 endTime 都是 NSDate 对象
	if let sleepType = HKObjectType.categoryTypeForIdentifier(HKCategoryTypeIdentifierSleepAnalysis) {
	
	    // 根据我们想要做的事情,选择相应的类型和起止时间，构造出一个新的HKCategorySample对象，我们将通过这个对象和系统的Health应用进行交互
	    let object = HKCategorySample(type:sleepType, value: HKCategoryValueSleepAnalysis.InBed.rawValue, startDate: self.alarmTime, endDate: self.endTime)
	
	    // 然后，保存下来
	    healthStore.saveObject(object, withCompletion: { (success, error) -> Void in
	
	        if error != nil {
	            // 这里可以处理下error
	            return
	        }
	
	        if success {
	            print("数据已经保存到Health App中了")
	
	        } else {
	            // 处理其他异常
	        }
	
	    })
	
	    let object2 = HKCategorySample(type:sleepType, value: HKCategoryValueSleepAnalysis.Asleep.rawValue, startDate: self.alarmTime, endDate: self.endTime)
	
	    healthStore.saveObject(object2, withCompletion: { (success, error) -> Void in
	        if error != nil {
	            // 这里可以处理下error
	            return
	        }
	
	        if success {
	            print("数据已经保存到Health App中了")
	        } else {
	            // 处理其他异常
	        }
	
	    })
	
	}
}
```

当我们想要保存我们自己App的睡眠数据到HealthKit的时候，可以调用这个方法。

## 读取睡眠分析数据

要读取睡眠分析数据,我们需要创建一个`HKSampleQuery`来进行查询。 通过指定`HKCategoryTypeIdentifierSleepAnalysis`来创建一个`HKObjectType` 的对象 。 您可能还希望使用谓词来过滤获取到的数据，你可以通过指定`startDate` 和`endDate` 来确定你要查询的时间范围。 你可能还想要创建一个sortDescriptor 来对最终结果进行排序。

获取睡眠分析结果数据的代码如下：

```swift
func retrieveSleepAnalysis() {
	// 首先，通过构造一个HKObjectType，来指定我们要查询的类型
	if let sleepType = HKObjectType.categoryTypeForIdentifier(HKCategoryTypeIdentifierSleepAnalysis) {
	
	    // 使用sortDescriptor来获取到最新的数据
	    let sortDescriptor = NSSortDescriptor(key: HKSampleSortIdentifierEndDate, ascending: false)
	
	    // 创建一次查询，在下一步执行查询后，会回调这个构造函数的block，我们可以通过这个回调来对获取的结果进行处理
	    let query = HKSampleQuery(sampleType: sleepType, predicate: nil, limit: 30, sortDescriptors: [sortDescriptor]) { (query, tmpResult, error) -> Void in
	
	        if error != nil {
	
	            // 这里可以处理下error
	            return
	
	        }
	
	        if let result = tmpResult {
	
	            // 处理数据
	            for item in result {
	                if let sample = item as? HKCategorySample {
	                    let value = (sample.value == HKCategoryValueSleepAnalysis.InBed.rawValue) ? "InBed" : "Asleep"
	                    print("Healthkit sleep: \(sample.startDate) \(sample.endDate) - value: \(value)")
	                }
	            }
	        }
	    }
	
	    // 最后执行查询
	    healthStore.executeQuery(query)
	}
}
```

这段代码做的事情是：指定期望为降序排列，然后向`HealthKit`查询所有的睡眠数据。然后每个查询结果都会打印(开始时间，结束时间，睡眠状态)。在构造查询对象时，通过设置`limit: 30`来指定需要查询的条数为30条，因为前面的期望为降序排列，所以是最近的30条记录，你可以通过指定`predicate`来限定你想要获取记录的开始和结束时间。



## App测试

在这个demo中，当你点击`Start`按钮的时候，我使用了NSTimer来刷新Label的显示，以表示时间的累加。当你点击`Start`和`Stop`按钮的时候，会分别创建一个`NSDate`对象来保存当前时间。当你点击`Stop`按钮的时候，会计算2个时间的时间差，然后根据这个时间，保存用户的睡眠数据，在`func stop(sender: AnyObject)`中，你可以调用`saveSleepAnalysis()` 和 `retrieveSleepAnalysis()` 方法来保存和获取用户的睡眠数据。

```swift
@IBAction func stop(sender: AnyObject) {
	endTime = NSDate()
	saveSleepAnalysis()
	retrieveSleepAnalysis()
	timer.invalidate()
}
```

在你的App中，你可能想要修改NSDate对象来选择相关的开始和结束时间(可能是不同的)来保存在床上和睡着了的状态下的数据。

一旦你做出了更改，你就可以运行这个demo，接着启动timer。让app运行几分钟，然后点击`Stop`按钮。然后，打开`Health`应用，你会发现你的App的睡眠数据已经保存在里面了。

![sleep-analysis-test][image-4]



## 给HealthKit Apps的一些建议

HealthKit旨在给开发者提供一个公共的平台，用于非常便利地访问和共享用户的健康数据，并且避免任何可能状况下的重复或者异常数据。Apple的审核指南非常明确地指出，如果你App使用了HealthKit来向用户请求读写健康数据的权限，但是不能给出明确的用途的话，你的App是会被拒的。

保存虚假数据或者不正确的数据到Helath应用的App也会被拒。这意味着，你不能轻信你的App中那些计算健康数据的算法(比如在这个教程中的睡眠分析)。你应该尝试使用内置的传感器数据来读取和操作任何参数，以避免计算出错误的数据。

在[这里][6]，你可以下载到这个教程对应的完整项目。

[1]:	http://www.appcoda.com/sleep-analysis-healthkit/
[2]:	http://weibo.com/u/1376767097
[3]:	https://www.appcoda.com/healthkit-introduction/
[4]:	https://github.com/appcoda/SleepAnalysis/blob/master/SleepAnalysisStarter.zip?raw=true
[5]:	https://developer.apple.com/library/ios/documentation/HealthKit/Reference/HealthKit_Constants/index.html#//apple_ref/doc/uid/TP40014710
[6]:	https://github.com/appcoda/SleepAnalysis

[image-1]:	https://www.appcoda.com/wp-content/uploads/2016/05/HealthKit-allow-1024x640.png
[image-2]:	https://www.appcoda.com/wp-content/uploads/2016/05/Health-App-Permission.png
[image-3]:	https://www.appcoda.com/wp-content/uploads/2016/05/record_sleep_data-1024x525.png
[image-4]:	https://www.appcoda.com/wp-content/uploads/2016/06/sleep-analysis-test-1024x725.png

