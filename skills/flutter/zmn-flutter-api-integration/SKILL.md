---
name: zmn-flutter-api-integration
description: Flutter 接口联调技能。根据接口信息完成 Api 地址注册、实体类创建（有返回数据时）、Api 层请求方法编写、Logic 层业务逻辑处理。用户提供接口地址和入参/返回字段即可自动生成标准代码。
license: MIT
metadata:
  author: zmn-team
  version: "1.0"
---

# Flutter 接口联调技能

> 当用户提供接口地址、入参、返回字段时，按照本技能完成标准化的接口对接工作。

## 触发场景

- 用户提供了接口地址和入参/返回字段，要求对接
- 用户说"联调"、"对接接口"、"写请求代码"
- 用户提供了后端 API 文档并要求实现调用

## 前置判断

拿到接口信息后，**首先判断接口是否有返回数据**：

| 情况 | 判断依据 | 处理方式 |
|------|---------|---------|
| **有返回数据** | 接口返回 JSON 中有业务字段 | 走 [情况一](#情况一有返回数据) |
| **无返回数据** | 用户明确说明"无返回数据"或仅需判断请求成功 | 走 [情况二](#情况二无返回数据) |

---

## 情况一：有返回数据

### 步骤 1：创建实体类

在当前页面目录下的 `entity/` 文件夹中新建实体类文件。

**文件位置**：`modules/xxx/entity/xxx_entity.dart`

**标准模板**：

```dart
/// 接口描述注释
/// 接口：/ratel/xxx/xxxService/xxxMethod
class XXXEntity {
  XXXEntity({
    this.field1,
    this.field2,
  });

  XXXEntity.fromJson(dynamic json) {
    field1 = json['field1'];
    field2 = json['field2'];
  }

  String? field1; // 字段1说明
  int? field2; // 字段2说明

  Map<String, dynamic> toJson() {
    final data = <String, dynamic>{};
    data['field1'] = field1;
    data['field2'] = field2;
    return data;
  }

  /// 按需添加计算属性
  // bool get isXxx => field2 == 1;

  /// Mock 数据
  static XXXEntity mock() {
    return XXXEntity(
      field1: 'mock_value',
      field2: 1,
    );
  }
}
```

**实体类规范**：
- 必须包含：`fromJson` 命名构造、`toJson` 实例方法、`static mock()` 静态方法
- 字段类型使用可空类型（`String?`、`int?`）
- 每个字段后添加行内注释说明用途
- 文件开头写接口描述和接口地址注释
- 按需添加计算属性（如枚举值转 bool、格式化等）

### 步骤 2：注册接口地址

在 `lib/utils/api.dart` 的 `Api` 类末尾添加接口常量：

```dart
/// 接口中文描述
static const xxxMethod = '/ratel/xxx/xxxService/xxxMethod';
```

### 步骤 3：编写 Api 层请求方法

在 `logic.dart` 的 `XXXApi` 类中添加方法：

```dart
/// 接口中文描述
Future<XXXEntity?> xxxMethod({
  required String param1,
  required String param2,
}) async {
  BaseResp resp = await HttpManger.request(
    Api.xxxMethod,
    reqData: {
      'param1': param1,
      'param2': param2,
    },
  );
  if (!resp.isSuccess) return null;
  return XXXEntity.fromJson(resp.dict);
}
```

### 步骤 4：编写 Logic 层业务逻辑

在 `XXXLogic` 中调用 Api 并处理响应：

```dart
Future<void> doSomething() async {
  try {
    final entity = await _api.xxxMethod(
      param1: state.param1,
      param2: state.param2,
    );

    if (entity == null) {
      Toast.show('操作失败，请重试');
      return;
    }

    // 处理返回数据（更新 State、刷新 UI 等）
    state.pageData = entity;
    update();
  } catch (e) {
    LogUtils.e('操作失败: $e');
    Toast.show('操作失败，请重试');
  }
}
```

### 步骤 5：在 logic.dart 中 import 实体类

```dart
import 'entity/xxx_entity.dart';
```

---

## 情况二：无返回数据

**不需要创建实体类**，只需判断请求是否成功。

### 步骤 1：注册接口地址

同情况一，在 `lib/utils/api.dart` 中添加常量。

### 步骤 2：编写 Api 层请求方法

返回类型为 `Future<bool>`，直接返回 `resp.isSuccess`：

```dart
/// 接口中文描述（无返回数据，仅判断请求成功）
Future<bool> xxxMethod({
  required String param1,
  required String param2,
}) async {
  BaseResp resp = await HttpManger.request(
    Api.xxxMethod,
    reqData: {
      'param1': param1,
      'param2': param2,
    },
  );
  return resp.isSuccess;
}
```

### 步骤 3：编写 Logic 层业务逻辑

```dart
Future<void> doSomething() async {
  try {
    final success = await _api.xxxMethod(
      param1: state.param1,
      param2: state.param2,
    );

    if (success) {
      // 成功后的业务逻辑（跳转、刷新等）
    } else {
      Toast.show('操作失败');
    }
  } catch (e) {
    LogUtils.e('操作失败: $e');
    Toast.show('操作失败，请重试');
  }
}
```

---

## 禁止事项

- **禁止手动 loading**：不要在请求前后添加 `Toast.showLoading()` / `Toast.dismiss()`，加载动画已封装在网络层 `HttpManger` 中统一处理
- **禁止创建空实体类**：无返回数据的接口不需要创建实体类，直接用 `resp.isSuccess` 判断
- **禁止硬编码颜色**：业务代码中禁止使用 `HexColor('#...')` 或 `Color(0xFF...)`，必须使用 `lib/ui/colors.dart` 中的常量

## 完整对接流程检查清单

```
□ 1. 判断接口是否有返回数据
□ 2. [有返回] 在 entity/ 目录创建实体类（fromJson + toJson + mock）
□ 3. 在 api.dart 中注册接口地址常量
□ 4. 在 logic.dart 的 Api 类中编写请求方法
□ 5. 在 logic.dart 的 Logic 类中编写业务逻辑
□ 6. [有返回] 在 logic.dart 中 import 实体类
□ 7. 在 view.dart 中绑定按钮/事件调用
□ 8. 编译验证无报错
```

## 真实案例参考

以下均来自收款页面（collect_payment）联调实践：

### 案例 1：有返回数据 — 获取页面初始化数据

**接口**：`getAfterSubmitOrderPage`  
**返回**：`{ qrCode, concerned, amount, amountType, countDownEndTime }`

```
entity/collect_payment_entity.dart  ← 新建实体类
api.dart                            ← 注册接口地址
logic.dart (Api)                    ← Future<CollectPaymentEntity?> 方法
logic.dart (Logic)                  ← initPageData() 处理响应，赋值 state.pageData
```

### 案例 2：有返回数据 — 推送微信消息

**接口**：`pushWechatMessage`  
**返回**：`{ pushStatus, concerned }`

```
entity/push_message_entity.dart     ← 新建实体类
api.dart                            ← 注册接口地址
logic.dart (Api)                    ← Future<PushMessageEntity?> 方法
logic.dart (Logic)                  ← sendPushMessage() 处理响应，更新关注状态
```

### 案例 3：有返回数据 — 切换用户支付

**接口**：`modifySaleWorkUserPay`  
**返回**：`{ qrCode }`

```
entity/modify_user_pay_entity.dart  ← 新建实体类
api.dart                            ← 注册接口地址
logic.dart (Api)                    ← Future<ModifyUserPayEntity?> 方法
logic.dart (Logic)                  ← modifyUserPay() 返回二维码 URL
```

### 案例 4：无返回数据 — 校验已支付

**接口**：`checkPaid`  
**返回**：无业务字段

```
（无需实体类）
api.dart                            ← 注册接口地址
logic.dart (Api)                    ← Future<bool> 方法，return resp.isSuccess
logic.dart (Logic)                  ← checkPaidAndGoNext() 成功跳转/失败提示
```
