# NHX `.mk` 文件机制

NHX 的 `.mk` 文件按用途分两类，分别放在不同目录、用不同方式加载：

| 板块 | 路径 | 加载方式 | 用途 |
|------|------|---------|------|
| [替代](#1-替代) | `openwrt-patches/nhxpkg/<PKG_NAME>/<PKG_NAME>.mk` | FindPackage 自动加载（`Build/openwrt-patches` hook） | 抑制 / 修改 stock 包 |
| [Helper 宏](#2-helper-宏) | `qsdk-package/nhxpkg/<DIR>/<FILE>.mk` | nhxpkg 内 Makefile 直接 `include`（不走 FindPackage） | 给 nhxpkg 自家包用的辅助 make 宏 |

补丁放 `patches/` 子目录自动加载，不需要 `.mk`。

---

## 1. 替代

零侵入修改官方包构建。官方代码不动，靠 `.mk` override stock 包定义。两种模式：

### 1.1 抑制模式

替换包有自己独立 Makefile（如 nhxuhttpd 替代 uhttpd），`.mk` 让 stock 变空壳：

```makefile
# openwrt-patches/nhxpkg/<PKG_NAME>/<PKG_NAME>.mk

define Package/<PKG_NAME>
  $(Package/<PKG_NAME>/default)
  BUILDONLY:=1
endef

define Package/<PKG_NAME>-mod-xxx
  $(Package/<PKG_NAME>/default)
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

define Package/<PKG_NAME>/install
  true
endef

define Package/<PKG_NAME>-mod-xxx/install
  true
endef
```

要点：

- `BUILDONLY:=1` 隐藏 kconfig，保留 make target
- 必须展开 `/default`（`:282` 要求 TITLE/CATEGORY/SECTION/VERSION 非空）
- Build 四阶段全部覆盖（stock 若 `include cmake.mk` 则 `PKG_INSTALL:=1`，`Build/Install` 不覆盖会跑 `make install`）
- BUILDONLY 列表必须 1:1 对齐当前版本上游 `BuildPackage` 列表，漏一个 = 那个子包仍编出来撞包

### 1.2 源码替换模式

改动小、不值得独立包时，直接注入 NHX 源码到 stock 包：

```makefile
NHX_DIR:=$(TOPDIR)/nhxpkg/nhxbase/xxx

define Build/Prepare
  mkdir -p $(PKG_BUILD_DIR)
  $(CP) $(NHX_DIR)/src/* $(PKG_BUILD_DIR)/
endef

define Package/<PKG_NAME>
  $(Package/<PKG_NAME>/default)
  DEPENDS:=+原有依赖 +新增依赖
endef

define Package/<PKG_NAME>/install
  $(INSTALL_DIR) $(1)/usr/sbin
  $(INSTALL_BIN) $(PKG_BUILD_DIR)/xxx $(1)/usr/sbin/xxx
endef
```

nhxpkg / nhxpkg_bin 自动探测（仅此模式需要）：

```makefile
NHX_DIR:=$(if $(wildcard $(TOPDIR)/nhxpkg/nhxbase/xxx/src),\
  $(TOPDIR)/nhxpkg/nhxbase/xxx,\
  $(TOPDIR)/nhxpkg_bin/nhxbase/xxx)
```

### 1.3 禁忌

| 做法 | 后果 |
|------|------|
| 替换包用 `PROVIDES:=<stock>` | `metadata.pl` 给 stock 加 `vdepends` → make target 消失 |
| 替换包用 `CONFLICTS:=<stock>` | kconfig 排除 stock → make target 消失 |
| 清空 `CATEGORY` | `package.mk:282` 报 missing TITLE |
| 漏覆盖 `Build/Install` | cmake 包有 `PKG_INSTALL:=1` → 空目录跑 `make install` 失败 |

### 1.4 覆盖能力速查

| 方式 | ? | 说明 |
|------|:-:|------|
| `define Package/xxx` | ✅ | 最后 define 胜出 |
| `define Build/Prepare\|Configure\|Compile\|Install` | ✅ | recipe 阶段执行 |
| `define Package/xxx/install` | ✅ | 同上 |
| `PKG_BUILD_DEPENDS:=` | ✅ | 简单变量后赋值覆盖 |
| `BUILDONLY:=1` | ✅ | 隐藏 kconfig，保留 make target |
| `DEPENDS:=`（裸赋值） | ❌ | 被 `Package/xxx` 内赋值覆盖 |
| `CATEGORY:=`（清空） | ❌ | `:282` 要求非空 |
| `USE_SOURCE_DIR:=` | ❌ | `ifdef` 在 `:83`，早于 BuildPackage |

### 1.5 当前清单

| 包 | 模式 | 替换包 | 文件 |
|----|------|--------|------|
| uhttpd | 抑制 | nhxuhttpd | `openwrt-patches/nhxpkg/uhttpd/uhttpd.mk` |

---

## 2. Helper 宏

跨包共用的 make 宏文件，**不动 stock 包**，只给 nhxpkg 自家包的 Makefile 直接 `include` 用。

```makefile
# 用法 (在 nhxpkg 内某个 Makefile 顶部)
include $(TOPDIR)/qsdk-package/nhxpkg/<DIR>/<FILE>.mk
```

文件名不要求等于目录名（如 `nhxlua/nhx-lua.mk`）。不走 FindPackage，纯 GNU make `include`，写错路径会直接编译断在 `include` 行。

### 2.1 当前清单

| 文件 | 提供 | 被谁 include | 用途 |
|------|------|--------------|------|
| `qsdk-package/nhxpkg/nhxlua/nhx-lua.mk` | `NhxLuaCompile`、`NhxLuaCompileFile` | `nhxbase/{uapi,nhxcfg}/Makefile` | 调 `nhxluac`（host 工具，nhxlua 包提供）把 `.lua` 编成混淆字节码装到目标 rootfs |
