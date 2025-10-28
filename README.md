> [【从UnityURP开始探索游戏渲染】](https://github.com)**专栏-直达**

置换贴图（Displacement Map）是一种通过修改顶点位置来实现表面凹凸细节的技术，与法线贴图仅影响光照不同，它直接改变几何形状，适用于需要真实物理变形的场景。

# **技术原理与解决的问题**

## ‌**功能差异**‌：

* 置换贴图通过灰度值控制顶点位移（白色凸起，黑色凹陷），相比法线贴图能产生真实的轮廓阴影和遮挡效果，解决了低模表现高精度几何细节的难题。

## ‌**性能权衡**‌：

* 需要细分曲面（Tessellation）支持，计算开销大于法线贴图，但视觉效果更真实。
  + [曲面细分参看](https://github.com)

# **历史发展节点**

## ‌**早期阶段**‌：

* 传统Shader Model 5.0引入硬件细分曲面，Unity通过自定义Shader实现置换效果。

## ‌**URP集成**‌：

* URP 7.0+原生支持Decal Projector组件，简化了贴花（含置换效果）的投影流程。
  + [Decal Projector 参看](https://github.com):[大象加速器](https://daxiangjsq.com	)

## ‌**优化演进**‌：

* 结合SRP Batcher减少Draw Call，提升实例化渲染效率。

# 原理示意图：

* 置换流程：贴图采样 → 顶点偏移 → 细分曲面 → 像素着色
* 优化链路：SRP Batcher合并材质 → GPU Instancing处理动态实例

# **URP实现步骤**

## ‌**材质配置**‌：

* 创建URP Lit Shader Graph，添加`Tessellation`和`Displacement`节点。
* 连接置换贴图（R通道）到顶点偏移量，调整细分级。

## ‌**组件绑定**‌：

* 创建Decal Projector GameObject
* 分配置换材质
* 设置投影范围（Size/Depth）和剔除层级

## ‌**脚本控制**‌（可选）：

```
|  |  |
| --- | --- |
|  | csharp |
|  | // 动态修改置换强度 |
|  | void Update() { |
|  | decalProjector.material.SetFloat("_DisplacementScale", intensity); |
|  | } |
```

# **弹孔效果实现**

* **血迹/弹痕**‌：通过Decal Projector动态投射到场景物体，结合碰撞检测确定UV坐标。

## ‌**贴图生成**‌：

* 使用Substance Designer或Photoshop绘制灰度图，白色区域表示弹孔凹陷深度。

## ‌**Shader Graph配置**‌：

* 添加`Parallax Occlusion Mapping`节点模拟深度偏移。
* 混合置换与法线贴图增强细节。

## ‌**性能优化**‌：

* 启用GPU Instancing减少相同材质的Draw Call。
* 使用MaterialPropertyBlock动态修改实例属性。
* [优化参看](https://github.com)

# 置换贴图生成裂缝、岩石凸起

* ‌**地形细节**‌：与Terrain系统配合，在运行时置换顶点生成裂缝或岩石凸起。
* 通过置换贴图动态修改地形顶点生成裂缝/岩石凸起的完整实现方案，结合Terrain系统与Shader Graph。

## **核心原理**

* ‌**置换贴图作用**‌：灰度图控制顶点位移（白色凸起，黑色凹陷），需配合曲面细分（Tessellation）增加几何精度
* ‌**URP适配**‌：通过Shader Graph的`Tessellation`节点和`Height`节点实现硬件细分与顶点偏移
* ‌**地形融合**‌：将置换Shader作为Terrain Layer材质，动态影响局部顶点

## **完整实现步骤**

### **置换贴图生成**

* ‌**工具选择**‌：使用Substance Designer或Photoshop绘制灰度图（岩石凸起区域为白色，裂缝为黑色）
* ‌**规范要求**‌：
  + 分辨率：2048x2048（匹配地形尺寸）
    格式：PNG无损压缩
    色彩空间：Linear

### **Shader Graph配置**

* [Height Map] → [Sample Texture 2D] → [Remap(0-1 to -1-1)]
* [Tessellation]节点设置细分级别（Edge:16, Inside:8）
* [Parallax Occlusion]节点增强深度感知
* 混合法线贴图与置换效果

### **Terrain系统集成**

```
|  |  |
| --- | --- |
|  | csharp |
|  | // C#脚本动态加载置换材质 |
|  | void ApplyDisplacementToTerrain() { |
|  | TerrainLayer layer = new TerrainLayer(); |
|  | layer.diffuseTexture = rockAlbedo; |
|  | layer.normalMapTexture = rockNormal; |
|  | layer.maskMapTexture = displacementMap;// 置换贴图 |
|  | Terrain.activeTerrain.terrainData.terrainLayers = new TerrainLayer[]{ layer }; |
|  | } |
```

### **性能优化**

* ‌**动态细分**‌：根据摄像机距离调整细分因子（`UnityDistanceBasedTess`）
* ‌**LOD控制**‌：超过50米后禁用置换效果
* ‌**批次合并**‌：启用SRP Batcher减少Draw Call

### **动态裂缝生成**

* ‌**运行时修改贴图**‌：

  ```
  |  |  |
  | --- | --- |
  |  | csharp |
  |  | // 通过RenderTexture实时绘制裂缝 |
  |  | void UpdateCrackTexture(Vector3 hitPoint) { |
  |  | Graphics.Blit(crackBrush, displacementRT, displacementMat); |
  |  | Shader.SetGlobalTexture("_DynamicDisplacement", displacementRT); |
  |  | } |
  ```
* ‌**Shader动态采样**‌：

  ```
  |  |  |
  | --- | --- |
  |  | hlsl |
  |  | float height = tex2Dlod(_DynamicDisplacement, float4(uv,0,0)).r; |
  |  | v.vertex.y += height * _DisplacementScale; |
  ```

### **技术对比**

| 方案 | 优点 | 缺点 | 适用场景 |
| --- | --- | --- | --- |
| 法线贴图 | 性能开销低 | 无真实几何变形 | 小范围表面细节 |
| 视差遮蔽映射 | 中等精度遮挡效果 | 高频细节失真 | 砖墙/地板接缝 |
| ‌**置换贴图**‌ | 真实几何变形 | 需硬件细分支持 | 地形/大规模结构变形 |

---

### **调试建议**

* ‌**可视化模式**‌：启用URP的`Depth Normals` Pass检查细分效果
* ‌**参数调优**‌：
  + \_TessellationEdgeLength：控制细分密度（建议值8-20）
    \_DisplacementScale：位移强度（建议值0.1-2.0）

---

> [【从UnityURP开始探索游戏渲染】](https://github.com)**专栏-直达**
> （欢迎*点赞留言*探讨，更多人加入进来能更加完善这个探索的过程，🙏）
