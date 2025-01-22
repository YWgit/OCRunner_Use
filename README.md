>之前公司项目使用的是JSPatch老的版本做的热更(最新的要收费)，每次发布补丁的时候都要做改动方法或者类的OC代码转成JS代码的修正，头有点大，改个小bug调试起来也比较麻烦。于是想着能不能不通过JSCore下发的方式做热更呢，去github上面搜了一波，无意中找到了基于二进制热更的方案，配置好后不用再做代码的转换和翻译，直接生成二进制文件，当补丁用就把bug给修复了。

####先放下二进制热更的原理图
![二进制热更原理图.png](https://upload-images.jianshu.io/upload_images/1984312-c9225f34cecf273b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####经过实践，我在本地和github下载的zip包都实现了代码更新的效果，新手接入的时候有几个注意的地方：
1.将OCRunner Pod下来
```
pod 'OCRunner'      #支持所有架构，包含libffi.a
# 或者
pod 'OCRunnerArm64' #仅支持 arm64和arm64e，没有libffi.a
```
2.在项目中添加需要热更的文件夹HotPath，把热更的资源放到这个文件夹下。在demo中我将这个库弄成一个模块OCRunnerLib，以AppDelegate+OCRunner分类的形式引入了，方便管理。

![添加脚本.png](https://upload-images.jianshu.io/upload_images/1984312-2714d2814fbcb6f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

3.添加HotPath目录下资源生成binarypatch二进制文件和jsonpatch的脚本，对应改下项目名，并添加存放内联函数、预编译函数、C函数的转义，否则会出现不支持OC的一些属性设置和系统函数方法的Scripts.bundle，具体可以看下GCDRefrences和UIKitRefrences，对应自己项目用到的少哪个加哪个就行。
###PatchGeneratorBinary
```
$SRCROOT/OCRunner_Use/PatchGenerator -files $SRCROOT/OCRunner_Use/HotPath -refs  $SRCROOT/OCRunner_Use/Scripts.bundle -output $SRCROOT/OCRunner_Use/binarypatch
```
###PatchGeneratorJson
```
$SRCROOT/OCRunner_Use/PatchGenerator -files $SRCROOT/OCRunner_Use/HotPath -refs  $SRCROOT/OCRunner_Use/Scripts.bundle -output $SRCROOT/OCRunner_Use/jsonpatch -type json
```
![添加脚本.png](https://upload-images.jianshu.io/upload_images/1984312-31815f98c059bf5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



4.对项目热更的VC做改动，想改哪个类就把类名设置一样即可，这里作者没有说明，我也是自己摸索好长时间才知道规律的。想改哪个方法重写就可以了，可新增类，也可修改原有类。```注意，新增类要把这个.m文件加到build phases->compile sources中，否则识别不到类中的方法出现闪退的情况，而修改的类不用，否则编译器会报错。```

![新增的类.png](https://upload-images.jianshu.io/upload_images/1984312-299cd426992ffa91.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5.build成功后把binarypatch二进制上传服务器，app加载二进制热更。以下是我在demo中做的本地和服务器(我没有服务器就用上传github的zip下载做了)。测试一波完成了HotPath改动的更新，不用再做代码的转换，对coder更友好了。
```
- (void)loadOCRunner {
    [ORSystemFunctionPointerTable reg:@"CGPointEqualToPoint" pointer:&CGPointEqualToPoint];
    [ORSystemFunctionPointerTable reg:@"CGSizeEqualToSize" pointer:&CGSizeEqualToSize];
#if 0
    NSString *finalPath = [[NSBundle mainBundle] pathForResource:@"binarypatch" ofType:nil];
    [ORInterpreter excuteBinaryPatchFile:finalPath];
#else
    //若补丁已经下载好并解压了，就读取本地的补丁，避免每次都去下载新的补丁
    NSURL *documentsDirectoryURL = [[NSFileManager defaultManager] URLForDirectory:NSDocumentDirectory inDomain:NSUserDomainMask appropriateForURL:nil create:NO error:nil];
    NSString *finalPath = [[documentsDirectoryURL.absoluteString stringByAppendingPathComponent:@"binarypatch"] stringByReplacingOccurrencesOfString:@"file:" withString:@""];
    if ([[NSFileManager defaultManager] fileExistsAtPath:finalPath]) {
        [ORInterpreter excuteBinaryPatchFile:finalPath];
        return;
    }
    //创建信号量，保证先把补丁包下载下来再进入主页面，若补丁包大的话，可以考虑不阻塞主线程，给予等待loding显示
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    AFURLSessionManager *manager = [[AFURLSessionManager alloc] initWithSessionConfiguration:configuration];
    manager.completionQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    NSURL *URL = [NSURL URLWithString:@"https://github.com/Charles2016/OCRunner_Use/raw/master/OCRunner_Use/OCRunnerLib/ConfigPath/binarypatch_Use.zip"];
    NSURLRequest *request = [NSURLRequest requestWithURL:URL];
    NSURLSessionDownloadTask *downloadTask = [manager downloadTaskWithRequest:request progress:nil destination:^NSURL *(NSURL *targetPath, NSURLResponse *response) {
        return [documentsDirectoryURL URLByAppendingPathComponent:[response suggestedFilename]];
    } completionHandler:^(NSURLResponse *response, NSURL *filePath, NSError *error) {
        NSLog(@"File downloaded to: %@", filePath);
        NSString *filePathStr = filePath.relativePath;
        NSString *toPath = [filePathStr stringByDeletingLastPathComponent];
        NSString *finalPath = [NSString stringWithFormat:@"%@/binarypatch", toPath];
        //先移除已存在的解压包，以防解压过后的文件干扰到，取不到最新的二进制文件
        NSFileManager *fileManager = [NSFileManager defaultManager];
        if ([fileManager fileExistsAtPath:finalPath]) {
            [fileManager removeItemAtPath:finalPath error:nil];
        }
        [SSZipArchive unzipFileAtPath:filePathStr toDestination:toPath progressHandler:nil completionHandler:^(NSString *path, BOOL succeeded, NSError *error) {
           NSLog(@"File unzip to: %@", finalPath);
            [ORInterpreter excuteBinaryPatchFile:finalPath];
            dispatch_semaphore_signal(semaphore);
        }];
    }];
    [downloadTask resume];
    //信号量等待
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
#endif
}
```

以上就是二进制热更的实践了，测试起来感觉还不错。服务器下载zip的话可以与后台协商，觉得可以弄个灰度下发之类的操作，版本号相关配置判断，想让哪台手机哪个版本更新都可以了，看作者说明是可以过App Store的审核，没有具体上传过，需要进一步验证，希望能帮到你。[本篇GitHubDemo](https://github.com/Charles2016/OCRunner_Use)传送阵在此。
感谢作者[@SilverFruity/OCRunner](https://github.com/SilverFruity/OCRunner)的开源封装。




**OCRunner QQ群: 860147790**

[中文介绍](https://github.com/SilverFruity/OCRunner/blob/master/README-CN.md)

[相关文章](https://github.com/SilverFruity/OCRunner/issues/11)

[Wiki](https://github.com/SilverFruity/OCRunner/wiki)

## Introduction

### The work flow of using [OCRunner](https://github.com/SilverFruity/OCRunner) to generate a patch 

![image](https://raw.githubusercontent.com/SilverFruity/silverfruity.github.io/9a371dcb9cece8deefa4fe05b155ae7cbd5834b5/source/_posts/OCRunner/OCRunner_EN_0.jpeg)

### Responsibilities of all parties

* [oc2mangoLib](https://github.com/SilverFruity/oc2mango/tree/master/oc2mangoLib) is equivalent to a simple compiler, responsible for generating the abstract syntax tree.
* [ORPatchFile](https://github.com/SilverFruity/oc2mango/tree/master/oc2mangoLib/PatchFile) is responsible for serializing and deserializing the abstract syntax tree and determining whether the version is available.
* [PatchGenerator](https://github.com/SilverFruity/oc2mango/tree/master/PatchGenerator) is responsible for integrating the functions of oc2mangoLib and ORPatchFile. (All the above tools are in the [oc2mango](https://github.com/SilverFruity/oc2mango) project).
* [OCRunner](https://github.com/SilverFruity/OCRunner) is responsible for executing the abstract syntax tree.

### Difference from other hotfix libraries

* Use binary patch files. Increase security, reduce patch size, optimize startup time, and can be optimized in the PatchGenerator stage.

* Custom Arm64 ABI （You can also choose to use libffi）

* Complete Objective-C syntax support, but does not support pre-compilation and partial syntax.

## Run patches locally using OCRunnerDemo
[OCRunnerDemo](https://github.com/SilverFruity/OCRunner/tree/master/OCRunnerDemo) can be used as a reference for the entire process.

You can't run it successlly with downloading zip file. You must using the below shell commands to tour OCRunnerDemo.
```
git clone --recursive https://github.com/SilverFruity/OCRunner.git
```

###  Cocoapods

```ruby
pod 'OCRunner'      #Support all architectures, including libffi.a
# or
pod 'OCRunnerArm64' #Only supports arm64 and arm64e, does not include libffi.a
```

### Download [PatchGenerator](https://github.com/SilverFruity/oc2mango/releases)

Unzip PatchGenerato.zip, then save **PatchGenerator** to /usr/local/bin/ or the project directory.

### add `Run Script`  of PatchGenerator 

1. **Project Setting** -> **Build Phases** -> click  `+`  in the upper left corner -> `New Run Script Phase`

2. [Path to PatchGenerator file]  **-files** [Objective-C source files or diretory] **-refs** [Objective-C header files or diretory]  **-output**  [Path to save the patch]

3. for example:  `Run Script` in OCRunnerDemo

   ```shell
   $SRCROOT/OCRunnerDemo/PatchGenerator -files $SRCROOT/OCRunnerDemo/ViewController1 -refs  $SRCROOT/OCRunnerDemo/Scripts.bundle -output $SRCROOT/OCRunnerDemo/binarypatch
   ```

### Development environment:  Execute patch file

1. Add the generated patch file as a resource file to the project.

2. Appdelegate.m

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
#if DEBUG
    NSString *patchFilePath = [[NSBundle mainBundle] pathForResource:@"PatchFileName" ofType:nil];
#else
   // download from server
#endif
    [ORInterpreter excuteBinaryPatchFile:patchFilePath];
    return YES;
}
```

3. Every time you modify the file, remember to use **Command+B**, call `Run Scrip` to regenerate the patch file.



### Online environment

1. Upload the patch to the resource server.
2. Download and save the patch file in the App.
3. Use **[ORInterpreter excuteBinaryPatchFile:PatchFilePath]** to execute the patch.



## Use introduction



### Structure, Enum, Typedef

You can run the following code by modifying **ViewController1** in **OCRunnerDemo**.

```objc
// A new type called dispatch_once_t will be added
typedef NSInteger dispatch_once_t;
// link NSLog
void NSLog(NSString *format, ...);

typedef enum: NSUInteger{
    UIControlEventTouchDown                                         = 1 <<  0,
    UIControlEventTouchDownRepeat                                   = 1 <<  1,
    UIControlEventTouchDragInside                                   = 1 <<  2,
    UIControlEventTouchDragOutside                                  = 1 <<  3,
    UIControlEventTouchDragEnter                                    = 1 <<  4
}UIControlEvents;

int main(){
    UIControlEvents events = UIControlEventTouchDown | UIControlEventTouchDownRepeat;
    if (events & UIControlEventTouchDown){
        NSLog(@"UIControlEventTouchDown");
    }
    NSLog(@"enum test: %lu",events);
    return events;
}
main();
```

**Tips:** 

It is recommended to create a new file to place the above code, similar to the UIKitRefrence and GCDRefrence files in OCRunnerDemo, and then add the patch generation in the form of **-links**.



### Use system built-in C functions

```objc
//you only need to add the C function declaration in Script.
//link NSLog
void NSLog(NSString *format, ...);

//then you can use it in Scrtips.
NSLog(@"test for link function %@", @"xixi");
```

You can run the code by changing the content of **ViewController1** in OCRunnerDemo.

When you add this code in scripts. OCRunner will use `ORSearchedFunction` to search the pointer of function name. It's core is `SymbolSearch` (edit from fishhook).

If the searched result of function name is NULL，OCRunner will notice you in console like this:

```objc
|----------------------------------------------|
|❕you need add ⬇️ code in the application file |
|----------------------------------------------|
[ORSystemFunctionTable reg:@"dispatch_source_set_timer" pointer:&dispatch_source_set_timer];
```

### Fix Objective-C 's object (class) method and add attributes

If you want to fix a method, you can reimplement the method without implementing other methods.


```objc
@interface ORTestClassProperty:NSObject
@property (nonatomic,copy)NSString *strTypeProperty;
@property (nonatomic,weak)id weakObjectProperty;
@end
@implementation ORTestClassProperty
- (void)otherMethod{
    self.strTypeProperty = @"Mango";
}
- (NSString *)testObjectPropertyTest{
  	[self ORGtestObjectPropertyTest] // Add'ORG' before the method name to call the original method
    [self otherMethod];
    return self.strTypeProperty;
}
@end
```



### Use of Block and solve circular references

```objc
__weak id object = [NSObject new];
// Minimal block
void (^a)(void) = ^{
    int b = 0;
};
a();
```



### Use GCD

Its essence is **Use system built-in C functions**. It is added through the **GCDRefrences** file in OCRunnerDemo. The GCD related function declaration and typedef are all included in it.

For Example:

```objc
// link dispatch_sync
void dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);
void main(){
  dispatch_queue_t queue = dispatch_queue_create("com.plliang19.mango",DISPATCH_QUEUE_SERIAL);
	dispatch_async(queue, ^{
   	completion(@"success");
	});
}
main();
```



### Use inline functions, precompiled functions

```objc
// Inline function: just add a global function in the patch, such as `CGRectMake` in UIKitRefrences
CGRect CGRectMake(CGFloat x, CGFloat y, CGFloat width, CGFloat height)
{
  CGRect rect;
  rect.origin.x = x; rect.origin.y = y;
  rect.size.width = width; rect.size.height = height;
  return rect;
}
// Pre-compiled function: you need to add the following code in the App
[[MFScopeChain top] setValue:[MFValue valueWithBlock:^void(dispatch_once_t *onceTokenPtr,
                                                                  dispatch_block_t _Nullable handler){
        dispatch_once(onceTokenPtr,handler);
    }] withIndentifier:@"dispatch_once"];
```

### How to determine if source files are included in a patch

![image](https://raw.githubusercontent.com/SilverFruity/silverfruity.github.io/9a371dcb9cece8deefa4fe05b155ae7cbd5834b5/source/_posts/OCRunner/OCRunner_2.jpeg)



## Performance Testing

### Loading time

![2](https://raw.githubusercontent.com/SilverFruity/silverfruity.github.io/9a371dcb9cece8deefa4fe05b155ae7cbd5834b5/source/_posts/OCRunner/OCRunner_1.jpeg)

### Execution speed and memory usage

Device:  iPhone SE2 , iOS 14.2,  Xcode 12.1.

Take the classic Fibonacci sequence function as an example, find the test result of the value of the 25th term

#### JSPatch

* Execution time, the average time is 0.169s

  ![](./ImageSource/JSPatchTimes.png)

* Memory usage has been stable at around 12MB


![](./ImageSource/JSPatchMemory.png)

#### OCRunner
* Execution time, the average time is 1.05s

  ![](./ImageSource/OCRunnerTimes.png)

* Memory usage, peak value is about 60MB


![](./ImageSource/OCRunnerMemory.png)

#### Mango
* Execution time, the average time is 2.38s

  ![](./ImageSource/MangoTimes.png)

* The memory usage continues to rise, reaching about 350MB at the highest point.


![](./ImageSource/MangoMemory.png)

* When testing with recursive functions, the performance of OCRunner is 1/5 times JSPatch's, and is 2.5 times Mango's.
* OCRunner's patch loading speed is about 20 times + that of Mango, and this value increases as the patch size increases.  and the result of JSPatch is unknown.
* Regarding the memory usage of recursive method calls, there is currently a problem of excessive usage. When finding the 30th item of the Fibonacci sequence, Mango will burst the memory, and the peak memory usage of OCRunner is about 600MB.



## Current problems

1. Pointer and multiplication sign identification conflicts, derived problems: type conversion, etc.
2. Not support static、inline function declaration
3. Not support C array declaration:  type a[]、type a[2]、value = { 0 , 0 , 0 , 0 }
4. Not support '->' operation symbol...
5. Not support fix C function 



## Support grammar

1. Class declaration and implementation, support Category
2. Protocol
3. Block
4. struct、enum、typedef
5. Use function declarations to link system function pointers
6. Global function
7. Multi-parameter call (methods and functions)
8. **\***、**&**  (Pointer operation)
9. Variable static keyword
10. NSArray: @[value1, value2]，NSDictionary: @{ key: value },  NSNumer:  @(value)
11. NSArray, NSDictionary value and assignment syntax: id value = a[var];  a[var] = value;
12. [Operator, except for'->' all have been implemented](https://en.cppreference.com/w/c/language/operator_precedence)

etc.


### Thanks for
* [Mango](https://github.com/YPLiang19/Mango)
* [libffi](https://github.com/libffi/libffi)
* Procedure Call Standard for the ARM 64-bit Architecture. 
* [@jokerwking](https://github.com/jokerwking)


