<template>
  <div id="picker">
    <el-button
      type="primary"
      icon="el-icon-edit"
      circle
      @click.stop="getBodymovinJson"
    ></el-button>
    <el-color-picker v-model="pickColor"></el-color-picker>
  </div>
  <div id="main">
    <img
      id="colorImg"
      src="https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fgss0.baidu.com%2F9vo3dSag_xI4khGko9WTAnF6hhy%2Fzhidao%2Fpic%2Fitem%2F902397dda144ad34a88dcba9d3a20cf431ad85ea.jpg&refer=http%3A%2F%2Fgss0.baidu.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1624419963&t=9ed71981c48b1cbdafd74a7afdf82d4a"
      alt="夏目友人帐"
      crossorigin="Anonymous"
    />
    <div id="colorDiv" v-show="isShowMask"></div>
  </div>
</template>

<script lang="ts">
import { defineComponent, reactive, nextTick, toRefs } from "vue";
import { ElMessage } from "element-plus";

export default defineComponent({
  emits: ["strawColor"],
  setup(props, ctx) {
    const data = reactive({
      magnifierSize: 120, // 放大镜大小
      ratio: 8, // 放大镜放大比例
      magnifierOffset: 80, // 放大镜偏移量
      step: 11, // 网格间隙
      middleX: 0, // 中间小方格x
      middleY: 0, // 中间小方格y
      parentNode: null,
      isShowMask: false, // 是否展示遮罩
      pickColor: null,
    });

    nextTick(() => {
      data.parentNode = document.getElementById("main");
    });

    const getBodymovinJson = async () => {
      initCanvas();
      data.isShowMask = true;
    };

    const initCanvas = async () => {
      const colorImg: any = document.getElementById("colorImg");
      const canvas = draw(colorImg);
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
      // 禁止滚动
      canvasImg.addEventListener("mousewheel", function (e: any) {
        e.preventDefault();
        e.stopPropagation();
      });
    };

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
          const ytclipValueH =
            ((img.height / element.height) * diameter) / ratio; //在原图截取图片的宽度
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
      // 中心小方格
      context.beginPath();
      context.lineWidth = 0.5;
      context.strokeStyle = "red";
      data.middleX = diameter / 2 + x - step / 2;
      data.middleY = diameter / 2 + y - step / 2;
      context.rect(data.middleX, data.middleY, step + 0.5, step + 0.5);
      context.closePath();
      context.stroke();

      //恢复之前保存的绘图上下文，可以继续绘制
      context.restore();
    };

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

    return {
      ...toRefs(data),
      initCanvas,
      getBodymovinJson,
    };
  },
});
</script>

<style scoped>
#picker {
  display: flex;
  align-items: center;
  justify-content: center;
}
#main {
  width: 40%;
}
#colorImg {
  z-index: 1003;
  position: absolute;
  cursor: pointer;
  width: 960px;
  height: 800px;
}
#colorDiv {
  position: fixed;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  background: rgba(0, 0, 0, 0.5);
}
</style>
