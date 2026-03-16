# 嵌入式 C 项目构建

构建嵌入式 C/C++ 项目，检测并修复构建错误：

1. **检测项目类型**：
   - Makefile 项目
   - CMake 项目
   - 平台特定项目（STM32CubeIDE、ESP-IDF 等）

2. **交叉编译工具链检测**：
   ```bash
   arm-none-eabi-gcc --version
   riscv64-unknown-elf-gcc --version
   xtensa-esp32-elf-gcc --version
   ```

3. **嵌入式项目构建**：
   ```bash
   # 清理并构建
   make clean
   make -j$(nproc)

   # 或使用 CMake
   mkdir -p build && cd build
   cmake .. -DCMAKE_TOOLCHAIN_FILE=../toolchain.cmake
   make
   ```

4. **常见问题检测**：
   - 工具链路径配置
   - 链接脚本（.ld 文件）
   - 启动文件（startup.s）
   - HAL/LL 库配置
   - 交叉编译标志

5. **链接器问题**：
   - 区域溢出（FLASH/RAM）
   - 未定义符号
   - 重复定义
   - 库依赖

6. **平台特定检查**：
   - STM32：检查 .ioc 文件配置
   - ESP32：检查 sdkconfig
   - Nordic：检查链接器脚本

如果构建失败，使用 `agent: embedded-c-build-resolver` 修复错误。
