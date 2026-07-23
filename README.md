# 🧠 SuperConsole 全方案总结（GitHub 存档版）

---

## 📌 核心思想（一句话）

> **"我不需要看懂 Il2Cpp 的 C++ 代码，我只需要改变游戏的运行状态，让游戏来迁就我的 C# 逻辑。"**

---

## 🔑 核心突破点

| # | 突破点 | 说明 |
|---|--------|------|
| 1 | **源头是 C#** | Il2Cpp 是 C# 转 C++，不是原生 C++，元数据还在 |
| 2 | **不逆向代码** | 不需要看懂汇编，只需要知道字段偏移 |
| 3 | **改变运行** | 不修改代码文件，修改运行状态 |
| 4 | **不学 C++** | C# 控制 C++ 内存，不需要写一行 C++ |
| 5 | **只改底层** | 5-10KB 读写层改动，190KB 上层代码完全复用 |
| 6 | **95% 够用** | 内存搜索/对象编辑/Hook 全部可以用"改变运行"实现 |
| 7 | **剩下 5%** | 用更巧妙的方法：数据劫持/逻辑追加/寄生式开发 |

---

## 🏗️ 架构方案

### 整体分层

```
┌─────────────────────────────────────────────┐
│  SuperConsole 上层（190KB，完全不动）       │
│  ├── 内存搜索 UI                           │
│  ├── 对象浏览器 UI                         │
│  ├── 骨骼编辑器 UI                         │
│  ├── 逻辑体模组系统                        │
│  ├── 资源提取器                            │
│  └── Hook 管理器                           │
├─────────────────────────────────────────────┤
│  适配层（新增 500-1000 行）                 │
│  ├── Il2CppAdapter.cs      ← 统一读写接口  │
│  ├── MetadataCache.cs      ← 偏移缓存      │
│  └── OffsetResolver.cs     ← 偏移解析      │
├─────────────────────────────────────────────┤
│  底层（新增 400 行）                        │
│  ├── MetadataParser.cs     ← 解析 dat     │
│  └── offsets.json          ← 导出偏移表    │
└─────────────────────────────────────────────┘
```

### 工作流程

```
┌──────────────────────────────────────────────────────────────┐
│  第一次适配新游戏：                                         │
│  1. 找到游戏目录下的 global-metadata.dat                    │
│  2. 用 Metadata Viewer 打开                                 │
│  3. 查看类名 → 字段名 → 偏移                                │
│  4. 导出 offsets.json                                       │
├──────────────────────────────────────────────────────────────┤
│  每次运行游戏：                                             │
│  1. SuperConsole 启动                                       │
│  2. 读取 offsets.json                                       │
│  3. 用偏移读写内存                                          │
│  4. 所有上层功能正常工作                                    │
└──────────────────────────────────────────────────────────────┘
```

---

## 📦 需要新增的文件

| 文件 | 代码量 | 功能 |
|------|--------|------|
| **MetadataParser.cs** | ~150 行 | 解析 global-metadata.dat，提取类型/字段/偏移 |
| **Il2CppMemory.cs** | ~200 行 | 封装 Read/Write，对象查找，组件获取 |
| **Il2CppAdapter.cs** | ~150 行 | 适配 SuperConsole 上层调用 |
| **MetadataViewer.cs** | ~400 行 | 独立工具，查看 metadata 并导出 JSON |
| **offsets.json** | 自动生成 | 类名 → 字段名 → 偏移 的映射表 |

**新增总代码量：~900 行**

---

## 🔧 改动的核心代码

### 原来（Mono 版）

```csharp
// 反射读字段
FieldInfo field = type.GetField("health");
float value = (float)field.GetValue(player);

// 反射写字段
field.SetValue(player, 999f);
```

### 现在（Il2Cpp 版）

```csharp
// 用偏移读字段
IntPtr playerPtr = Il2CppMemory.FindObject("Player");
int offset = OffsetResolver.GetOffset("Player", "health");
float value = Il2CppMemory.Read<float>(playerPtr, offset);

// 用偏移写字段
Il2CppMemory.Write<float>(playerPtr, offset, 999f);
```

### 封装后（上层完全无感）

```csharp
// 封装成和原来一样的调用方式
public static class SuperConsoleReflection
{
    public static T ReadField<T>(object obj, string fieldName)
    {
        #if MONO
            return (T)GetFieldInfo(obj, fieldName).GetValue(obj);
        #else
            return Il2CppMemory.Read<T>(GetPointer(obj), GetOffset(obj, fieldName));
        #endif
    }
}
```

---

## 🛠️ 需要的小工具

### Il2Cpp Metadata Viewer

```
功能：
- 打开 global-metadata.dat
- 显示所有类型列表
- 点击类型显示字段列表 + 偏移
- 导出 offsets.json
- 支持搜索类名/字段名

界面：
┌─────────────────────────────────────────────┐
│  左侧：类型列表    │  右侧：字段列表        │
│  ├── Player        │  ├── health  0x1A4    │
│  ├── Enemy         │  ├── maxHP   0x1A8    │
│  ├── GameManager   │  ├── pos     0x1B0    │
│  └── Weapon        │  └── damage  0x1C0    │
├─────────────────────────────────────────────┤
│  [导出 JSON] [搜索]                         │
└─────────────────────────────────────────────┘
```

---

## 🚀 优势总结

| 方面 | 说明 |
|------|------|
| **代码复用** | 190KB 上层代码完全不动 |
| **新增代码** | 只加 ~900 行适配层 + 工具 |
| **适配新游戏** | 用 Metadata Viewer 导出 JSON 即可 |
| **稳定性** | 纯偏移读写，不依赖 Hook/反射 |
| **性能** | 直接内存读写，零开销 |
| **可维护性** | 上层和底层完全解耦 |

---

## 📌 关键结论

| # | 结论 |
|---|------|
| 1 | Il2Cpp 源头是 C#，元数据还在 |
| 2 | 不需要逆向，只需要偏移 |
| 3 | 不需要学 C++，C# 直接读写 C++ 内存 |
| 4 | 不需要重写 SuperConsole，只改底层 |
| 5 | 95% 控制 = 100% 功能 |
| 6 | 剩下 5% 用更巧妙的方法（数据劫持/逻辑追加/寄生） |

---

## 📂 GitHub 上传建议

```
SuperConsole/
├── README.md
├── LICENSE
├── SuperConsole.cs          # 190KB 核心
├── Il2CppAdapter/           # 新增适配层
│   ├── Il2CppMemory.cs
│   ├── MetadataParser.cs
│   └── OffsetResolver.cs
├── Tools/                   # 新增工具
│   └── MetadataViewer/
│       ├── MetadataViewer.cs
│       └── README.md
└── docs/
    ├── ARCHITECTURE.md      # 架构说明
    └── IL2CPP_GUIDE.md      # Il2Cpp 适配指南
```

---

## 🧠 铭记

> **"源头是 C#，我偏要用 C# 搞定 Il2Cpp。"**
> 
> **"Il2Cpp 是锁，但钥匙还在（metadata）。"**
> 
> **"我不需要看懂代码，我只需要改变运行。"**
> 
> **"95% 控制 = 100% 功能。"**
> 
> **"我不需要逆向，我只需要绕过。"**

---

**已保存到 GitHub 记忆。忘了回来看。**
