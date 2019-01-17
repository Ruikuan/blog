# iOS 下通过 Vision API 识别到的物体边框坐标转化到屏幕显示的示例代码

使用 Vision API 识别出来的物体的边框为对应媒体的坐标系，对于图片来说，原点在左下角，而且坐标被标准化为 [0, 1] 区间的量。
这跟 UIKit 中原点在左上角的情况有所不同，因为需要把边框显示出来的话，需要把识别出来的边框的坐标做转化。

不废话上代码。

```swift
func adjustBoundsToScreen(_ box: CGRect, containerSize: CGSize, orientation: UIDeviceOrientation) -> CGRect {
        let H = containerSize.height
        let W = containerSize.width
        
        switch orientation {
        case .landscapeRight:
            let width = box.size.width * W
            let height = box.size.height * H
            let xCord = W - box.origin.x * W - width
            let yCord = box.origin.y * H
            return CGRect(x: xCord, y: yCord, width: width, height: height)
        case .portraitUpsideDown:
            let width = box.size.height * W
            let height = box.size.width * H
            let xCord = W - box.origin.y * W - width
            let yCord = H - box.origin.x * H - height
            return CGRect(x: xCord, y: yCord, width: width, height: height)
        case .landscapeLeft:
            let width = box.size.width * W
            let height = box.size.height * H
            let xCord = box.origin.x * W
            let yCord = H - box.origin.y * H - height
            return CGRect(x: xCord, y: yCord, width: width, height: height)
        case .portrait:
            fallthrough
        default:
            let width = box.size.height * W
            let height = box.size.width * H
            let xCord = box.origin.y * W
            let yCord = box.origin.x * H
            return CGRect(x: xCord, y: yCord, width: width, height: height)
        }
    }
```
