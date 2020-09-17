## Android R 新特性

### 1、隐私权更新

| 隐私权变更                                                   | 受影响的应用                                                 | 缓解策略                                                     |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **强制执行分区存储机制** 以 Android 11 为目标平台的应用始终会受分区存储行为的影响 | 以 Android 11 为目标平台的应用，以及以 Android 10 为目标平台且未将 `requestLegacyExternalStorage` 设为 `true` 以停用分区存储的应用 | 更新您的应用以使用分区存储 [详细了解分区存储变更](https://developer.android.google.cn/preview/privacy/storage#scoped-storage) |
| **单次授权** 使用单次授权功能，用户可以授予对位置信息、麦克风和摄像头的临时访问权限 | 以任何版本为目标平台且请求位置信息、麦克风或摄像头权限的应用 | 在尝试访问受某项权限保护的数据之前，检查您的应用是否具有该权限 [遵循请求权限方面的最佳做法](https://developer.android.google.cn/training/permissions/requesting) |
| **自动重置权限** 如果用户在 Android 11 上几个月未与应用互动，系统会自动重置应用的敏感权限 | 以 Android 11 为目标平台且在后台执行大部分工作的应用         | 要求用户阻止系统重置应用的权限 [详细了解自动重置权限](https://developer.android.google.cn/preview/privacy/permissions#auto-reset) |
| **后台位置信息访问权限** Android 11 更改了用户向应用授予后台位置信息权限的方式 | 以 Android 11 为目标平台且需要访问[后台位置信息](https://developer.android.google.cn/training/location/permissions#background)的应用 | 通过对权限请求方法的多次单独调用，逐步请求前台（粗略或精确）和后台位置权限。必要时，说明用户授予该权限所能得到的益处 [详细了解 Android 11 中的后台位置信息访问权限](https://developer.android.google.cn/preview/privacy/location#background-location) |
| **软件包可见性** Android 11 更改了应用查询同一设备上的其他已安装应用及与之互动的方式 | 以 Android 11 为目标平台且与设备上的其他已安装应用交互的应用 | 将 `<queries>` 元素添加到应用的清单 [详细了解软件包可见性](https://developer.android.google.cn/preview/privacy/package-visibility) |
| **前台服务** Android 11 更改了前台服务访问位置信息、摄像头和麦克风相关数据的方式 | 以任何版本为目标平台且在前台服务中访问位置信息、摄像头或麦克风的应用 | 分别针对需要访问摄像头和麦克风的前台服务，声明 `camera` 和 `microphone` 前台服务类型。但请注意，应用在后台运行时启动的前台服务通常无法访问位置信息、摄像头或麦克风。 [详细了解前台服务的变更](https://developer.android.google.cn/preview/privacy/foreground-services) |

#### 1.1、强制执行分区存储机制

<u>首先要知道的是：</u>

**Android 中存储可以分为两大类：私有存储和共享存储**

- 私有存储 (Private Storage) : 每个应用在都拥有自己的私有目录，其它应用看不到，彼此也无法访问到该目录

  **内部**存储私有目录 (/data/data/packageName) 

  **外部**存储私有目录 (/sdcard/Android/data/packageName)

- 共享存储 (Shared Storage) : 存储其他应用可访问文件， 包含媒体文件、文档文件以及其他文件，

  对应设备DCIM、Pictures、Alarms、Music、Notifications、Podcasts、Ringtones、Movies、Download等目录

##### 1.1.1、什么是分区存储？

因为大部分应用都会请求 ( READ_EXTERNAL_STORAGE ) ( WRITE_EXTERNAL_STORAGE ) 存储权限，来做一些诸如在 SD 卡中存储文件或者读取多媒体文件等常规操作。这些应用可能会在磁盘中存储大量文件，即使应用被卸载了还会依然存在。另外，这些应用还可能会读取其他应用的一些敏感文件数据。为此，Google 在 Android 10 中引入了**分区存储**，**对权限进行场景的细分，按需索取**

**也就是说分区存储机制早在android 10 就已经引入了，只是在 Android 11 中进行了进一步的调整，需要强制执行并做了一些调整**

##### 1.1.2、共享目录权限的划分

Android 10 中主要对共享目录进行了权限详细的划分，不再能通过绝对路径访问

**访问不同分区的方式：**

1. 私有目录：和以前的版本一致，可通过 File() API 访问，无需申请权限
2. 共享目录：需要通过MediaStore和Storage Access Framework API 访问，视具体情况申请权限

其中，对共享目录的权限进行了细分：

1. 无需申请权限的操作：
   通过 MediaStore API对媒体集、文件集进行媒体/文件的添加、对 **自身APP** 创建的 媒体/文件 进行查询、修改、删除的操作
2. 需要申请READ_EXTERNAL_STORAGE 权限：
   通过 MediaStore API对所有的媒体集进行查询、修改、删除的操作。
3. 调用 Storage Access Framework API ：
   会启动系统的文件选择器向用户申请操作指定的文件

新的访问方式：

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8yMzA4OTIwNS02NDM3NWRkN2JiZjRkZTFm?x-oss-process=image/format,png)

##### 1.1.3、android11的一些调整

- **新的权限MANAGE_EXTERNAL_STORAGE**

  在 Android 11 上，使用分区存储模型的应用只能访问自身的应用专用缓存文件。如果您的应用需要管理设备存储空间，请执行以下操作：

  1.在清单中声明 MANAGE_EXTERNAL_STORAGE 权限
  2.使用 ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION intent 操作将用户引导至一个系统设置页面，在该页面上，用户可以的应用启用以下选项：授予所有文件的管理权限

1. MANAGE_EXTERNAL_STORAGE : 类似以前的 READ_EXTERNAL_STORAGE + WRITE_EXTERNAL_STORAGE ，除了应用专有目录都可以访问
2. 如果设备上的可用空间不足，请提示用户同意让应用清除所有缓存。为此，请调用 ACTION_CLEAR_APP_CACHE intent 操作
    **注意**：`ACTION_CLEAR_APP_CACHE` intent 操作会严重影响设备的电池续航时间，并且可能会从设备上移除大量的文件
3. 在 Google Play 上架的话，需要提交使用此权限的说明，只有指定的几种类型的 APP 才能使用
- **新增执行批量操作**

  为实现各种设备之间的一致性并增加用户便利性，Android 11 向 [`MediaStore`] API 中添加了多种方法。对于希望简化特定媒体文件更改流程（例如在原位置编辑照片）的应用而言，这些方法尤为有用

  添加的方法如下：

  - [`createWriteRequest()`]

    用户向应用授予对指定媒体文件组的写入访问权限的请求

  - [`createFavoriteRequest()`]

    用户将设备上指定的媒体文件标记为“收藏”的请求。对该文件具有读取访问权限的任何应用都可以看到用户已将该文件标记为“收藏”

  - [`createTrashRequest()`]

    用户将指定的媒体文件放入设备垃圾箱的请求。垃圾箱中的内容会在系统定义的时间段后被永久删除。**注意**：如果您的应用是设备 OEM 的预安装图库应用，您可以将文件放入垃圾箱而不显示对话框。如需执行该操作，请直接将 [`IS_TRASHED`] 设置为 `1`

  - [`createDeleteRequest()`]

    用户立即永久删除指定的媒体文件（而不是先将其放入垃圾箱）的请求

  系统在调用以上任何一个方法后，会构建一个 [`PendingIntent`]对象。应用调用此 intent 后，用户会看到一个对话框，请求用户同意应用更新或删除指定的媒体文件

  例如，以下是构建 `createWriteRequest()` 调用的方法：

  ```java
  https://developer.android.google.cn/previewList<Uri> urisToModify = /* A collection of content URIs to modify. */
  PendingIntent editPendingIntent = MediaStore.createWriteRequest(contentResolver,
                    urisToModify);
  
  // Launch a system prompt requesting user permission for the operation.
  startIntentSenderForResult(editPendingIntent.getIntentSender(),
      EDIT_REQUEST_CODE, null, 0, 0, 0);
  ```

  评估用户的响应，然后继续操作，或者在用户不同意时向用户说明您的应用为何需要获取权限：

  ```java
  @Override
  protected void onActivityResult(int requestCode, int resultCode,
                     @Nullable Intent data) {
      ...
      if (requestCode == EDIT_REQUEST_CODE) {
          if (resultCode == Activity.RESULT_OK) {
              /* Edit request granted; proceed. */
          } else {
              /* Edit request not granted; explain to the user. */
          }
      }
  }
  ```

  createFavoriteRequest()、createTrashRequest()和 createDeleteRequest()使用方法一样
  
- **使用直接文件路径和原生库访问文件**

   为了帮助应用更顺畅地使用第三方媒体库，Android 11 允许使用除 MediaStore API 之外的 API 访问共享存储空间中的媒体文件。也可以转而选择使用以下任一 API 直接访问媒体文件：
  
   File API
   原生库，例如 fopen()
  
   **性能：**
  
   通过 File () 等直接通过路径访问的 API 实际上也会映射为MediaStore API 
   按文件路径顺序读取的时候性能相当；随机读取和写入的时候则会更慢

  **简单来说就是，可以通过 File() 等API 访问有权限访问的媒体集了，还是推荐使用MediaStoreAPI**

- **限制访问其他应用中的数据**

  <u>访问内部存储设备上的数据目录</u>

  如果应用以 Android 11 为目标平台，则不能访问其他任何应用的数据目录中的文件，即使其他应用以 Android 8.1或更低版本为目标平台且已使其数据目录中的文件全局可读也是如此

  <u>访问外部存储设备上的应用专用目录</u>

  在 Android 11 上，应用无法再访问外部存储设备中的任何其他应用的专用于特定应用的目录中的文件
  
  **简言之在 Android 11 上不能访问其他应用中的数据了**
  
- **文档访问限制**

  访问目录

  无法再使用 [`ACTION_OPEN_DOCUMENT_TREE`] intent 操作请求访问以下目录：

  - 内部存储卷的根目录
  - 设备制造商认为可靠的各个 SD 卡卷的根目录，无论该卡是模拟卡还是可移除的卡。可靠的卷是指应用在大多数情况下可以成功访问的卷
  - `Download` 目录

  访问文件

  无法再使用 `ACTION_OPEN_DOCUMENT_TREE` 或 [`ACTION_OPEN_DOCUMENT`]intent 操作请求用户从以下目录中选择单独的文件：

  - `Android/data/` 目录及其所有子目录
  - `Android/obb/` 目录及其所有子目录
  
  **Android 11 上访问设备上的文档受到了更大的限制**

#### 1.2、单次授权

在 Android 11 中，每当应用请求与位置信息、麦克风或摄像头相关的权限时，面向用户的权限对话框会包含**仅限这一次**选项。如果用户在对话框中选择此选项，系统会向应用授予临时的单次授权

然后，应用可以在一段时间内访问相关数据，具体时间取决于应用的行为和用户的操作：

- 当应用的 Activity 可见时，应用可以访问相关数据
- 如果用户将应用转为后台运行，应用可以在短时间内继续访问相关数据
- 如果您在 Activity 可见时启动了一项前台服务，并且用户随后将您的应用转到后台，那么您的应用可以继续访问相关数据，直到该前台服务停止
- 如果用户撤消单次授权（例如在系统设置中撤消），无论您是否启动了前台服务，应用都无法访问相关数据。与任何权限一样，如果用户撤消了应用的单次授权，应用进程就会终止

当用户下次打开应用并且应用中的某项功能请求访问位置信息、麦克风或摄像头时，系统会再次提示用户授予权限。

下图为MIUI12中的‘单次授权’：

<img src="/home/yaoyifei/图片/Screenshot_2020-09-10-17-38-05-145_com.miui.securitycenter.jpg" alt="Screenshot_2020-09-10-17-38-05-145_com.miui.securitycenter" style="zoom: 25%;" />

#### 1.3、自动重置权限

如果应用以 Android 11 为目标平台并且数月未使用，系统会通过自动重置用户已授予应用的运行时敏感权限来保护用户数据。此操作与用户在系统设置中查看权限并将应用的访问权限级别更改为**拒绝**的做法效果一样

**用户停用自动重置功能**

1. 点按**权限**，系统会加载**应用权限**设置屏幕
2. 关闭**如果未使用此应用，则移除相关权限**选项（设置->应用和通知->具体应用->权限）

**确定是否已停用自动重置功能**
调用 isAutoRevokeWhitelisted()。如果此方法返回true，则系统不会自动重置应用的权限

**测试自动重置功能**

如需验证系统是否重置了应用的权限，请执行以下操作：

1. 保存系统重置应用权限所需等待的默认时长。在测试后恢复此设置：

   ```
   threshold=$(adb shell device_config get permissions \
     auto_revoke_unused_threshold_millis2)
   ```

2. 减少系统重置权限所需等待的时长。下面的示例对系统进行了修改，以致当您停止与应用互动后仅一秒钟，系统就会重置应用的权限：

   ```
   adb shell device_config put permissions \
     auto_revoke_unused_threshold_millis2 1000
   ```

3. 手动调用自动重置进程，如以下代码段所示。在运行此命令之前，请确保测试设备已开启片刻（大约 45 秒钟）。

   ```
   adb shell cmd jobscheduler run -u 0 -f \
     com.google.android.permissioncontroller 2
   ```

4. 验证应用能否处理自动重置事件。

5. 恢复系统在自动重置应用权限之前所需等待的默认时长：

   `adb shell device_config put permissions \  auto_revoke_unused_threshold_millis2 $threshold`

**总结：应用如果长时间不使用，系统会收回之前授予的权限，此外，测试自动重置功能的时候我们可以发现这个时间OEM可以自己定义，而且这个功能OEM也可以选择停用**

#### 1.4、其他一些权限相关的调整

##### 1.4.1、权限对话框的可见性

Android 11 建议不要请求用户已选择拒绝的权限。在应用安装到设备上后，如果用户在使用过程中屡次针对某项特定的权限点按**拒绝**，此操作表示其希望“不再询问” 

应用：我连舔狗都不配当了吗？

##### 1.4.2、系统提醒窗口变更

向应用授予 [`SYSTEM_ALERT_WINDOW`] 权限的方式发生了一些变更。这些变更可以让权限的授予更有目的性，从而达到保护用户的目的

根据请求自动向某些应用授予 SYSTEM_ALERT_WINDOW 权限

系统会根据请求自动向某些类型的应用授予 `SYSTEM_ALERT_WINDOW` 权限：

- 系统会自动向具有 [`ROLE_CALL_SCREENING`] 且请求 `SYSTEM_ALERT_WINDOW` 的所有应用授予该权限。如果应用失去 `ROLE_CALL_SCREENING`，就会失去该权限
- 系统会自动向通过 [`MediaProjection`]截取屏幕且请求 `SYSTEM_ALERT_WINDOW` 的所有应用授予该权限，除非用户已明确拒绝向应用授予该权限。当应用停止截取屏幕时，就会失去该权限。此用例主要用于游戏直播应用

这些应用无需发送 [`ACTION_MANAGE_OVERLAY_PERMISSION`] 以获取 `SYSTEM_ALERT_WINDOW` 权限，它们只需直接请求 `SYSTEM_ALERT_WINDOW` 即可

MANAGE_OVERLAY_PERMISSION intent 始终会将用户转至系统权限屏幕

从 Android 11 开始，[`ACTION_MANAGE_OVERLAY_PERMISSION`] intent 始终会将用户转至顶级**设置**屏幕，用户可在其中授予或撤消应用的 [`SYSTEM_ALERT_WINDOW`] 权限。intent 中的任何 `package:` 数据都会被忽略

在更低版本的 Android 中，`ACTION_MANAGE_OVERLAY_PERMISSION` intent 可以指定一个软件包，它会将用户转至应用专用屏幕以管理权限。Android 11 不再支持此功能，而是必须由用户先选择要对其授予或撤消权限的应用。此变更可以让权限的授予更有目的性，从而达到保护用户的目的

总结：针对部分应用自动赋予系统提醒窗口权限，其他应用申请系统提醒窗口权限需要用户从顶级设置屏幕进入，选择该应用才能赋予

##### 1.4.3、电话号码权限变更

Android 11 更改了您的应用在读取电话号码时使用的与电话相关的权限

如果应用以 Android 11 为目标平台，而且需要调用下列方法必须请求 [`READ_PHONE_NUMBERS`] 权限，而不是 `READ_PHONE_STATE` 权限

- [`TelephonyManager`] 类和 [`TelecomManager`] 类中的 `getLine1Number()` 方法
- [`TelephonyManager`] 类中不受支持的 `getMsisdn()` 方法

#### 1.5、后台位置信息访问权限

Android 11 更改了应用中的功能获取后台位置信息访问权限的方式

##### 1.5.1、单独请求在后台访问位置信息

如果应用以 Android 11 为目标平台，系统会强制执行此最佳做法。如果同时请求前台位置信息和后台位置信息，系统会忽略该请求

##### 1.5.2、权限对话框的变更

在搭载 Android 11 的设备上，当应用中的某项功能请求在后台访问位置信息时，用户看到的系统对话框不再包含用于启用后台位置信息访问权限的按钮。如需启用后台位置信息访问权限，用户必须在设置页面上针对应用的位置权限设置**一律允许**选项

在针对后台位置信息请求运行时权限时，遵循最佳做法，帮助用户导航到此设置页面。授予权限的过程取决于应用的目标 SDK 版本

以 Android 11 为目标平台的应用

如果 shouldShowRequestPermissionRationale()返回 true，向用户显示包含以下内容的指导界面：

- 明确说明应用功能需要在后台访问位置信息的原因
- 用于授予后台位置权限的设置选项的用户可见标签。可以调用 [`getBackgroundPermissionOptionLabel()`]获取此标签。此方法的返回值会根据用户设备的语言偏好设置进行本地化
- 供用户拒绝授予权限的选项。如果用户拒绝应用在后台访问位置信息，他们应该能够继续使用应用

以 Android 10 或更低版本为目标平台的应用

当应用中的某项功能请求后台位置信息访问权限时，用户会看到一个系统对话框。此对话框包含一个选项，可用于导航到设置页面上的应用位置权限选项

#### 1.6、**软件包可见性** 

Android 11 更改了应用查询用户已在设备上安装的其他应用以及与之交互的方式。使用新的 `<queries>` 元素，应用可以定义一组自身可访问的其他应用。通过告知系统应向您的应用显示哪些其他应用，此元素有助于鼓励最小权限原则

- 如果应用以 Android 11 为目标平台，可能需要在应用的清单文件中添加 `<queries>` 元素。在 `<queries>` 元素中，可以通过**包名、intent或provider**指定应用


- 返回其他应用相关结果的 `PackageManager` 方法（如 [`queryIntentActivities()`]会根据发起调用的应用的 `<queries>` 声明进行过滤。与其他应用的显式交互（如 [`startService()`]还要求目标应用与 `<queries>` 中的某项声明相符
- 在极少数情况下，应用可能需要查询设备上的所有已安装应用或与之交互，不管这些应用包含哪些组件。为了允许应用看到其他所有已安装应用，Android 11 引入了 **QUERY_ALL_PACKAGES **权限

#### 1.7、前台服务

在 Android 11（API 级别 30）中，前台服务何时可以访问设备的位置信息、摄像头和麦克风发生了一些变化。这些变更有助于保护敏感的用户数据

##### **1.7.1、使用时访问的限制**

应用在后台运行时启动前台服务，该服务就具有以下访问限制：

- 除非用户已向您的应用授予 `ACCESS_BACKGROUND_LOCATION` 权限，否则该服务无法访问位置信息
- 该服务无法访问麦克风或摄像头

应用在前台运行时启动前台服务，该服务就具有以下访问许可：

- 如果用户已向您的应用授予 `ACCESS_BACKGROUND_LOCATION` 权限，该服务就始终都能访问位置信息。否则，如果用户已向您的应用授予 `ACCESS_FINE_LOCATION`或 `ACCESS_COARSE_LOCATION`权限，该服务就可以在使用时访问位置信息
- 如果用户已向您的应用授予 `CAMERA` 权限，该服务就可以在使用时访问摄像头
- 如果用户已向您的应用授予 `RECORD_AUDIO` 权限，该服务就可以在使用时访问麦克风

**什么是后台应用**

只要满足以下各个条件，就会将应用视为在后台运行：

1. 该应用的所有 Activity 目前对用户都不可见
2. 该应用目前未运行任何在它的某个 Activity 对用户可见时启动的前台服务。

以下列表显示了应用在后台运行时管理的常见待处理任务：

- 应用在清单文件中注册广播接收器
- 应用使用闹钟管理器设定重复闹钟
- 应用使用工作管理器将后台任务调度为工作器，或使用作业调度程序将后台任务调度为作业

**有一些例外的情况**

在以下某种情况下启动前台服务时，该服务在使用时访问位置信息、摄像头和麦克风不受限制：

- 该服务由系统组件启动
- 该服务通过与应用微件进行交互启动
- 该服务通过与通知进行交互启动
- 该服务作为从其他可见应用发送的 `PendingIntent` 启动
- 该服务由作为在设备所有者模式下运行的设备政策控制器的应用启动
- 该服务由提供 `VoiceInteractionService` 的应用启动
- 该服务由具有 `START_ACTIVITIES_FROM_BACKGROUND` 特权的应用启动

此外，如果服务的前台服务类型为 `location` 并且由具有 `ACCESS_BACKGROUND_LOCATION`权限的应用启动，则该服务始终都能访问位置信息

##### **1.7.2、前台服务的类型**

如果应用以 Android 11 为目标平台并且在前台服务中访问摄像头或麦克风，您需要在该前台服务的声明的 `foregroundServiceType` 属性中添加新的 `camera` 和 `microphone` 类型

**注意**：虽然添加前台服务类型使前台服务能够访问摄像头和麦克风，但从在后台运行的应用启动此前台服务时，它仍受到使用时权限的限制

默认情况下，如果在运行时调用 `startForeground()`，系统允许访问在应用清单中声明的每个服务类型。当然我们也可以选择限制对声明的一部分服务类型的访问

**举个例子**

```xml
<manifest>
    ...
    <service ...
        android:foregroundServiceType="location|camera|microphone" />
</manifest>
```

上述清单文件声明的三个服务类型，我们可以限制部分声明的服务类型

```java
Notification notification = ...;
Service.startForeground(notification,
        FOREGROUND_SERVICE_TYPE_LOCATION | FOREGROUND_SERVICE_TYPE_CAMERA);
```

这里之就限制了只可以使用位置信息和摄像头

### 2、功能和API

#### 2.1、设备控件

Android 11 包含一个新的 `ControlsProviderService`API，可用于提供所连接的外部设备的控件。这些控件显示于 Android 电源菜单中的**设备控制器**下。

长按power键进入：

<img src="/home/yaoyifei/.config/Typora/typora-user-images/image-20200909164153413.png" alt="image-20200909164153413" style="zoom: 50%;" />

#### 2.2、媒体控件

Android 11 更新了媒体控件的显示方式。媒体控件显示于快捷设置旁。来自多个应用的会话排列在一个可滑动的轮播界面中，其中包括在手机本地播放的会话流、远程会话流（例如在外部设备上检测到的会话或投射会话）以及可继续播放的以前的会话（按上次播放的顺序排列）

用户无需启动相关应用即可在轮播界面中重新开始播放以前的会话。当播放开始后，用户可按常规方式与媒体控件互动

#### 2.3、屏幕相关

**更好地支持瀑布屏**

Android 11 提供了一些 API 以支持瀑布屏，这是一种无边框的全面屏。这种显示屏被视为刘海屏的变体。现有的 `DisplayCutout.getSafeInset…()` 方法现在会返回能够避开瀑布区域以及刘海的**安全边衬区**。如需在瀑布区域中呈现应用内容，请执行以下操作：

- 调用 `DisplayCutout.getWaterfallInsets()` 以获取瀑布边衬区的精确尺寸
- 将窗口布局属性 `layoutInDisplayCutoutMode` 设为 [`LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS`]，以允许窗口延伸到屏幕各个边缘上的刘海和瀑布区域。但是必须确保刘海或瀑布区域中没有重要的内容

**注意**：如果未将上述窗口布局属性设为 `LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS`，Android 会在黑边模式下显示窗口，从而避开缺口和瀑布区域

**新增合页角度传感器更好地支持可折叠设备**

使用 Android 11，可以通过以下方法使运行在采用合页式屏幕配置的设备上的应用能够确定合页角度：

提供具有 `TYPE_HINGE_ANGLE`的新传感器，以及新的 `SensorEvent`，后者可以监控合页角度，并提供设备的两部分之间的角度测量值

使用这些原始测量值在用户操作设备时执行精细的动画显示

尽管对于某些类型的应用（例如启动器和壁纸）而言，知道确切的合页角度会很有用，但大多数应用都应该使用 **Jetpack窗口管理器库**，通过调用 `DeviceState.getPosture()` 检索设备状态

也可以调用registerDeviceStateChangeCallback()，以在 `DeviceState` 更改时收到通知，并在状态发生变化时做出响应。

> Jetpack 是一个由多个库组成的套件，可帮助开发者遵循最佳做法，减少样板代码并编写可在各种 Android 版本和设备中一致运行的代码，让开发者精力集中编写重要的代码。

android11针对刘海屏、瀑布屏、折叠屏等进行了适配

#### **2.4、会话相关**

**改进了会话——会话通知**

<img src="/home/yaoyifei/.config/Typora/typora-user-images/image-20200909172704302.png" alt="image-20200909172704302" style="zoom: 50%;" />

**点击**: 启动 activity
**长按**: 添加到桌面，显示气泡，提醒我，设置重要性

**聊天气泡**

气泡是 Android 10 中的一项实验性功能，需要通过开发者选项启用；在 Android 11 中，不再需要如此操作

**气泡位于应用程序上方的浮动窗口中，显示在屏幕左/右边缘的小图标，新建时会弹出消息视图**

<img src="/home/yaoyifei/.config/Typora/typora-user-images/image-20200909173259146.png" alt="image-20200909173259146" style="zoom:50%;" />

<img src="/home/yaoyifei/.config/Typora/typora-user-images/image-20200909173554365.png" alt="image-20200909173554365" style="zoom:50%;" />

**何时显示气泡**：

为了减少对用户造成干扰的次数，气泡仅会在满足以下一个或多个条件时显示：

- 通知使用 [MessagingStyle]，并添加了 [Person]
- 通知来自对 Service.startForeground 的调用，类别为 CATEGORY_CALL，并添加了 Person
- 发送通知时，应用在前台运行

如果上述条件均不满足，则仅显示通知

**怎样发送气泡**：

- 按照一般步骤创建通知。
- 调用 `Notification.BubbleMetadata.Builder`以创建 BubbleMetadata 对象
- 使用 `setBubbleMetadata` 将元数据添加到通知中

气泡功能有多项改进，现在用户可以更灵活地在每个应用中启用和停用气泡功能。对于实现了实验性支持的开发者，**Android 11 中的 API 有一些变更**：

- 不带参数的 `BubbleMetadata.Builder()`构造函数已弃用。改为使用 `BubbleMetadata.Builder(PendingIntent, Icon)`(android.app.PendingIntent, android.graphics.drawable.Icon)) 和 `BubbleMetadata.Builder(String)`这两个新构造函数中的任意一个
- 通过调用 `BubbleMetadata.Builder(String)`，根据快捷方式 ID 创建 `BubbleMetadata`传递的字符串应与提供给 `Notification.Builder` 的快捷方式 ID 一致
- 使用 `Icon.createWithContentUri()` 或新方法 `createWithAdaptiveBitmapContentUri()` 创建气泡图标

#### 2.5、5G图标显示

在 Android 11（API 级别 30）及更高版本中，具有 `android.Manifest.permission.READ_PHONE_STATE` 权限的应用可以通过 `PhoneStateListener.onDisplayInfoChanged()` 请求更新电话显示信息，其中包括用于营销和品牌塑造的无线接入技术信息。

这款新 API 提供了适用于不同运营商的各种 5G 图标显示解决方案。支持的技术包括：

- LTE
- 采用载波聚合技术的 LTE (LTE+)
- 高级专业版 LTE (5Ge)
- NR (5G)
- 毫米波移动网络频段上的 NR (5G+)

#### 2.6、安全相关

**生物识别身份验证机制更新**

Android 11 引入了 `BiometricManager.Authenticators` 接口，该接口定义了 `BiometricManager` 类支持的身份验证类型：

- `BIOMETRIC_STRONG`

  使用满足[兼容性定义]页面上定义的**强**强度级别要求的硬件元素进行身份验证

- `BIOMETRIC_WEAK`
  使用满足[兼容性定义]页面上定义的**弱**强度级别要求的硬件元素进行身份验证

- `DEVICE_CREDENTIAL`
  使用屏幕锁定凭据（即用户的 PIN 码、解锁图案或密码）进行身份验证

**安全共享大型数据集**

在某些情况下，例如涉及机器学习或媒体播放时，我们的应用可能需要与其他应用使用同一个大型数据集

在较早的 Android 版本中，我们的应用与其他应用需要各自单独下载该数据集，为帮助减少网络中和磁盘上的数据冗余，Android 11 允许使用共享数据 blob 在设备上缓存这些大型数据集

**因 OTA 更新而重启设备后在未提供用户凭据的情况下执行文件级加密**

设备完成 OTA 更新并重启后，放在受凭据保护的存储空间中的凭据加密 (CE) 密钥可立即用于执行文件级加密 (FBE)操作

这意味着，完成 OTA 更新后，应用可以在用户输入其 PIN 码、解锁图案或密码之前恢复需要 CE 密钥的操作

#### 2.7、性能和质量

**无线调试**

Android 11 支持通过 Android 调试桥 (adb) 从工作站以无线方式部署和调试应用

可以将可调试的应用部署到多台远程设备，而无需通过 USB 实际连接您的设备，从而避免常见的 USB 连接问题（例如驱动程序安装方面的问题）

**ADB 增量 APK 安装**

在设备上安装大型（2GB 以上）APK 可能需要很长的时间，即使应用只是稍作更改也是如此。ADB（Android 调试桥）增量 APK 安装可以安装足够的 APK 以启动应用，同时在后台流式传输剩余数据，从而加速这一过程。如果设备支持该功能，并且安装了最新的SDK 平台工具，`adb install` 将自动使用此功能。如果不支持，系统会自动使用默认安装方法

运行**adb install --incremental**命令以使用该功能。如果设备不支持增量安装，该命令将会失败并输出详细的解释

在运行 ADB 增量 APK 安装之前，您必须先为 APK 签名并创建一个**APK 签名方案 v4 文件**。必须将 v4 签名文件放在 APK 旁边，才能使此功能正常运行

**APK 签名方案 v4 文件**

Android 11 添加了对APK 签名方案 v4的支持。此方案会在单独的文件 (`apk-name.apk.idsig`) 中生成一种新的签名，但在其他方面与 v2 和 v3 类似。没有对 APK 进行任何更改。此方案支持ADB 增量 APK 安装，这样会加快 APK 安装速度

**文本和输入**

**改进了 IME 转换**

Android 11 引入了新的 API 以改进输入法 (IME) 的转换，例如屏幕键盘。这些 API 可让您更轻松地调整应用内容，与 IME 的出现和消失以及状态和导航栏等其他元素保持同步

如需在聚焦至任何 `EditText` 时显示 IME调用 `view.getInsetsController().show(Type.ime())`

隐藏 IME调用 `view.getInsetsController().hide(Type.ime())`

调用 `view.getRootWindowInsets().isVisible(Type.ime())` 检查 IME 当前是否可见

同步应用的视图与 IME 的显示和消失，通过提供 `WindowInsetsAnimation.Callback` 到 `View.setWindowInsetsAnimationCallback()`在视图上设置监听器

IME 会调用监听器的 `onPrepare()` 方法，之后会在转换开始时调用 `onStart()`。然后，它会在每次转换的过程中调用 `onProgress()`。转换完成后，IME 会调用 `onEnd()`

在转换过程中，您随时可以调用 `WindowInsetsAnimation.getFraction()` 以了解转换的进度

> 有关如何使用这些 API 的示例，请参阅https://github.com/android/user-interface-samples/tree/master/WindowInsetsAnimation
>

#### 2.8、其他功能

**Frame rate API**

Android 11 提供了一个 API，可让应用告知系统其预期帧速率，从而减少支持多个刷新率的设备上的抖动，适配高刷新率

**支持并发使用多个摄像头**

Android 11 添加了 API 以查询对同时使用多个摄像头（包括前置摄像头和后置摄像头）的支持

如需在运行应用的设备上检查支持情况，请使用以下方法：

- `getConcurrentCameraIds()`可返回摄像头 ID 组合 `Set`，这些组合可与有保证的数据流组合并发进行流式传输（如果它们是由同一应用进程配置的）
- `isConcurrentSessionConfigurationSupported()` 可查询摄像头设备是否可以并发支持相应的会话配置

**应用进程退出原因**

Android 11 引入了 `ActivityManager.getHistoricalProcessExitReasons()`方法，用于报告近期任何进程终止的原因。应用可以使用此方法收集崩溃诊断信息，例如进程终止是由于 ANR、内存问题还是其他原因所致。

此外，还可以使用新的 `setProcessStateSummary()`方法存储自定义状态信息，以便日后进行分析

`getHistoricalProcessExitReasons()` 方法会返回 `ApplicationExitInfo` 类的实例，该类包含与应用进程终止相关的信息。通过对此类的实例调用 `getReason()`，您可以确定应用进程终止的原因。

例如，返回值为 `REASON_CRASH` 表示您的应用中发生了未处理的异常。如果您的应用需要确保退出事件的唯一性，它可以维护一个应用专用的标识符，如基于来自 `getTimestamp()` 方法的时间戳的哈希值

> **注意**：某些设备无法报告内存不足终止事件。在这些设备上，`getHistoricalProcessExitReasons()` 方法会返回 `REASON_SIGNALED` 而不是 `REASON_LOW_MEMORY`，并且 `getStatus()`的返回值为 `SIGKILL`
>
> 如需检查设备是否可以报告内存不足终止事件，请调用 `ActivityManager.isLowMemoryKillReportSupported()`
>

### 3、Window Management的变化

#### 3.1、WM结构和功能的变化

从androidR开始，wm / wms被重构为 wm core 和 wm shell，以及附加的 wm jetpack 组件。

**wm / wms ——> wm core + wm shell + wm jetpack** 

|            | 运行所在的进程 | 功能                                                         |
| :--------- | :------------- | :----------------------------------------------------------- |
| wm core    | system server  | WM core仅包含与策略相关的代码，功能策略实施和一致性层，主线模块 |
| wm shell   | SystenUI       | 表示和定制层，AOSP包含默认实现，OEM可以定制特定的闭源产品    |
| wm jetpack | App            | API 层 WM的支持库                                            |

androidR之前：

<img src="/home/yaoyifei/图片/2020-09-10 16-22-10屏幕截图.png" alt="2020-09-10 16-22-10屏幕截图"  />

androidR以后

![2020-09-10 16-54-26屏幕截图](/home/yaoyifei/图片/2020-09-10 16-54-26屏幕截图.png)

WM核心仅包含与策略相关的代码，与呈现相关的所有内容（例如应用程序/窗口动画）都移至wm shell并在systemui中运行
Shell通过新的TaskOrganizer API与内核进行通信

#### 3.2、TaskOrganizer API

WM core 实现ITaskOrganizerController接收并执行请求。

```java
frameworks/base/core/java/android/app/ITaskOrganizerController.aidl
interface ITaskOrganizerController {
	void registerTaskOrganizer(ITaskOrganizer organizer, intwindowingMode);
	int applyContainerTransaction(in WindowContainerTransaction t, ITaskOrganizer organizer);
...
}
```

WM Shell 实现ITaskOrganizer接收来自核心的回调

```java
frameworks/base/core/java/android/view/ITaskOrganizer.aidl
oneway interface ITaskOrganizer {
	void taskAppeared(in ActivityManager.RunningTaskInfo taskInfo);
	void taskVanished(in IWindowContainer container);
	void transactionReady(int id, in SurfaceControl.Transaction t);
	void onTaskInfoChanged(in ActivityManager.RunningTaskInfo info);
}
```

这个’任务组织者‘是systemUI和system server之间的媒介

#### 3.3、新的WindowInsets

R以前：

view.setSystemUiVisibility(SYSTEM_UI_FLAG_HIDE_NAVIGATION)
view.setSystemUiVisibility(SYSTEM_UI_FLAG_IMMERSIVE)
view.setSystemUiVisibility(SYSTEM_UI_FLAG_LIGHT_STATUS_BAR)

R以后：

wic = view.getWindowInsetsController()
wic.hide(WindowInsets.Type.navigationBar())
wic.setBehavior(BEHAVIOR_SHOW_BARS_BY_SWIPE)
wic.setAppearance(APPEARANCE_LIGHT_STATUS_BAR)



