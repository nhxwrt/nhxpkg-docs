# NHX `.mk` 文件机制

NHX 用两个目录放 `.mk` 文件，按用途分类：

| 板块 | 路径 | 加载方式 | 用途 |
|------|------|---------|------|
| [1. 替代](#1-替代) | `openwrt-patches/nhxpkg/<PKG_NAME>/<PKG_NAME>.mk` | FindPackage 自动加载 | 替换 / 修改 stock 包 |
| [2. Helper 宏](#2-helper-宏) | `qsdk-package/nhxpkg/<DIR>/<FILE>.mk` | nhxpkg 内 Makefile 直接 `include` | 跨包共用的 make 宏 |

补丁放 `patches/` 子目录自动加载，不需要 `.mk`。

---

## 1. 替代

替换 stock 包有两种模式：

### 1.1 抑制模式 — 替换包独立存在

替换包是独立 Makefile（如 nhxuhttpd 替代 stock uhttpd），分两步：

**主仓侧** `openwrt-patches/nhxpkg/<stock>/<stock>.mk` —— 把 stock 编成空壳（不出 `.ipk`）：

```makefile
define Package/<stock>
  $(Package/<stock>/default)
  BUILDONLY:=1
endef

define Package/<stock>-mod-xxx
  $(Package/<stock>/default)
  TITLE+= (xxx)
  BUILDONLY:=1
endef

define Build/Prepare
  mkdir -p $(PKG_BUILD_DIR)
endef
define Build/Configure
endef
define Build/Compile
endef
define Build/Install
endef

define Package/<stock>/install
  true
endef
define Package/<stock>-mod-xxx/install
  true
endef
```

**替换包侧** —— 替换包自己 Makefile 加 `PROVIDES` + `CONFLICTS`：

```makefile
define Package/nhxuhttpd
  $(Package/nhxuhttpd/default)
  DEPENDS:=...
  PROVIDES:=uhttpd
  CONFLICTS:=uhttpd
endef

define Package/nhxuhttpd-mod-ubus
  $(Package/nhxuhttpd/default)
  DEPENDS:=nhxuhttpd ...
  PROVIDES:=uhttpd-mod-ubus
  CONFLICTS:=uhttpd-mod-ubus
endef
```

- `PROVIDES:=<stock>` —— opkg 把替换 `.ipk` 当 stock 等价物，满足其他包 `+<stock>` 依赖
- `CONFLICTS:=<stock>` —— 阻止两者同时安装

**要点**：

- 必须展开 `$(Package/<stock>/default)` —— `package.mk:282` 要求 `TITLE / CATEGORY / SECTION / VERSION` 非空
- Build 四阶段（`Prepare` / `Configure` / `Compile` / `Install`）全部覆盖，否则 cmake 包（`include cmake.mk` → `PKG_INSTALL:=1`）会跑 `make install` 撞空目录
- BUILDONLY 子包列表必须 1:1 对齐 stock Makefile 的 `BuildPackage` 列表，漏一个 → 那个子包仍编出 `.ipk` 撞包
- 替换包对每个 stock 子包都要写 `PROVIDES + CONFLICTS`（如 `uhttpd-mod-lua` / `uhttpd-mod-ubus`）

### 1.2 源码替换模式 —— 改动小，注入到 stock 包

```makefile
NHX_DIR:=$(TOPDIR)/nhxpkg/nhxbase/xxx

define Build/Prepare
  mkdir -p $(PKG_BUILD_DIR)
  $(CP) $(NHX_DIR)/src/* $(PKG_BUILD_DIR)/
endef

define Package/<stock>
  $(Package/<stock>/default)
  DEPENDS:=+原有依赖 +新增依赖
endef

define Package/<stock>/install
  $(INSTALL_DIR) $(1)/usr/sbin
  $(INSTALL_BIN) $(PKG_BUILD_DIR)/xxx $(1)/usr/sbin/xxx
endef
```

`nhxpkg` / `nhxpkg_bin` 自动探测：

```makefile
NHX_DIR:=$(if $(wildcard $(TOPDIR)/nhxpkg/nhxbase/xxx/src),\
  $(TOPDIR)/nhxpkg/nhxbase/xxx,\
  $(TOPDIR)/nhxpkg_bin/nhxbase/xxx)
```

### 1.3 速查

| 字段 | ✓ / ✗ | 说明 |
|------|:---:|------|
| `define Package/xxx` | ✓ | 最后 define 胜出 |
| `define Build/Prepare\|Configure\|Compile\|Install` | ✓ | 各 recipe 阶段 |
| `define Package/xxx/install` | ✓ | 同上 |
| `BUILDONLY:=1` | ✓ | 抑制 `.ipk` 生成 |
| `PROVIDES:=<stock>` | ✓ | 替换包必备 |
| `CONFLICTS:=<stock>` | ✓ | 替换包必备 |
| `PKG_BUILD_DEPENDS:=` | ✓ | 简单变量后赋值覆盖 |
| `DEPENDS:=`（裸赋值） | ✗ | 被 `Package/xxx` 内的赋值覆盖 |
| `CATEGORY:=`（清空） | ✗ | `package.mk:282` 要求非空 |
| `USE_SOURCE_DIR:=` | ✗ | `ifdef` 早于 `BuildPackage` 调用 |

### 1.4 当前清单

| stock | 模式 | 替换包 | 文件 |
|-------|------|--------|------|
| uhttpd | 抑制 | nhxuhttpd | `openwrt-patches/nhxpkg/uhttpd/uhttpd.mk` |

---

## 2. Helper 宏

跨包共用的 make 宏文件，nhxpkg 内 Makefile 直接 `include` 用：

```makefile
# 用法（在 nhxpkg 内某个 Makefile 顶部）
include $(TOPDIR)/qsdk-package/nhxpkg/<DIR>/<FILE>.mk
```

不走 FindPackage，纯 GNU make `include`。文件名不要求等于目录名。

### 2.1 当前清单

| 文件 | 提供 | 被谁 include | 用途 |
|------|------|--------------|------|
| `qsdk-package/nhxpkg/nhxlua/nhx-lua.mk` | `NhxLuaCompile`、`NhxLuaCompileFile` | `nhxbase/{uapi,nhxcfg}/Makefile` | 调 `nhxluac`（host 工具）把 `.lua` 编成混淆字节码 |
