# Qt C++ 项目构建

构建 Qt C++11 项目，检测并修复构建错误：

1. **检测项目类型**：
   - qmake 项目（.pro 文件）
   - CMake 项目（CMakeLists.txt）
   - Qbs 项目（.qbs 文件）

2. **Qt 项目构建**：
   ```bash
   # qmake 项目
   qmake -r project.pro
   make -j$(nproc)

   # CMake 项目
   mkdir -p build && cd build
   cmake ..
   make -j$(nproc)
   ```

3. **常见问题检测**：
   - Qt 库路径配置
   - MOC/UIC/RCC 工具可用性
   - 模块依赖（QtCore, QtGui, QtWidgets）
   - 编译器工具链（qmake/moc/uic/rcc）

4. **Qt 特定检查**：
   - .pro 文件配置
   - QT += 声明
   - DEFINES 变量
   - INCLUDEPATH 配置
   - LIBS 配置

5. **错误解决**：
   - 缺失库链接
   - MOC 生成失败
   - 头文件缺失
   - Qt 版本不匹配

如果构建失败，使用 `agent: qt-build-resolver` 修复错误。
