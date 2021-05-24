## 需求介绍

实现吸管功能：

1. 当点击吸管图标，出现遮罩层，突出需要吸色的画布

2. 鼠标移动至画布，显示放大镜

3. 放大镜中有网格线与中心点小方格。取色取的是中心点小方格的色值

4. 网格线里的每个小方格都是像素点

5. 放大镜需要一直出现在画布中，即鼠标移到画布的各个角落，放大镜都在

<img src="https://img.youpin.mi-img.com/luban/p3k5mug508o_22026080841621842941665.gif" style="zoom:50%;" />

## 处理图像

首先需要将图片渲染至 canvas 中，后续我们统称 canvas 为画布

```js
const colorImg: any = document.getElementById("colorImg");
const canvas = draw(colorImg);

const draw = (img: any) => {
  const style = window.getComputedStyle(img);
  const width = parseInt(style.width, 10);
  const height = parseInt(style.height, 10);
  const canvas = document.createElement("canvas");
  canvas.width = width;
  canvas.height = height;
  canvas.setAttribute("id", "canvasImg");
  canvas.style.zIndex = "10003";
  canvas.style.position = "absolute";
  canvas.style.cursor = "pointer";
  const context: any = canvas.getContext("2d");
  context.drawImage(img, 0, 0);
  data.parentNode.appendChild(canvas);

  // 获取放大镜
  getMagnifier(
    context,
    canvas,
    img,
    data.magnifierSize,
    data.ratio,
    data.magnifierOffset,
    data.step
  );

  // 获取坐标颜色
  function getPixelColor(x: number, y: number) {
    const imageData = context.getImageData(
      x + data.step / 2,
      y + data.step / 2,
      data.step - data.step / 2,
      data.step - data.step / 2
    );
    const pixel = imageData.data;
    const r = pixel[0];
    const g = pixel[1];
    const b = pixel[2];
    let a = pixel[3] / 255;
    a = Math.round(a * 100) / 100;
    let rHex = r.toString(16);
    r < 16 && (rHex = "0" + rHex);
    let gHex = g.toString(16);
    g < 16 && (gHex = "0" + gHex);
    let bHex = b.toString(16);
    b < 16 && (bHex = "0" + bHex);
    const rgbaColor = "rgba(" + r + "," + g + "," + b + "," + a + ")";
    const rgbColor = "rgb(" + r + "," + g + "," + b + ")";
    const hexColor = "#" + rHex + gHex + bHex;
    return {
      rgba: rgbaColor,
      rgb: rgbColor,
      hex: hexColor,
      r: r,
      g: g,
      b: b,
      a: a,
    };
  }

  return {
    getColor: getPixelColor,
  };
};
```

## 实现放大镜

放大镜需要跟随着鼠标移动，并且始终保持在画布中。这里注意：节流。控制一定时间再渲染放大镜，不然会很卡

