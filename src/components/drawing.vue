<template>
  <div class="container">
    <canvas id="drawingBorad"></canvas>
    <div class="tool-container">
      <div class="icon-div icon" @click="isShowDrawPane = !isShowDrawPane">
        <icon name="draw" scale="4"></icon>
      </div>
      <div class="icon-div icon" @click="filterObject('erase')">
        <icon name="erase" scale="4"></icon>
      </div>
      <div class="icon-div icon" @click="filterObject('line')">
        <icon name="line" scale="4"></icon>
      </div>
      <div class="icon-div icon" @click="filterObject('arrows')">
        <icon name="arrows" scale="4"></icon>
      </div>
      <div class="icon-div icon" @click="filterObject('rect')">
        <icon name="rect" scale="4"></icon>
      </div>
      <div class="icon-div icon" @click="filterObject('circle')">
        <icon name="circle" scale="4"></icon>
      </div>
      <div class="icon-div icon" @click="filterObject('text')">
        <icon name="text" scale="4"></icon>
      </div>
      <div class="icon-div icon" @click="clearCanvas()">
        <icon name="clear" scale="4"></icon>
      </div>
      <div class="icon-div icon" @click="redo()">
        <icon :name="historyImageData.length > 0 ? 'redo' : 'grey-redo' " scale="4"></icon>
      </div>
      <div class="icon-div icon" @click="cancelRedo()">
        <icon :name="newHistoryImageData.length > 0 ? 'cancelRedo' : 'grey-cancelRedo' " scale="4"></icon>
      </div>
      <div class="icon-div icon" @click="downLoad()">
        <icon name="download" scale="4"></icon>
      </div>
      <div class="drawPane" v-show="isShowDrawPane">
        <div @click="isShowDrawPane = false">
          <icon class="close-draw-pane icon" name="close" scale="3"></icon>
        </div>
        <h5>画笔大小</h5>
        <input type="range" id="lwRange" min="1" max="10" value="1" />
        <h5>画笔颜色</h5>
        <input type="color" id="lcolor" value="#ffa500" />
      </div>
    </div>
    <textarea
      id="textarea"
      name="textBox"
      cols="9"
      rows="1"
      class="text-style"
      v-show="isShowText"
    ></textarea>
  </div>
</template>

<script>

