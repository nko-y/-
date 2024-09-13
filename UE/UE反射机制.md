#### 基本原理

（1）含义：

- 描述了类在运行时的内容(用于在运行时识别对象和类的信息)，这些数据存储了类的名称、类的数据成员、每个数据成员的类型、每个成员位于内存应向的偏移、类的成员函数信息
- 可以用于实现序列化，editor的detail panel，垃圾回收，网络复制，蓝图/C++通信和相互调用
- 只有主动标记的类型、属性、方法会被反射系统追踪，Unreal HeaderTool会收集这些信息，生成用于支持反射机制的C++代码

（2）一个简单的示意

```c++
// 这个是获取变量在结构体中的偏移的宏
#define offsetof(A,m) (int)(&((A*)0)->m);
struct A{
    int p1;
    int p2;
    void func(int i){
        cout << i << endl;
    }
};

// 这一行是获取变量p2的偏移，换算到UE中这一行就是UHT帮忙自动生成的
int param_p2_offset = offsetof(A, p2);
void (A:: * func_ptr)(int) = &A::func;
    
int main(){
    A a = {10, 100};
    void* param_p2_addr = (char*)&a + param_p2_offset;
    cout << *(int*)param_p2_addr;
    
    (a.*func_ptr)(-1);
}
```

（3）原理：

- 标记：
  - 包含反射类型的头文件 #include "FileName.generated.h"
  - 可以使用ENUM()、UCLASS()、USTRUCT()、UFUNCTION()、UPROPERTY()来标记不同的类型和成员变量

- UnrealHeaderTool 和 UnrealBuildTool
  - UBT扫描头文件记录所有包含反射类型的modules，当头文件改变时就会用UHT更新反射数据
  - UHT解析头文件扫描标记生成支持反射的C++代码，包括filename.generated.h 和 filename.gen.cpp。将反射数据保存为C++代码的好处是可以与二进制文件保持一致永远不会加载过期的反射数据
  - 不要用#if #endif把标记包起来这样会出现错误，需要用兼容宏比如WITH_EDITOR 和 WITH_EDITORONLY_DATA



#### 实例分析

```c++
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/PlayerController.h"
#include "TestPlayerController.generated.h"
 
UCLASS()
 
class SHOOTERGAME_API ATestPlayerController : public
APlayerController
{
 GENERATED_BODY()
 
 //Construct
public:
 ATestPlayerController();
 
 //Property
public:
 UPROPERTY(BlueprintReadOnly, Category = PlayerController)
 AActor *TestProperty;
 
 UPROPERTY(Replicated)
 int RepProperty;
 
 //Function
public:
 UFUNCTION(Server, Reliable, WithValidation)
 void ServerRPC(int a, bool b);
 
 UFUNCTION(Client, Reliable)
 void ClientRPC(int a, bool b);
 
 UFUNCTION(BlueprintCallable)
 void BlueprintCallableFunc(int a, bool b);
 
 UFUNCTION(BlueprintNativeEvent)
 void BlueprintNativeEventFunc(int a, bool b);
};
```

（1）filename.generated.h

- 最终转换为文件ID、行号、"GENERATED_BODY"字符组合成的标记

```c++
#define GENERATED_BODY(...) BODY_MACRO_COMBINE(CURRENT_FILE_ID,_,__LINE__,_GENERATED_BODY);
#define BODY_MACRO_COMBINE_INNER(A,B,C,D) A##B##C##D 
```

- 实际包含的是如下两个宏，各自都相当于一系列宏组合，这里_LEGACY标签的宏是为了服务UE4旧代码

```c++
#define ShooterGame_Source_ShooterGame_Public_Player_TestPlayerController_h_15_GENERATED_BODY_LEGACY
#define ShooterGame_Source_ShooterGame_Public_Player_TestPlayerController_h_15_GENERATED_BODY \
PRAGMA_DISABLE_DEPRECATION_WARNINGS \
public: \
 ShooterGame_Source_ShooterGame_Public_Player_TestPlayerController_h_15_PRIVATE_PROPERTY_OFFSET \
 ShooterGame_Source_ShooterGame_Public_Player_TestPlayerController_h_15_RPC_WRAPPERS_NO_PURE_DECLS \
 ShooterGame_Source_ShooterGame_Public_Player_TestPlayerController_h_15_CALLBACK_WRAPPERS \
 ShooterGame_Source_ShooterGame_Public_Player_TestPlayerController_h_15_INCLASS_NO_PURE_DECLS \
 ShooterGame_Source_ShooterGame_Public_Player_TestPlayerController_h_15_ENHANCED_CONSTRUCTORS \
private: \
PRAGMA_ENABLE_DEPRECATION_WARNINGS
```

- CURRENT_FILE_ID就是这个文件的ID

```c++
#undef CURRENT_FILE_ID
#define CURRENT_FILE_ID ShooterGame_Source_ShooterGame_Public_Player_TestPlayerController_h 
```

