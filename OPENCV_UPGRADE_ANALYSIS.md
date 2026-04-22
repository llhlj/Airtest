# OpenCV 升级排查文档

## 目标
将 `opencv-contrib-python` 从 `>=4.4.0.46, <=4.6.0.66` 升级到 `>=4.10.*`

## 一、需要修改的文件（高优先级）

### 1. `airtest/core/settings.py` (第16行)
**问题**: 版本检查逻辑限制了 SIFT 的使用

```python
# 当前代码
if Version('3.4.2') < Version(cv2.__version__) < Version('4.4.0'):
    CVSTRATEGY = ["mstpl", "tpl", "brisk"]
```

**原因**: 
- 当前逻辑：OpenCV 版本在 3.4.2 到 4.4.0 之间时，排除 SIFT
- OpenCV 4.4.0+ 后，SIFT 已从 contrib 移到主模块，不再需要 contrib
- 升级到 4.10.* 后，SIFT 应该可以直接使用

**建议修改**: 移除或更新此版本检查逻辑

---

### 2. `airtest/aircv/keypoint_matching_contrib.py`
**涉及方法**: 
- `check_cv_version_is_new()` (第17-20行)
- `BRIEFMatching.init_detector()` (第28-47行)
- `SIFTMatching.init_detector()` (第68-88行)
- `SURFMatching.init_detector()` (第109-127行)

**问题分析**:

#### SIFT (第76-85行)
```python
if check_cv_version_is_new():
    try:
        # opencv3 >= 3.4.12 or opencv4 >=4.5.0, sift is in main repository
        self.detector = cv2.SIFT_create(edgeThreshold=10)
    except AttributeError:
        try:
            self.detector = cv2.xfeatures2d.SIFT_create(edgeThreshold=10)
        except:
            raise NoModuleError(...)
else:
    # OpenCV2.x
    self.detector = cv2.SIFT(edgeThreshold=10)
```

**状态**: ✅ 代码已兼容 OpenCV 4.10，优先尝试 `cv2.SIFT_create()`

#### SURF (第119行)
```python
self.detector = cv2.xfeatures2d.SURF_create(self.HESSIAN_THRESHOLD, upright=self.UPRIGHT)
```

**状态**: ⚠️ SURF 仍在 contrib 模块中，需要 `opencv-contrib-python`

#### BRIEF (第34-35行)
```python
self.star_detector = cv2.xfeatures2d.StarDetector_create()
self.brief_extractor = cv2.xfeatures2d.BriefDescriptorExtractor_create()
```

**状态**: ⚠️ BRIEF 仍在 contrib 模块中，需要 `opencv-contrib-python`

---

### 3. `airtest/aircv/sift.py` (第82-91行)
**问题**: `_init_sift()` 函数的版本检查不完整

```python
def _init_sift():
    if cv2.__version__.startswith("3."):
        try:
            sift = cv2.xfeatures2d.SIFT_create(edgeThreshold=10)
        except:
            raise NoSIFTModuleError(...)
    else:
        # OpenCV2.x
        sift = cv2.SIFT(edgeThreshold=10)
```

**问题**: 
- 没有处理 OpenCV 4.x 的情况
- OpenCV 4.x 应该使用 `cv2.SIFT_create()`

**建议修改**: 添加 OpenCV 4.x 的处理逻辑

---

## 二、可能需要检查的文件（中优先级）

### 4. `airtest/aircv/keypoint_matching.py`
**内容**: KAZE/AKAZE/BRISK/ORB 特征检测器
**状态**: ✅ 这些检测器在 OpenCV 主模块中，无需 contrib

### 5. `airtest/aircv/keypoint_base.py`
**内容**: 特征点匹配基类
**状态**: ✅ 使用标准 OpenCV API，无需修改

### 6. `airtest/aircv/template_matching.py`
**内容**: 模板匹配
**状态**: ✅ 使用标准 API (`cv2.matchTemplate`, `cv2.minMaxLoc`)

### 7. `airtest/aircv/multiscale_template_matching.py`
**内容**: 多尺度模板匹配
**状态**: ✅ 使用标准 API

### 8. `airtest/aircv/cal_confidence.py`
**内容**: 置信度计算
**状态**: ✅ 使用标准 API (`cv2.matchTemplate`, `cv2.copyMakeBorder`)

---

## 三、低优先级/无需修改的文件

| 文件 | 说明 | 状态 |
|------|------|------|
| `airtest/aircv/aircv.py` | 图像读写、显示 | ✅ 标准 API |
| `airtest/aircv/utils.py` | 工具函数 | ✅ 标准 API |
| `airtest/aircv/template.py` | 模板匹配工具 | ✅ 标准 API |
| `airtest/aircv/screen_recorder.py` | 屏幕录制 | ✅ 标准 API |
| `airtest/core/cv.py` | 图像识别入口 | ✅ 间接使用 |

---

## 四、OpenCV 4.10 API 变更总结

### SIFT 状态
- OpenCV 3.x: 在 `cv2.xfeatures2d` (contrib)
- OpenCV 4.4.0+: 移至主模块 `cv2.SIFT_create()`
- **结论**: 升级后 SIFT 更容易使用

### SURF 状态
- 所有版本都在 `cv2.xfeatures2d` (contrib)
- **结论**: 仍需要 opencv-contrib-python

### BRIEF 状态
- 所有版本都在 `cv2.xfeatures2d` (contrib)
- **结论**: 仍需要 opencv-contrib-python

### 其他检测器 (KAZE/AKAZE/BRISK/ORB)
- 主模块 API: `cv2.KAZE_create()`, `cv2.AKAZE_create()`, `cv2.BRISK_create()`, `cv2.ORB_create()`
- **结论**: 无需 contrib

---

## 五、修改建议汇总

### 必须修改
1. **settings.py 第16行**: 移除或更新版本检查逻辑
2. **sift.py `_init_sift()`**: 添加 OpenCV 4.x 支持

### 建议修改
3. **keypoint_matching_contrib.py**: 简化 SIFT 初始化逻辑（移除对 `xfeatures2d.SIFT_create` 的 fallback）

### 无需修改
- 其他使用标准 OpenCV API 的文件

---

## 六、测试建议

升级后需测试的功能点：
1. SIFT 特征匹配
2. SURF 特征匹配（需要 contrib）
3. BRIEF 特征匹配（需要 contrib）
4. 模板匹配
5. 多尺度模板匹配
6. 图像读写功能

---

## 七、依赖关系

```
requirements.txt 修改:
- opencv-contrib-python>=4.4.0.46, <=4.6.0.66
+ opencv-contrib-python>=4.10.*
```

**注意**: 如果只需要 SIFT，可以考虑只安装 `opencv-python`（主模块），不需要 contrib
