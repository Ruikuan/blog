# 如何使用 UIView 作为 SCNPlane 的 material 显示

在开发 ARKit 或者 SceneKit 应用时，有时候会希望 3D 空间的物体表面呈现出自己自定义的 UIView。有时候可能是一些文字显示，有时候是一些图表。

下面用 ARKit 中的操作做个例子，将 UIView 的内容显示在一个 SCNPlane 上。

我们在 `renderer(_ renderer:didAdd:for:)` 中做这个操作。在这个回调中，已经有一个 `SCNNode` 被创建了，而且被放置到了对应 `anchor` 的位置，
现在我们把它当作我们后面创建的 node 的父节点。

```swift
func renderer(_ renderer: SCNSceneRenderer, didAdd node: SCNNode, for anchor: ARAnchor) {
  DispatchQueue.main.async { //操作 UIView 相关代码委托到主线程
      
      //设定要作为表面显示的 view
      let view = UIView(frame: CGRect(x: 0, y: 0, width: 400, height: 300)) //view 会被拉伸充满整个 plane，所以宽高的比例最好跟 plane 一致
      view.backgroundColor = .red //设置个颜色，让它看得见
      
      let plane = SCNPlane(width: 0.4, height: 0.3)
      plane.firstMaterial?.diffuse.contents = view  //将 material 设置成 view，搞定
      plane.firstMaterial?.isDoubleSided = true   //正反面都显示同样的 material
      
      let planeNode = SCNNode(geometry: plane)
     
      /*
       `SCNPlane` 在它自己的坐标系是竖直的,但 `ARImageAnchor` 认为它在自己的空间是水平的, 所以要将它翻起来。
       */
      planeNode.eulerAngles.x = -.pi / 2

      // 将 plane 加入到 scene 中
      node.addChildNode(planeNode)
  }
}
```

就这么简单，把自定义的 UIView 显示上去了。

不过在实际开发中，我发现存在问题。调试时，老是提示存在主线程外修改 UI 自动布局引擎的情况。仔细检查自己代码，确实自己所有操作 UIKit 相关的代码全都
委托给了主线程，并不存在无意中用后台线程修改了 UIView 的情况。

经过一番查找，得知 SceneKit 自己的渲染部分并不会使用主线程，而是使用自己的线程来做批量的处理。而由于我们的 UIView 现在已经成为了它其中一部分节点的纹理，
渲染
