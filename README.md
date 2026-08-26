# 高性能日志系统

C++17 / CMake 编写的高性能日志系统示例,聚焦于**高吞吐、加密归档、崩溃安全**:

- **结构化序列化**:日志消息以 Protobuf(`EffectiveMsg`,LITE_RUNTIME)序列化,字段化存储,支持离线字段级还原
- **流式压缩**:集成 ZSTD,消息级流式压缩(`ZSTD_e_flush`),窗口跨消息复用;重复/模板化日志场景压缩比可达 80:1(实测 2.0GB → 24MB)
- **加密**:ECDH(P-256)密钥协商 + AES-256-CBC 对称加密,每条日志独立密文归档
- **mmap 双缓冲**:master/slave 双缓存轮换(指针交换、零拷贝交接),页级动态扩容;进程崩溃后按 magic+size 头恢复未落盘数据
- **异步调度**:线程池 + 串行执行域(Strand 语义)任务调度,常规/延时/周期三类任务接口,解耦日志收集与持久化
- **离线解码器**:`decode/decoder` 支持 ECDH 解密、ZSTD 续解、字段级 pattern 渲染

## 构建

依赖均已内置(`logger/third_party/`:zstd 1.5.6、zlib 1.2.13、protobuf 21.8、cryptopp 8.9、fmt 11.0.2),
无需联网下载。构建前需用内置 `protoc` 生成 protobuf 源码(见下)。

### Linux / WSL

```bash
python3 script/build_linux.py     # 或手动:
# ./script/bin/protoc -I=logger/proto --cpp_out=build/Debug logger/proto/effective_msg.proto
# cd build/Debug && cmake ../.. -DCMAKE_BUILD_TYPE=Debug && cmake --build . --parallel 8
```

### Windows(Visual Studio)

```powershell
python script\build_windows.py
# 或手动(示例为 VS 2022 工具链;较新 MSVC 下 crypto++ 8.9 需要两处 stdext 兼容补丁,
# 本项目源码已自带该补丁):
# .\script\bin\protoc.exe -I=logger/proto --cpp_out=build\x64-Debug logger\proto\effective_msg.proto
```

## 运行示例

```bash
./build/Debug/example/logger_example          # 1M 条 x 2KB 性能基准(单机实测约 45 万条/秒 Release)
./build/Debug/example/macro_logger_example    # fmt 宏 API 版本
./build/Debug/example/crypt_example           # ECDH / AES 演示
./build/Debug/example/zstd_example            # zstd 演示
```

> 注意:`example/*.cc` 中 `conf.dir = "D:/logger"` 为 Windows 路径示例,Linux 下请改为
> 合适的目录(如 `/home/<user>/loggerlogs`)后重新构建。

## 离线解码

```bash
# <文件> <服务器私钥HEX> <输出文件>
./build/Debug/decode/decoder loggerdemo_20260826153642.log \
    FAA5BBE9017C96BF641D19D0144661885E831B5DDF52539EF1AB4790C05E665E decoded.txt
```

解码器按 `decode_formatter` 中 pattern 渲染(如 `[%l][%D:%S][%p:%t][%F:%f:%#]%v`),
支持:级别 `%l`、日期 `%D`、秒 `%S`、毫秒 `%M`、进程 `%p`、线程 `%t`、行号 `%#`、文件 `%F`、函数 `%f`、消息 `%v`。

## 目录结构

```
logger/          日志库(序列化/压缩/加密/mmap/上下文调度)
example/         4 个示例程序
decode/          离线解码器
script/          构建脚本 + 内置 protoc
logger/proto/    effective_msg.proto(消息协议)
```

## 依赖版本

- zstd 1.5.6 / zlib 1.2.13 / protobuf 21.8 / cryptopp 8.9 / fmt 11.0.2

## License

(待添加:本项目暂未包含 LICENSE 文件,开源前请确认版权归属并补充许可证。)