```js
/*
 *  context:绘制环境对象,
 *  element:canvas元素对象
 *  img：图片对象
 *  diameter:放大镜的大小，
 *  ratio：图形的放大比例，
 * （比例 = 原图 ：镜中图像）0<ratio<1缩小图像，ratio>1放大图像
 * magnifierOffset: 放大镜偏移量
 * step: 网格间隙
 * */
let timer: any;
const getMagnifier = (
  context: any,
  element: any,
  img: any,
  diameter: any,
  ratio: any,
  magnifierOffset: number,
  step: number
) => {
  //绘制图片
  context.drawImage(img, 0, 0, element.width, element.height);
  //鼠标在element中移动触发事件
  element.onmousemove = function (e: any) {
    timer && clearTimeout(timer);
    setTimeout(() => {
      context.clearRect(0, 0, element.width, element.height);
      //绘制图片
      context.drawImage(img, 0, 0, element.width, element.height);
      //解决浏览器兼容问题
      e = e ? e : window.event;
      //获取鼠标在element元素中的坐标值
      const cxy = windowToCanvas(element, e.clientX, e.clientY);
      context.save(); //保存当前绘制环境

      const offsetX = cxy.x,
        offsetY = cxy.y;

      // 放大镜绘制图像的坐标
      let xCoordinate = 0,
        yCoordinate = 0;
      if (
        offsetX > img.width - diameter / 2 - magnifierOffset &&
        offsetY > img.height - diameter / 2 - magnifierOffset
      ) {
        // 右下角
        xCoordinate = offsetX - magnifierOffset;
        yCoordinate = offsetY - magnifierOffset;
      } else if (offsetX > img.width - diameter / 2 - magnifierOffset) {
        // 右上角
        xCoordinate = offsetX - magnifierOffset;
        yCoordinate = offsetY + magnifierOffset;
      } else if (offsetY > img.height - diameter / 2 - magnifierOffset) {
        //左下角
        xCoordinate = offsetX + magnifierOffset;
        yCoordinate = offsetY - magnifierOffset;
      } else {
        // 左上角及中间部分
        xCoordinate = offsetX + magnifierOffset;
        yCoordinate = offsetY + magnifierOffset;
      }
      getClip(context, xCoordinate, yCoordinate, diameter / 2);
      //将内容放入到放大镜中显示
      //根据鼠标点的坐标值计算出在原图的坐标值
      const ytx0 = (img.width / element.width) * offsetX; //计算出鼠标在原图的X坐标值
      const yty0 = (img.height / element.height) * offsetY; //计算出鼠标在原图的Y坐标值
      //（原图形/显示图形比例）* （放大镜直径/比例= 镜中的图形所占大小）= 原图要截取的图像大小
      const ytclipValueW = ((img.width / element.width) * diameter) / ratio; //在原图截取图片的宽度
      const ytclipValueH = ((img.height / element.height) * diameter) / ratio; //在原图截取图片的宽度
      //.drawImage(图像对象,原图像截取的起始X坐标,原图像截取的起始Y坐标,原图像截取的宽度,原图像截取的高度，
      // 绘制图像的起始X坐标,绘制图像的起始Y坐标,绘制图像所需要的宽度,绘制图像所需要的高度);
      console.log(ytx0, yty0, ytclipValueW, ytclipValueH);
      context.drawImage(
        img, // 图像对象
        ytx0 - ytclipValueW / 2, // 原图像截取的起始X坐标
        yty0 - ytclipValueH / 2, //原图像截取的起始Y坐标
        ytclipValueW, // 原图像截取的宽度
        ytclipValueH, //原图像截取的高度
        xCoordinate - diameter / 2, //绘制图像的起始X坐标
        yCoordinate - diameter / 2, //绘制图像的起始Y坐标
        diameter, //绘制图像所需要的宽度
        diameter //绘制图像所需要的高度
      );
      // 获取放大镜网格线
      drawGrid(
        context,
        step,
        xCoordinate - diameter / 2,
        yCoordinate - diameter / 2,
        diameter
      );
      context.restore(); //恢复当前保存的绘制环境
    }, 10);
  };
};

/*
 * 坐标转换：将window中的坐标转换到元素盒子中的坐标，并返回(x,y)坐标
 * element：canvas元素对象
 * x:鼠标在当前窗口X坐标值
 * y:鼠标在当前窗口Y坐标值
 * */
const windowToCanvas = (element: any, x: any, y: any) => {
  //获取当前鼠标在window中的坐标值
  // alert(event.clientX+"-------"+event.clientY);
  //获取元素的坐标属性
  const box = element.getBoundingClientRect();
  const bx = x - box.left;
  const by = y - box.top;
  return { x: bx, y: by };
};

/*
 * 获取放大镜框：进行图层切割
 * context：绘制环境对象
 * x:鼠标在画布中的X坐标
 * y:鼠标在画布中的Y坐标
 * r:放大镜的直径
 * */
const getClip = (context: any, x: number, y: number, r: number) => {
  context.beginPath();
  context.strokeStyle = "#ffffff"; // 圆形线颜色
  context.lineWidth = 0;
  context.arc(x, y, r, 0, Math.PI * 2, true);
  context.stroke();
  context.clip(); //沿形状切除向外的图层
};
```

## 像素点处理

