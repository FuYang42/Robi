# Robi

一款 iOS AR 应用。用户可以看到当前坐标附近被渲染成像素风 3D 场景的现实世界，并在其中留下标记（文字、图片、视频）或上传现实物体（长椅、树、邮箱等），这些内容可以被所有经过同一坐标的用户看到。

---

## 技术栈

- **引擎**：Unity 6（iOS 构建）
- **AR**：ARKit（iOS 空间定位）
- **地理数据**：OpenStreetMap Overpass API（建筑/道路轮廓）
- **高程数据**：AWS Terrarium 瓦片
- **物体识别**：Apple Vision + Core ML（设备端，离线可用）
- **后端**：Firebase Firestore + Firebase Storage
- **地理查询**：GeoFirestore（GeoHash 范围查询）

---

## 核心功能

1. 加载当前坐标附近的建筑、道路、地形，渲染成像素风 3D 场景
2. 陀螺仪同步：转动设备，场景视角随之旋转
3. 用户可在场景中任意坐标留下标记（文字/图片/视频）
4. 用户可用摄像头扫描现实物体，系统识别类型后在场景中放置对应像素模型，并挂载用户照片和信息
5. 所有用户共享同一套标记和物体数据

---

## 数据流

```
GPS 定位当前坐标
    ↓
[数据获取] Overpass API → 建筑/道路轮廓 + AWS Terrarium → 地形高程
    ↓
[几何处理] 地理坐标 → Unity 世界坐标，多边形填充，建筑高度推断
    ↓
[渲染] 生成 3D 网格 → 像素风 Shader → 陀螺仪控制摄像机
    ↓
[标记层] Firebase 拉取附近标记和物体 → 渲染在对应坐标 → 点击展开内容
```

---

## Firebase 数据结构

### 用户标记 `marks/{markId}`
```
lat: number
lng: number
alt: number
type: "text" | "image" | "video"
content: string
mediaUrl: string | null
createdAt: timestamp
userId: string | null
```

### 场景物体 `objects/{objectId}`
```
lat: number
lng: number
alt: number
objectType: "bench" | "tree" | "fountain" | "mailbox" | ...
modelId: string          # 对应本地像素模型库
photoUrl: string         # 用户上传的原始照片
uploadedBy: string | null
createdAt: timestamp
```

---

## 开发阶段

### 阶段一：地基
让像素世界出现在屏幕上。

**环境搭建**
- [ ] 安装 Unity 6，配置 iOS 构建环境（Xcode、Apple Developer 账号）
- [ ] 创建 Unity 项目，导入 ARKit XR Plugin 包
- [ ] 配置 iOS 权限：定位、陀螺仪、摄像头

**地理数据获取**
- [ ] 实现 GPS 定位，获取当前经纬度
- [ ] 封装 Overpass API 请求，拉取当前坐标半径 500m 内的 building、highway、landuse 数据
- [ ] 封装 AWS Terrarium 高程瓦片请求，解码 RGB → 高程值
- [ ] 实现 OSM JSON 解析，提取 Way 节点坐标列表

**坐标转换**
- [ ] 实现地理坐标（lat/lng）→ Unity 世界坐标（x/z）的转换（参考 arnis 的 Haversine 方案）
- [ ] 实现高程 → Unity Y 轴的映射
- [ ] 实现多边形轮廓 → 方块化填充（参考 arnis 的 Flood Fill 算法）

**3D 场景生成**
- [ ] 根据 building:levels 标签推断建筑高度，生成立方体网格
- [ ] 生成道路平面网格
- [ ] 生成地形网格

**像素风渲染**
- [ ] 编写像素化 Shader（降低采样，方块化着色）
- [ ] 按建筑类型（住宅/商业/工业）分配不同颜色
- [ ] 调整美术风格到满意

**陀螺仪控制**
- [ ] 读取 IMU 数据（Input.gyro）
- [ ] 将设备旋转映射到 Unity 摄像机旋转
- [ ] 处理坐标系转换（iOS 陀螺仪坐标系 → Unity 坐标系）

---

### 阶段二：标记层
让世界有人留下的痕迹。

**Firebase 接入**
- [ ] 创建 Firebase 项目，启用 Firestore 和 Storage
- [ ] 在 Unity 中导入 Firebase SDK
- [ ] 实现匿名认证（userId 可为 null，先不做登录）

**上传标记**
- [ ] UI：长按场景中某点 → 弹出创建标记面板
- [ ] 实现文字标记上传到 Firestore
- [ ] 实现图片标记：调用系统相册/相机 → 上传到 Storage → 写入 Firestore
- [ ] 实现视频标记：调用系统相机录制 → 上传到 Storage → 写入 Firestore

**拉取并渲染标记**
- [ ] 实现 GeoHash 范围查询，拉取当前坐标附近标记
- [ ] 在场景对应坐标渲染发光像素图标
- [ ] 实现点击图标 → 展开内容面板（文字/图片/视频播放）

---

### 阶段三：扫描物体
让用户填充世界的细节。

**识别系统**
- [ ] 建立物体类型列表（初版 20 种：bench, tree, fountain, mailbox, streetlight, trashcan, sign, car, bicycle, dog, cat, flower, rock, stairs, door, window, fence, bridge, statue, phone_booth）
- [ ] 为每种类型制作对应的像素风 3D 模型
- [ ] 训练或复用 Core ML 模型，输入照片 → 输出物体类型
- [ ] 识别置信度低于阈值时，退回显示像素贴图（2D 像素化照片悬浮在坐标上）

**扫描上传流程**
- [ ] UI：点击扫描按钮 → 打开摄像头 → 拍照
- [ ] 调用 Apple Vision 检测画面中的主体物体
- [ ] 调用 Core ML 识别物体类型
- [ ] 展示识别结果，用户确认或手动修正类型
- [ ] 上传照片到 Storage，写入 objects 集合
- [ ] 在当前坐标放置对应像素模型

**渲染物体**
- [ ] 拉取附近 objects 数据，在场景中实例化像素模型
- [ ] 点击模型 → 展开信息面板（物体类型、用户照片、上传者信息）

---

### 阶段四：打磨

**性能**
- [ ] 按距离实现 LOD：近处高精度，远处低精度或不渲染
- [ ] 场景数据本地缓存，已加载区域不重复请求
- [ ] 标记和物体数据分页加载，避免一次拉取过多

**体验细节**
- [ ] 场景加载进度提示
- [ ] 弱网或无网提示
- [ ] 标记数量上限和举报机制（防止滥用）
- [ ] 调整陀螺仪灵敏度，增加平滑插值减少抖动

---

## 参考项目

- [arnis](https://github.com/louis-e/arnis)：OSM → Minecraft 像素世界，Rust 实现。坐标转换、Flood Fill、建筑高度推断等算法逻辑可直接参考移植。
