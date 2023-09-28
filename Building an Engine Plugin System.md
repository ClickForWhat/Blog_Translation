*本文档译自 bitsquid.blogspot.com 的 "Building an Engine Plugin System" 文章，作者 Niklas Frykholm*


## 概述 - *Overview*
----
插件系统是开发者扩展引擎能力的一个好方法。当然，引擎也可以直接通过修改源代码来进行扩展，但是这种方法有几个缺点：

+ 更改代码需要重新编译引擎。任何想要修改引擎的人都必须拥有完整的源代码，能够访问所有库并正确设置构建环境。
+ 每次从上游提取更改时，都必须将更改与传入的补丁合并。随着时间的推移，这将成为一项重量级的工作。
+ 由于直接在源代码中工作，而不是针对已发布的 *API*，因此重构引擎系统可能会导致从头开始重写代码。
+ 没有简单的方法可以与他人分享你所做的修改。

插件系统解决了上述难题。插件可以作为编译后的 *DLL* 分发。它们很容易共享，你可以通过把它们放在引擎的插件文件夹中来安装它们。由于插件使用显式 *API*，它们将继续与新版本的引擎一起工作（除非向后兼容性被显式破坏）。

当然，插件 *API* 不可能什么都能做，所以总有一些事情你需要通过修改引擎来完成。不过，这仍是一个很好的补充。


## 两个 API 的故事 - *A Tale of Two APIs*
---
当创造一个插件系统时，有两个 *API* 你需要考虑。

第一个，也是最明显的，就是插件向引擎所公开的 *API*：一组导出的函数，引擎将在预定义的时间调用这些函数。对于一个简单的系统，它看起来是这样的：

```C++
__declspec(dllexport) void init();
__declspec(dllexport) void update(float dt);
__declspec(dllexport) void shutdown();
```

另一个 *API* 通常要麻烦一些，它是引擎向插件公开的 *API*。

为了产生效果，插件通常需要调用引擎来做一些事情。这可以是生成一个单位，播放声音，渲染一些网格等。引擎需要为插件提供一些方法来供它调用这些服务。

有很多方法可以做到这一点。一种常见的解决方案是将所有共享功能放在一个公共 *DLL* 中，然后将引擎应用程序和插件链接到这个 *DLL*。

![[plugin_1.png]]

这种方法的缺点是，插件需要访问的功能越多，共享 *DLL* 中必须包含的功能就越多。最终，你不得不在共享 *DLL* 中放置大部分引擎功能，这与我们所追求的干净和简单的 *API* 相去甚远。

这在引擎和插件之间创建了一个非常强的耦合。每次我们想要修改引擎中的某些内容时，我们可能不得不修改共享 *DLL*，从而可能破坏所有插件。

任何读过我之前文章的人都知道，我不喜欢这种强耦合。它们是重写和重构系统的强大阻力，最终导致代码停滞不前。

另一种方法是让引擎的脚本语言（在我们的例子中是 *Lua*）作为引擎的 *API*。通过这种方法，任何时候插件想要引擎做一些事情，它可以调用 *Lua*。

对于许多应用程序，我认为这可能是一个非常好的解决方案，但在我们的情况下，它似乎不是一个完美的选择。首先，这些插件需要访问许多“低层级”的东西，而无法从 *Lua* 中访问。而且我并不热衷于将引擎的所有内部都暴露给 *Lua*。其次，由于插件和引擎都是用 *C++* 编写的，因此通过 *Lua* 编组它们之间的所有调用似乎过于复杂且效率低下。

我更喜欢一个简约的、面向数据的、基于 C 语言的接口（因为 *C++* 的 *ABI* 兼容性问题，也因为……嗯……*C++*）。


## 接口查询 - *Interface Querying*
----
我们可以在初始化插件时将引擎 *API* 发送给它，而不是将插件链接到提供引擎 *API* 的 *DLL*。像这样（一个简化的例子）：

```C++
// plugin_api.h
typedef struct EngineApi
{
 void (*spawn_unit)(World *world, const char *name, float pos[3]);
 ...
} EngineApi;
```

```C++
// plugin.h
#include "plugin_api.h"
__declspec(dllexport) void init(EngineApi *api);
__declspec(dllexport) void update(float dt);
__declspec(dllexport) void shutdown();
```

非常不错。插件开发者不需要再链接任何东西，只需要包含 *plugin_api.h* 文件即可，之后就可以调用 `EngineApi` 结构体中的函数来告诉引擎去做一些事情。

唯一不足的是版本支持。

在将来的某个时候，我们可能想要修改 *EngineApi*。也许我们发现我们想要在 `spawn_unit()` 或其他东西中添加一个旋转参数。我们可以通过在系统中引入版本控制来实现这一点。我们没有直接向插件发送引擎 *API*，而是向插件发送一个函数，让它查询特定版本的引擎 *API*。

通过这种方法，我们还可以将 *API* 分解为可以单独查询的更小的子模块。这给了我们一个更干净的结构组织。

