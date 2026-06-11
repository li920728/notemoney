# 记账 App - Android APK 编译指南

## 环境要求

1. **Java JDK 11** 或更高版本
   - 下载地址：https://adoptium.net/

2. **Android Studio** (可选，用于真机调试)
   - 下载地址：https://developer.android.com/studio

## 一键编译 APK

### Windows

打开命令提示符（CMD）或 PowerShell，进入项目目录后执行：

```bash
# 1. 进入项目目录
cd expense-tracker

# 2. 安装依赖（如果还没有）
npm install

# 3. 构建网页版
npm run build

# 4. 同步到 Android
npx cap sync android

# 5. 编译 APK
cd android
gradlew.bat assembleDebug

# 6. 编译完成后，APK 位置：
# android/app/build/outputs/apk/debug/app-debug.apk
```

### macOS / Linux

```bash
# 1. 进入项目目录
cd expense-tracker

# 2. 安装依赖
npm install

# 3. 构建网页版
npm run build

# 4. 同步到 Android
npx cap sync android

# 5. 编译 APK
cd android
./gradlew assembleDebug

# 6. APK 位置：
# android/app/build/outputs/apk/debug/app-debug.apk
```

## 安装到手机

1. 通过 USB 将 APK 文件传到手机
2. 在手机上打开文件管理器
3. 点击 `app-debug.apk` 安装

## 常见问题

### Q: 提示找不到 Java
A: 确保已安装 JDK 并设置了 JAVA_HOME 环境变量

### Q: gradlew.bat 无法运行
A: 在 Android 目录右键，选择"在终端中打开"，然后执行命令

### Q: 编译太慢
A: 第一次编译会下载 dependencies，后续会快很多

## 快速测试

如果只是想快速测试，可以直接用手机浏览器访问网页版：
1. 部署到任何 Web 服务器
2. 手机浏览器打开
3. 点击"添加到主屏幕"

效果和原生 App 一样！