经过以上处理，放大镜能够放大鼠标移动的区域。现在我们需要将放大的图像进行像素点处理。用到的 api 为：**getImageData**

```js
/*
 * 网格图像像素点
 * step: 网格大小
 * x: 鼠标在画布中的X坐标
 * y: 鼠标在画布中的Y坐标
 * diameter: 放大镜的大小
 * */
function getGidPixel(
  context: any,
  step: number,
  x: number,
  y: number,
  diameter: number
) {
  for (let i = 0; i < diameter; i += step) {
    for (let j = 0; j < diameter; j += step) {
      let pixel = context.getImageData(
        x + i + step / 2,
        y + j + step / 2,
        step - step / 2,
        step - step / 2
      );
      let data = pixel.data;
      let rgba =
        "rgba(" +
        data[0] +
        "," +
        data[1] +
        "," +
        data[2] +
        "," +
        (data[3] / 255).toFixed(2) +
        ")";
      context.fillStyle = rgba;
      context.fillRect(x + i, y + j, step, step);
    }
  }
}
```

## 实现网格线

最后往上面覆盖一层网格线

```js
/*
 * 画放大镜的网格线
 * context: 绘制环境对象,
 * step: 网格间隔
 * x: 鼠标在画布中的X坐标
 * y: 鼠标在画布中的Y坐标
 * diameter: :放大镜的大小
 * */
const drawGrid = (
  context: any,
  step: number,
  x: number,
  y: number,
  diameter: number
) => {
  // 先保存状态 已便于画完圆再用
  context.save();

  context.lineWidth = 0.5; // 线的宽度

  getGidPixel(context, step, x + 0.5, y + 0.5, diameter);

  // 创建垂直格网线路径
  //注意：canvas在两个像素的边界处画线
  //由于定位机制，1px的线会变成2px
  //于是要+0.5
  context.strokeStyle = "#ccc"; // 网格线颜色
  for (let i = 0.5 + step; i < diameter; i += step) {
    context.beginPath();
    context.moveTo(i + x, 0 + y);
    context.lineTo(i + x, diameter + y);
    context.closePath();
    context.stroke();
  }
  // 创建水平格网线路径
  for (let j = 0.5 + step; j < diameter; j += step) {
    context.beginPath();
    context.moveTo(0 + x, j + y);
    context.lineTo(diameter + x, j + y);
    context.closePath();
    context.stroke();
  }
  // 实现中心小方格
  // ...

  //恢复之前保存的绘图上下文，可以继续绘制
  context.restore();
};
```

## 实现中心点小方格

中心点小方格很有必要，它是取色点。这里需要记录其 x、y 坐标

```js
context.beginPath();
context.lineWidth = 0.5;
context.strokeStyle = "red";
data.middleX = diameter / 2 + x - step / 2;
data.middleY = diameter / 2 + y - step / 2;
context.rect(data.middleX, data.middleY, step + 0.5, step + 0.5);
context.closePath();
context.stroke();
```

## 吸色

通过监听画布的 click 事件，获取中心小方格 x、y 坐标的色值

```js
const canvasImg: any = document.getElementById("canvasImg");

// 关闭图片
canvasImg.addEventListener("click", function (e: any) {
  const x = data.middleX;
  const y = data.middleY;
  const rgba = canvas.getColor(x, y).rgba;
  data.pickColor = rgba;
  data.parentNode.removeChild(canvasImg);
  ElMessage.success(`已成功取色：${rgba}`);
  data.isShowMask = false;
  e.stopPropagation();
});
```

## 不足

因为在绘制放大镜之前，我们使用了 `clip() ` 来剪切图形为圆形。当处理像素点时，也把圆形的边框处理了。经过调整，四个角还是会出现几个像素点的透明，不知如何解决，但不太影响使用，遂没有继续研究。如果友友们有更好的方法，希望可以一起交流~

![https://img.youpin.mi-img.com/luban/v6aa5k3mcoo_22026080841621842545997.png](https://img.youpin.mi-img.com/luban/v6aa5k3mcoo_22026080841621842545997.png)

## 参考文章

[基于 canvas 开发吸色功能](https://juejin.cn/post/6844904118926983182#heading-0)