```C++
// plugin_api.h
#define WORLD_API_ID    0
#define LUA_API_ID      1

typedef struct World World;

typedef struct WorldApi_v0 {
  void (*spawn_unit)(World *world, const char *name, float pos[3]);
  ...
} WorldApi_v0;

typedef struct WorldApi_v1 {
  void (*spawn_unit)(World *world, const char *name, float pos[3], float rot[4]);
  ...
} WorldApi_v1;

typedef struct lua_State lua_State;
typedef int (*lua_CFunction) (lua_State *L);

typedef struct LuaApi_v0 {
  void (*add_module_function)(const char *module, const char *name, lua_CFunction f);
  ...
} LuaApi_v0;

typedef void*(*GetApiFunction)(unsigned api, unsigned version);
```

当引擎实例化插件时，它会传递 `get_engine_api()`，插件可以使用它来获取不同的引擎 *API*。

插件通常会在 `init()` 函数中设置 API：

```C++
static WorldApi_v1 *_world_api = nullptr;
static LuaApi_v0 *_lua_api = nullptr;

void init(GetApiFunction get_engine_api)
{
  _world_api = (WorldApi_v1*)get_engine_api(WORLD_API, 1);
  _lua_api = (LuaApi_v0*)get_engine_api(LUA_API, 0);
}
```

之后，插件将使用这些 *API*：

```C++
_world_api->spawn_unit(world, "player", pos);
```

如果我们需要对 *API* 进行重大更改，我们可以引入该 *API* 的新版本。只要 `get_engine_api()`在被请求时仍然可以返回旧的 *API* 版本，所有现有的插件都将继续工作。

有了这个引擎的查询系统，对插件也使用相同的方法是有意义的。也就是说，与暴露单独的函数 `init()`、`update()` 等不同，这个插件可以只暴露一个函数 `get_plugin_api()`，引擎可以用同样的方式从插件中查询 *API*。

```C++
// plugin_api.h
#define PLUGIN_API_ID 2
typedef struct PluginApi_v0
{
  void (*init)(GetApiFunction get_engine_api);
  ...
} PluginApi_v0;
```

```C++
// plugin.c
__declspec(dllexport) void *get_plugin_api(unsigned api, unsigned version);
```

由于我们现在对插件 *API* 也有版本控制，这意味着我们可以在不破坏现有插件的情况下修改它（添加新的所需功能等）。

## 组合起来 - *Putting It All Together*
---
综上所述，这是一个插件的完整（但非常小）示例，它向引擎的 *Lua* 层暴露了一个新功能：

```C
// plugin_api.h
#define PLUGIN_API_ID       0
#define LUA_API_ID          1

typedef void *(*GetApiFunction)(unsigned api, unsigned version);

typedef struct PluginApi_v0
{
  void (*init)(GetApiFunction get_engine_api);
} PluginApi_v0;

typedef struct lua_State lua_State;
typedef int (*lua_CFunction) (lua_State *L);

typedef struct LuaApi_v0
{
  void (*add_module_function)(const char *module, const char *name, lua_CFunction f);
  double (*to_number)(lua_State *L, int idx);
  void (*push_number)(lua_State *L, double number);
} LuaApi_v0;
```

```C
// plugin.c
#include "plugin_api.h"

LuaApi_v0* _lua;

static int test(lua_State* L)
{
  double a = _lua->to_number(L, 1);
  double b = _lua->to_number(L, 2);
  _lua->push_number(L, a + b);
  return 1;
}

static void init(GetApiFunction get_engine_api)
{
  _lua = get_engine_api(LUA_API_ID, 0);

  if (_lua)
    _lua->add_module_function("Plugin", "test", test);
}

__declspec(dllexport) void* get_plugin_api(unsigned api_id, unsigned version)
{
  if (api_id == PLUGIN_API_ID && version == 0) {
    static PluginApi_v0 api;
    api.init = init;
    // Do Some Init...
    return &api;
  }
  return 0;
}
```

```C
// engine.c
// Initialized elsewhere.
LuaEnvironment* _env = 0;

void add_module_function(const char* module, const char* name, lua_CFunction f)
{
  _env->add_module_function(module, name, f);
}

void *get_engine_api(unsigned api, unsigned version)
{
  if (api == LUA_API_ID && version == 0 && _env) {
    static LuaApi_v0 lua;
    lua.add_module_function = add_module_function;
    lua.to_number = lua_tonumber;
    lua.push_number = lua_pushnumber;
    return &lua;
  }
  return 0;
}

void load_plugin(const char* path)
{
  HMODULE plugin_module = LoadLibrary(path);
  if (!plugin_module)
    return;
  GetApiFunction get_plugin_api = (GetApiFunction)GetProcAddress(plugin_module, "get_plugin_api");
  if (!get_plugin_api)
    return;
  PluginApi_v0* plugin = (PluginApi_v0*)get_plugin_api(PLUGIN_API_ID, 0);
  if (!plugin)
    return;
  plugin->init(get_engine_api);
}
```