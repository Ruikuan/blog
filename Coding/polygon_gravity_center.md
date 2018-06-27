# 多边形中心计算

有时候显示一个区域时，需要在多边形区域的中心显示一个标签或者说明，这个中心通常需要根据多边形的顶点来计算出来，而且需要比较符合**中心**的感觉。

最简单的方法是计算所有点的坐标的平均。

```js
function getCenter(points) {
    let len = points.length
    let lat = 0.0;
    let lng = 0.0;
    for (let l of points) {
        lat += (l.latitude / len)
        lng += (l.longitude / len)
    }
    return { latitude: lat, longitude: lng }
}
```

存在的问题是如果多个点密集在某一个地方（比如那个地方的线条很复杂，所以需要很多点才能表达出来），那么导致这个平均点就格外偏向那个地方，一点也没有 core 的感觉。

还有也是比较粗略的方法是找出所有点中的 `min(x)`,`min(y)`,`max(x)`,`max(y)`，然后计算这些边缘的中点，大体可以得到一个很粗略的中心点。对于形状不规则比较厉害的形状，也难尽人意。

更靠谱的方法是计算多边形的重心，重心在均匀的密度情况下就是几何中心（我们显示的区域当然是均匀的），通常都符合人们对**中心**的期望。计算如下：

```js
function getCenterOfGravityPoint(points) {
  let area = 0.0;//面积
  let lat = 0.0, lng = 0.0;// 重心的 x、y
  for (let i = 1; i <= points.length; i++) {
    let iLat = points[i % points.length].latitude;
    let iLng = points[i % points.length].longitude;
    let nextLat = points[i - 1].latitude;
    let nextLng = points[i - 1].longitude;
    let temp = (iLat * nextLng - iLng * nextLat) / 2.0;
    area += temp;
    lat += temp * (iLat + nextLat) / 3.0;
    lng += temp * (iLng + nextLng) / 3.0;
  }
  lat = lat / area;
  lng = lng / area;
  return { latitude: lat, longitude: lng };
}
```

这样得出的中心即使在很不规则的多边形情况下，看起来也自然多了。

> 参考  
> [几何中心 - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/%E5%87%A0%E4%BD%95%E4%B8%AD%E5%BF%83)  
> [几何］计算不规则多边形的面积、中心、重心](https://blog.csdn.net/shao941122/article/details/53671643)