- 组合宏的第一个，定义了一些用于包装函数参数的struct，函数的名称变成了xxx_Implementation, xxx_Validate，DECLARE_FUNCTION用于创建一些模板代码需要加上exec前缀

```c++
#define ShooterGame_Source_ShooterGame_Public_Player_TestPlayerController_h_15_RPC_WRAPPERS_NO_PURE_DECLS \
 
 virtual void BlueprintNativeEventFunc_Implementation(int32 a, bool b); \
 virtual void ClientRPC_Implementation(int32 a, bool b); \
 virtual bool ServerRPC_Validate(int32 , bool ); \
 virtual void ServerRPC_Implementation(int32 a, bool b); \
\
 DECLARE_FUNCTION(execBlueprintNativeEventFunc) \
 { \
 P_GET_PROPERTY(UIntProperty,Z_Param_a); \
 P_GET_UBOOL(Z_Param_b); \
 P_FINISH; \
 P_NATIVE_BEGIN; \
 P_THIS->BlueprintNativeEventFunc_Implementation(Z_Param_a,Z_Param_b); \
 P_NATIVE_END; \
 } \ 
```

- 组合宏的最后一个，移动构造和拷贝构造被声明为了private防止外部调用

```c++
#define ShooterGame_Source_ShooterGame_Public_Player_TestPlayerController_h_15_ENHANCED_CONSTRUCTORS \
 
private: \
 /** Private move- and copy-constructors, should never be used */ \
 NO_API ATestPlayerController(ATestPlayerController&&); \
 NO_API ATestPlayerController(const ATestPlayerController&); \
public: \
 DECLARE_VTABLE_PTR_HELPER_CTOR(NO_API, ATestPlayerController); \
DEFINE_VTABLE_PTR_HELPER_CTOR_CALLER(ATestPlayerController); \
 DEFINE_DEFAULT_CONSTRUCTOR_CALL(ATestPlayerController) 
```

- DECLARE_VTABLE_PTR_HELPER_CTOR和DEFINE_VTABLE_PTR_HELPER_CTOR_CALLER用于热加载相关。
- DEFINE_DEFAULT_CONSTRUCTOR_CALL定义并实现了一个默认构造函数

（2）filename.generated.cpp

- 内部用ProcessEvent进行了封装，调用对应的函数时先在Map中通过函数名来查找
- StaticRegisterNativesATestPlayerController用于向UClass中注册C++原生函数

```c++
static FName NAME_ATestPlayerController_ServerRPC = FName(TEXT("ServerRPC"));
 
 void ATestPlayerController::ServerRPC(int32 a, bool b)
 {
 TestPlayerController_eventServerRPC_Parms Parms;
 Parms.a=a;
 Parms.b=b ? true : false;
 ProcessEvent(FindFunctionChecked(NAME_ATestPlayerController_ServerRPC),&Parms);
 }
 void ATestPlayerController::StaticRegisterNativesATestPlayerController()
 {
 UClass* Class = ATestPlayerController::StaticClass();
 static const FNameNativePtrPair Funcs[] = {
 { "BlueprintCallableFunc", &ATestPlayerController::execBlueprintCallableFunc },
 { "BlueprintNativeEventFunc", &ATestPlayerController::execBlueprintNativeEventFunc },
 { "ClientRPC", &ATestPlayerController::execClientRPC },
 { "ServerRPC", &ATestPlayerController::execServerRPC },
 };
 FNativeFunctionRegistrar::RegisterFunctions(Class, Funcs, ARRAY_COUNT(Funcs));
 } 
```

- UFUNCTION(…)中的信息都封装在了Z_Construct_UFunction_ATestPlayerController_BlueprintCallableFunc_Statics::FuncParams中，包括函数的名称，flag，metadata等等。函数的参数在Z_Construct_UFunction_ATestPlayerController_BlueprintCallableFunc_Statics中有描述。

```c++
UFunction* Z_Construct_UFunction_ATestPlayerController_BlueprintCallableFunc()
 
{
 static UFunction* ReturnFunction = nullptr;
 if (!ReturnFunction)
 {
 UE4CodeGen_Private::ConstructUFunction(ReturnFunction, Z_Construct_UFunction_ATestPlayerController_BlueprintCallableFunc_Statics::FuncParams);
 }
 return ReturnFunction;
} 
```

- Z_Construct_UClass_ATestPlayerController_Statics 为该类的描述信息，其中包含了函数信息、变量属性信息、metadata等等。
- IMPLEMENT_CLASS用于在startup时注册该类。
- FCompiledInDefer用于暂存生成UClass的函数，即Z_Construct_UClass_ATestPlayerController()，之后这个函数将被执行。
- DEFINE_VTABLE_PTR_HELPER_CTOR定义了一个接受FVTableHelper为参数的构造函数





参考文献

1. [UE反射机制](https://zhuanlan.zhihu.com/p/60622181)
2. [UE5：反射系统原理](https://zhuanlan.zhihu.com/p/656818991) 





















