# nobodywho-godot-lora

基于 [NobodyWho](https://github.com/nobodywho-ooo/nobodywho) **Godot 插件 13.0.0** 的修改版，增加一个功能：**每个 `NobodyWhoChat` 可以挂自己的 LoRA 补丁**。几个对话共用同一个模型，各自用不同的补丁。

## 这是做什么的

[NobodyWho](https://github.com/nobodywho-ooo/nobodywho) 是一个让 Godot 游戏在玩家电脑上本地运行大语言模型的插件，不用联网，也不用调用 API。

**LoRA 补丁**是给基础模型加的小号"性格/风格包"，通常只有几十 MB，能让同一个模型说话更像某个角色。原版插件一个模型只能挂一套补丁，所有对话一起生效。

这个修改版让**每个对话单独挂自己的补丁**，适合下面这种情况：

- 游戏里有好几个 NPC，想共用一个基础模型来省显存，但每个 NPC 的说话风格不同
- 同一个角色在不同年龄、不同阶段用不同的补丁
- 有的对话要用补丁，有的对话（比如判断、打分）要用干净的原模型

不挂补丁时，表现和官方 13.0.0 完全一样。

官方 13.0.0 自带的语音模块也都保留，可以照常使用：语音识别（NobodyWhoSpeechToText）、语音合成（NobodyWhoTextToSpeech）、语音活动检测（NobodyWhoVoiceActivityDetection）。用法见 [官方文档](https://github.com/nobodywho-ooo/nobodywho/tree/main/docs/docs-godot)。

适用平台：Windows x86_64，Vulkan 显卡加速。这是非官方修改版。

> English summary at the bottom.

## 下载

到本仓库的 **Releases** 页面下载 `libnobodywho-godot-x86_64-pc-windows-msvc-release.dll`。只有 Windows x86_64 版本，使用 Vulkan 显卡加速。

| 项目 | 值 |
|---|---|
| 文件大小 | 143,561,728 字节（约 143 MB） |
| SHA256 | `2DA4CCA108907878515A0D7B81D976E19F016EC08E562EE89E58A6278ECC57A7` |

下载后可以在 PowerShell 里核对文件是否完整：

```powershell
Get-FileHash .\libnobodywho-godot-x86_64-pc-windows-msvc-release.dll -Algorithm SHA256
```

## 安装

1. 先在项目里装好原版 NobodyWho 13.0.0 插件（`addons/nobodywho/`）。
2. **关掉 Godot**，然后用下载的 DLL 覆盖 `addons/nobodywho/` 里的同名文件。
3. `nobodywho.gdextension` 不需要改。本仓库也放了一份，内容和原版一样。

还原：把原版 DLL 复制回去，覆盖同名文件即可。

## 用法

```gdscript
var model = await NobodyWhoModel.create("res://models/your-model.gguf", {})

# 不挂补丁：和原版完全一样
var plain = await NobodyWhoChat.create(model, {"system_prompt": "..."})

# 挂补丁：只对这个对话生效
var npc_a = await NobodyWhoChat.create(model, {
    "system_prompt": "...",
    "lora_path": "res://lora/npc_a.gguf",     # LoRA 补丁文件（GGUF 格式）
    "lora_scale": 1.0,                         # 补丁强度，可不写，默认 1.0
})
```

- `lora_path` 为空或不写时，就是不挂补丁。
- 同一个模型下，同一个补丁文件**只加载一次**，多个对话可以共用。
- 补丁必须是给同一个基础模型训练的 GGUF 格式 LoRA（可用 llama.cpp 的 `convert_lora_to_gguf.py` 转换）。
- 文件不存在或加载失败时，`NobodyWhoChat.create` 会报错，说明是哪个文件出了问题。

## 测试结果

测试环境：RTX 4050 Laptop（6 GB），Qwen3.5 2B q4_k_m，n_ctx 4096，temperature 0（贪心采样）。测试用的补丁是随机生成的 r8、r16、r32 假补丁，只用来验证功能，不代表真实补丁的效果。

| 测试 | 结果 |
|---|---|
| 不挂补丁时，和原版 DLL 的输出是否一致 | 一字不差 ✅ |
| 挂补丁后输出是否改变 | 改变 ✅ |
| 两个对话挂同一个补丁，输出是否一致 | 一致 ✅ |
| 加载过补丁后，不挂补丁的对话是否受影响 | 不受影响 ✅ |
| JSON schema 约束输出 | 正常 ✅ |
| 3 个对话各挂不同补丁，轮流生成 2 轮 | 正常 ✅ |
| 同一个补丁只加载一次 | 是 ✅ |
| 一个真实游戏项目的自动测试，换 DLL 前后对比 | 结果相同，模型回复一字不差 ✅ |

显存占用（取自 llama.cpp 日志）：

| 项目 | 显存 |
|---|---|
| 模型本体（2B q4_k_m） | 约 1205 MB |
| r8 / r16 / r32 补丁 | 约 25 / 51 / 101 MB（每个模型只算一次） |
| 每多开一个对话（n_ctx 4096） | 约 553 MB（对话记忆 48 MB + 计算缓冲 505 MB） |

注意：显存大头是"每个对话"，补丁本身很小。原版插件也是这样。

## 自己编译

需要准备：

- Rust（stable，测试用 1.99.1，目标 `x86_64-pc-windows-msvc`）
- LLVM / libclang（测试用 23.1.5），设置 `LIBCLANG_PATH` 指向 LLVM 的 `bin` 目录
- Vulkan SDK **1.4.313.2**，设置 `VULKAN_SDK`
- Visual Studio 2022 的 C++ 工具（自带 CMake 和 Ninja）

步骤：

```powershell
mkdir work; cd work
git clone https://github.com/nobodywho-ooo/nobodywho.git
git -C nobodywho checkout nobodywho-godot-v13.0.0
git clone https://github.com/nobodywho-ooo/llama-cpp-rs.git
git -C llama-cpp-rs checkout 22e58741ca94a78abf43ffd1f74763b6945d312f
git -C llama-cpp-rs submodule update --init --recursive

git -C nobodywho apply ..\patches\nobodywho-lora.patch
git -C llama-cpp-rs apply ..\patches\llama-cpp-rs-lora.patch
```

`nobodywho` 和 `llama-cpp-rs` 两个文件夹必须**并排放**，补丁按相对路径 `../../../llama-cpp-rs/llama-cpp-2` 找引擎库。

然后在 "x64 Native Tools Command Prompt for VS 2022" 环境里编译：

```powershell
$env:CMAKE_GENERATOR = "Ninja"
$env:NOBODYWHO_SKIP_CODEGEN = "1"
$env:RUSTFLAGS = "-l Advapi32 -C debuginfo=0"
cd nobodywho\nobodywho
cargo build -p nobodywho-godot --release
```

产物是 `target\release\nobodywho_godot.dll`，改名为 `libnobodywho-godot-x86_64-pc-windows-msvc-release.dll` 后使用。第一次编译大约需要 10 多分钟。

Windows 路径太长时，可以只对本次编译开启长路径：`$env:GIT_CONFIG_COUNT="1"; $env:GIT_CONFIG_KEY_0="core.longpaths"; $env:GIT_CONFIG_VALUE_0="true"`。

国内网络可以用 rsproxy 镜像，在 `%CARGO_HOME%\config.toml` 里写：

```toml
[source.crates-io]
replace-with = "rsproxy-sparse"

[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"

[net]
git-fetch-with-cli = true
retry = 5
```

Rust 本身可以设置 `RUSTUP_DIST_SERVER=https://rsproxy.cn`、`RUSTUP_UPDATE_ROOT=https://rsproxy.cn/rustup` 后再安装。

## 改了什么

- `patches/nobodywho-lora.patch`：NobodyWho 部分（约 70 行）。`ChatConfig` 新增 `lora` 字段；`Model` 里缓存已加载的补丁；Godot 的 `NobodyWhoChat.create` 新增 `lora_path` 和 `lora_scale` 参数；引擎库依赖改为本地并排的 llama-cpp-rs。
- `patches/llama-cpp-rs-lora.patch`：llama-cpp-2 部分（约 56 行）。新增 `LlamaModel::lora_adapter_init_shared` 和 `LlamaContext::lora_adapter_set_shared`，让一个补丁能被多个对话共享。

## 许可证

- NobodyWho 使用 **EUPL-1.2** 许可证，本修改版同样以 EUPL-1.2 发布，见 [LICENSE](LICENSE)。源码改动全部在 `patches/` 里。
- `llama-cpp-rs-lora.patch` 修改的是 llama-cpp-rs，它使用 MIT 或 Apache-2.0 许可证，这个补丁也按原许可证提供。
- 这是非官方修改版，与 NobodyWho 官方无关。

---

## English

Unofficial build of the [NobodyWho](https://github.com/nobodywho-ooo/nobodywho) Godot plugin **13.0.0** (Windows x86_64, Vulkan) that adds a **per-chat LoRA adapter**:

```gdscript
var chat = await NobodyWhoChat.create(model, {"lora_path": "path/to/adapter.gguf", "lora_scale": 1.0})
```

- Several chats can share one `NobodyWhoModel`, each with its own adapter (or none).
- Each adapter file is loaded once per model.
- The voice modules of 13.0.0 (speech-to-text, text-to-speech, voice activity detection) are included unchanged.
- Without `lora_path`, output is identical to the official 13.0.0 build (tested at temperature 0).
- Download the DLL from **Releases** (SHA256 above) and replace `addons/nobodywho/libnobodywho-godot-x86_64-pc-windows-msvc-release.dll`.
- Source changes are in `patches/`. Apply them to `nobodywho` at tag `nobodywho-godot-v13.0.0` and `llama-cpp-rs` at commit `22e5874`, cloned side by side, then build with Vulkan SDK 1.4.313.2.
- License: EUPL-1.2 (same as NobodyWho). The llama-cpp-rs patch is MIT OR Apache-2.0.