export default {
  name: "Drawing",
  props: {
    msg: String,
  },
  data() {
    return {
      isShowDrawPane: false,
      canvas: null,
      context: null,
      //线宽
      lwidth: 2,
      //画笔颜色
      lcolor: "orange",
      //维护绘画状态的数组
      paintTypeArr: {
        painting: false,
        erase: false,
        line: false,
        arrows: false,
        rect: false,
        circle: false,
        text: false,
      },
      //最近一次的canvas图片的数据
      imageData: null,
      //是否显示文字编写框
      isShowText: false,
      //保存画布图片历史的数据
      historyImageData:[],
      //保存已被撤销的历史画布图片数据
      newHistoryImageData:[],
      socket:null
    };
  },
  mounted() {
    let self = this;
    self.init()  
    window.onresize = function () {
      self.init()  
    }  
    document.getElementById("lwRange").onchange = function () {
      self.lwidth = parseInt(document.getElementById("lwRange").value)  
    }  
    document.getElementById("lcolor").onchange = function () {
      self.context.fillStyle = document.getElementById("lcolor").value  
      self.context.strokeStyle = document.getElementById("lcolor").value  
    }  
    this.listen() 
    this.initWebSocket() 
  },
  methods: {
    //初始化画布
    init() {
      this.canvas = document.getElementById("drawingBorad")  
      this.context = this.canvas.getContext("2d")  
      this.canvas.width = window.innerWidth  
      this.canvas.height = window.innerHeight  
      this.imageData && this.context.putImageData(this.imageData, 0, 0)  
    },
    //监听鼠标,用于画笔任意绘制和橡皮擦
    listen() {
      let self = this  
      let lastPoint = { x: undefined, y: undefined }  
      let rect = self.canvas.getBoundingClientRect()  
      var scaleX = self.canvas.width / rect.width  
      var scaleY = self.canvas.height / rect.height  
      let textPoint = { x: undefined, y: undefined }  

      self.canvas.onmousedown = function (e) {
        self.paintTypeArr["painting"] = true  

        let x1 = e.clientX  
        let y1 = e.clientY  
        x1 -= rect.left  
        y1 -= rect.top  
        lastPoint = { x: x1 * scaleX, y: y1 * scaleY }  

        if (self.paintTypeArr["text"]) {
          let textarea = document.getElementById("textarea")  
          if (self.isShowText) {
            let textContent = textarea.value  
            self.isShowText = false  
            textarea.value = ""  
            self.drawText(textPoint.x, textPoint.y, textContent)  
          } else if (!self.isShowText) {
            self.isShowText = true  

            textarea.style.left = lastPoint.x + "px"  
            textarea.style.top = lastPoint.y + "px"  
            textPoint = { x: lastPoint.x, y: lastPoint.y }  
            // textarea.style['z-index'] = 6
          }
        }

        if (self.paintTypeArr["erase"]) {
          let ctx = self.context  
          ctx.save()  
          ctx.globalCompositeOperation = "destination-out"  
          ctx.beginPath()  
          let radius = self.lWidth / 2 > 5 ? self.lWidth / 2 : 5  
          ctx.arc(lastPoint.x, lastPoint.y, radius, 0, 2 * Math.PI)  
          ctx.clip()  
          ctx.clearRect(0, 0, self.canvas.width, self.canvas.height)  
          ctx.restore()  
        }

        var thee = e ? e : window.event  
        self.stopBubble(thee)  
      }  
      self.canvas.onmousemove = function (e) {
        let x2 = e.clientX  
        let y2 = e.clientY  
        x2 -= rect.left  
        y2 -= rect.top  
        let newPoint = { x: x2 * scaleX, y: y2 * scaleY }  

        if (self.isPainting()) {
          self.drawLine(lastPoint.x, lastPoint.y, newPoint.x, newPoint.y)  
          lastPoint = newPoint  
        } else if (self.paintTypeArr["erase"]) {
          if(!lastPoint.x && !lastPoint.y){return}
          self.handleErase(lastPoint.x, lastPoint.y, newPoint.x, newPoint.y)  
          lastPoint = newPoint  
        } else if (self.paintTypeArr["line"]) {
          self.clearCanvas()  
          self.imageData && self.context.putImageData(self.imageData, 0, 0)  
          self.drawLine(lastPoint.x, lastPoint.y, newPoint.x, newPoint.y)  
        } else if (self.paintTypeArr["arrows"]) {
          self.clearCanvas()  
          self.imageData && self.context.putImageData(self.imageData, 0, 0)  
          self.drawArrow(lastPoint.x, lastPoint.y, newPoint.x, newPoint.y)  
        } else if (self.paintTypeArr["rect"]) {
          self.clearCanvas()  
          self.imageData && self.context.putImageData(self.imageData, 0, 0)  
          self.drawRect(lastPoint.x, lastPoint.y, newPoint.x, newPoint.y)  
        } else if (self.paintTypeArr["circle"]) {
          self.clearCanvas()  
          self.imageData && self.context.putImageData(self.imageData, 0, 0)  
          self.drawCircle(lastPoint.x, lastPoint.y, newPoint.x, newPoint.y)  
        }

        var thee = e ? e : window.event  
        self.stopBubble(thee)  
      }  
      self.canvas.onmouseup = function () {
        lastPoint = { x: undefined, y: undefined }  
        self.canvasDraw()  
        console.log(123)
        self.filterObject()  
      }  
    },
    //更新绘画类型数组paintTypeArr的状态
    filterObject(type) {
      if (!type) {
        for (const key in this.paintTypeArr) {
          this.paintTypeArr[key] = false  
        }
      } else {
        for (const key in this.paintTypeArr) {
          key === type
            ? (this.paintTypeArr[key] = true)
            : (this.paintTypeArr[key] = false)  
        }
      }
    },
    //阻止事件冒泡
    stopBubble(evt) {
      if (evt.stopPropagation) {
        evt.stopPropagation()  
      } else {
        //ie
        evt.cancelBubble = true  
      }
    },
    //判断是否是自由绘画模式
    isPainting() {
      for (let key in this.paintTypeArr) {
        if (key !== "painting" && this.paintTypeArr[key]) {
          return false  
        }
      }
      if (this.paintTypeArr["painting"]) {
        return true  
      }
      return false  
    },
    //橡皮擦
    handleErase(x1, y1, x2, y2) {
      let ctx = this.context  
      //获取两个点之间的剪辑区域四个端点
      var asin = radius * Math.sin(Math.atan((y2 - y1) / (x2 - x1)))  
      var acos = radius * Math.cos(Math.atan((y2 - y1) / (x2 - x1)))  
      var x3 = x1 + asin  
      var y3 = y1 - acos  
      var x4 = x1 - asin  
      var y4 = y1 + acos  
      var x5 = x2 + asin  
      var y5 = y2 - acos  
      var x6 = x2 - asin  
      var y6 = y2 + acos   //保证线条的连贯，所以在矩形一端画圆

      ctx.save()  
      ctx.beginPath()  
      ctx.globalCompositeOperation = "destination-out"  
      let radius = this.lWidth / 2 > 5 ? this.lWidth / 2 : 5  
      ctx.arc(x2, y2, radius, 0, 2 * Math.PI)  
      ctx.clip()  
      ctx.clearRect(0, 0, this.canvas.width, this.canvas.height)  
      ctx.restore()   //清除矩形剪辑区域里的像素

      ctx.save()  
      ctx.beginPath()  
      ctx.globalCompositeOperation = "destination-out"  
      ctx.moveTo(x3, y3)  
      ctx.lineTo(x5, y5)  
      ctx.lineTo(x6, y6)  
      ctx.lineTo(x4, y4)  
      ctx.closePath()  
      ctx.clip()  
      ctx.clearRect(0, 0, this.canvas.width, this.canvas.height)  
      ctx.restore()  
    },
    //画线
    drawLine(fromX, fromY, toX, toY) {
      let ctx = this.context  
      ctx.beginPath()  
      ctx.lineWidth = this.lwidth  
      ctx.lineCap = "round"  
      ctx.lineJoin = "round"  
      ctx.moveTo(fromX, fromY)  
      ctx.lineTo(toX, toY)  
      ctx.stroke()  
      ctx.closePath()  
    },
    //画箭头
    drawArrow(fromX, fromY, toX, toY) {
      let ctx = this.context  
      var headlen = 10   //自定义箭头线的长度
      var theta = 45   //自定义箭头线与直线的夹角，个人觉得45°刚刚好
      var arrowX, arrowY   //箭头线终点坐标
      // 计算各角度和对应的箭头终点坐标
      var angle = (Math.atan2(fromY - toY, fromX - toX) * 180) / Math.PI  
      var angle1 = ((angle + theta) * Math.PI) / 180  
      var angle2 = ((angle - theta) * Math.PI) / 180  
      var topX = headlen * Math.cos(angle1)  
      var topY = headlen * Math.sin(angle1)  
      var botX = headlen * Math.cos(angle2)  
      var botY = headlen * Math.sin(angle2)  
      ctx.beginPath()  
      //画直线
      ctx.moveTo(fromX, fromY)  
      ctx.lineTo(toX, toY)  

      arrowX = toX + topX  
      arrowY = toY + topY  
      //画上边箭头线
      ctx.moveTo(arrowX, arrowY)  
      ctx.lineTo(toX, toY)  

      arrowX = toX + botX  
      arrowY = toY + botY  
      //画下边箭头线
      ctx.lineTo(arrowX, arrowY)  

      ctx.stroke()  
      ctx.closePath()  
    },
    //绘制矩形
    drawRect(topLeftX, topLeftY, botRightX, botRightY) {
      let ctx = this.context  
      ctx.strokeRect(
        topLeftX,
        topLeftY,
        Math.abs(botRightX - topLeftX),
        Math.abs(botRightY - topLeftY)
      )  
    },
    //画圆
    drawCircle(circleX, circleY, endX, endY) {
      let ctx = this.context  
      let radius = Math.sqrt(
        (circleX - endX) * (circleX - endX) +
          (circleY - endY) * (circleY - endY)
      )  
      ctx.beginPath()  
      ctx.arc(circleX, circleY, radius, 0, Math.PI * 2, true)  
      ctx.stroke()  
    },
    //画文字
    drawText(startX, startY, content) {
      let ctx = this.context  
      ctx.save()  
      ctx.beginPath()  
      ctx.font = "25px orbitron"  
      ctx.textBaseline = "top"  
      ctx.fillText(content, parseInt(startX), parseInt(startY))  
      ctx.restore()  
      ctx.closePath()  
    },
    //清屏
    clearCanvas() {
      this.context.clearRect(0, 0, this.canvas.width, this.canvas.height)  
    },
    //定格画布图片
    canvasDraw() {
      this.imageData = this.context.getImageData(0,0,this.canvas.width,this.canvas.height)  
      this.historyImageData.push(this.imageData)
      console.log(this.historyImageData)
    },
    //撤销
    redo(){
      let historyImageData = this.historyImageData
      let newHistoryImageData = this.newHistoryImageData
      if(historyImageData.length > 0){
        let hisImg = historyImageData.pop()
        newHistoryImageData.push(hisImg)
        if(historyImageData.length === 0){
          this.imageData = null
          this.clearCanvas()
        }else{
          this.context.putImageData(historyImageData[historyImageData.length - 1],0,0)
        }
      }
    },
    //反撤销
    cancelRedo(){
      if(this.newHistoryImageData.length > 0){
        const newHisImg = this.newHistoryImageData.pop()
        this.imageData = newHisImg
        this.context.putImageData(newHisImg,0,0)
        this.historyImageData.push(newHisImg)
      }
    },
    //保存图片
    downLoad(){
      const imgUrl = this.canvas.toDataURL('image/png')
      const a = document.createElement('a')
      a.href = imgUrl
      a.download = '绘图保存记录' + (new Date).getTime()
      a.target = '_blank'
      document.body.appendChild(a)
      a.click()
    }
  },
}  
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.container {
  width: 100%;
  height: 100%;
  margin: 10px auto;
  overflow: hidden;
}
.tool-container {
  position: fixed;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  border: 2px solid orange;
  border-radius: 10px;
  display: flex;
  justify-content: center;
}
.drawPane {
  padding: 25px 20px;
  height: 120px;
  position: absolute;
  top: -120px;
  left: 0px;
  border-radius: 5px;
  border: 2px solid orangered;
}
.close-draw-pane {
  position: absolute;
  right: 5px;
  top: 5px;
}
.icon-div {
  margin: 10px 12px;
}
.icon :hover {
  cursor: pointer;
}
input[type="range"] {
  -webkit-appearance: none;
  width: 180px;
  height: 24px;
  outline: none;
  margin-bottom: 3px;
}
input[type="range"]::-webkit-slider-runnable-track {
  background-color: orangered;
  height: 4px;
  border-radius: 5px;
}
input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: orange;
  cursor: pointer;
  margin-top: -4px;
}
.text-style {
  float: left;
  position: absolute;
  font: 25px orbitron;
  word-break: break-all;
  background-color: transparent;
}
</style>
