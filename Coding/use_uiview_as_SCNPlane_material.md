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

经过一番查找，得知 SceneKit 自己的渲染部分并不会使用主线程，而是使用自己的线程来做批量的处理。而由于我们的 UIView 现在已经成为了它其中一部分节点的纹理
，渲染时自然也是由它自己的线程进行界面绘制，其中可能涉及到重新布局的情况。找了一下，找不到可以从中修改渲染行为将 UIView 修改部分委托到主线程的方法。

还有个问题是如果显示的 UIView 内容比较复杂，在 AR 使用时，当摄像头初次扫过这个节点，整个 AR 视图会卡顿一下。可能由于绘制 UIView 卡住了渲染线程。

这让我们不禁想，能不能找到一个办法，即把 UIView 的内容显示出来，又不会存在上面说的两个问题呢？答案很简单，我们可以先把 UIView 渲染到一个 UIImage，然后把
这个 UIImage 作为 plane 的纹理。变成 UIImage 之后，在 scene 渲染的时候，就不会存在布局什么的变化，而且 SceneKit 对图像纹理的支持是天然友好的，不存在
绘制 UIView 的性能问题。

首先，创建一个简单的将 UIView 渲染到 UIImage 的方法：

```swift
extension UIView {
    func asImage() -> UIImage {
        let renderer = UIGraphicsImageRenderer(bounds: bounds)
        return renderer.image { rendererContext in
            layer.render(in: rendererContext.cgContext)
        }
    }
}
```

然后修改我们的代码如下：

```swift
func renderer(_ renderer: SCNSceneRenderer, didAdd node: SCNNode, for anchor: ARAnchor) {
  DispatchQueue.main.async { //操作 UIView 相关代码委托到主线程
      
      //设定要作为表面显示的 view
      let view = UIView(frame: CGRect(x: 0, y: 0, width: 400, height: 300)) //view 会被拉伸充满整个 plane，所以宽高的比例最好跟 plane 一致
      let image = view.asImage()
      
      let plane = SCNPlane(width: 0.4, height: 0.3)
      plane.firstMaterial?.diffuse.contents = image  //将 material 设置成 view，搞定
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

这样就可以顺利得到了我们预想中的效果。

由于渲染线程跟主线程并不是同一个，而且 SceneKit 并不要求要在主线程里面操作里面的 Node，我们可以把主线程的工作减少，把 SceneKit 的操作挪到其他线程去。

```swift
/// A serial queue for thread safety when modifying the SceneKit node graph.
    let updateQueue = DispatchQueue(label: Bundle.main.bundleIdentifier! +
        ".serialSceneKitQueue")

func renderer(_ renderer: SCNSceneRenderer, didAdd node: SCNNode, for anchor: ARAnchor) {
  DispatchQueue.main.async { //操作 UIView 相关代码委托到主线程
      
      //设定要作为表面显示的 view
      let view = UIView(frame: CGRect(x: 0, y: 0, width: 400, height: 300)) //view 会被拉伸充满整个 plane，所以宽高的比例最好跟 plane 一致
      let image = view.asImage()
      
      updateQueue.async { //把这部分逻辑转移到另一个队列，减轻主线程负担
        let plane = SCNPlane(width: 0.4, height: 0.3)
        plane.firstMaterial?.diffuse.contents = image  //将 material 设置成 view，搞定
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
}
```

搞定。
