<template>
    <div class="plotter">
        <canvas id="canvas" width="1000" height="800">
            your browser doesn't support canvas!
        </canvas>
    </div>
</template>

<script>
  export default {
    name: 'Plotter',
    props: {
      plotterData: Object
    },
    mounted() {
      this.ctx.rect(0, 0, this.canvas.width, this.canvas.height);
      this.ctx.stroke();
      const padding = 40;
      const {xAxis, yAxis, factor} = this.getFactorAndAxis(
        this.plotterData.maxCoords.x,
        this.plotterData.minCoords.x,
        this.plotterData.maxCoords.y,
        this.plotterData.minCoords.y,
        padding
      )
      this.drawAxis(xAxis, yAxis, factor, padding)
      this.drawFillCircle(this.plotterData.receiverA.x, this.plotterData.receiverA.y, 5, factor)
      this.drawFillCircle(this.plotterData.receiverB.x, this.plotterData.receiverB.y, 5, factor)
      this.drawFillCircle(this.plotterData.receiverC.x, this.plotterData.receiverC.y, 5, factor)
      this.ctx.beginPath();
      this.plotterData.transmitterRoute.forEach(point => {
        this.drawLineWithPointTo(point.x, point.y, 2, factor)
      })
    },
    computed: {
      canvas() {
        return document.querySelector('#canvas');
      },
      ctx() {
        return this.canvas.getContext('2d');
      }
    },
    methods: {
      //получение коэффициенты смещения и расположения осей
      getFactorAndAxis(maxX, minX, maxY, minY, padding) {
        let xFactor, yFactor = 1;
        let xAxis, yAxis;
        if (maxX >= 0 && minX >= 0) {
          xAxis = this.canvas.height - padding
          xFactor = (this.canvas.height - padding * 2) / maxX
        } else if (maxX <= 0 && minX <= 0) {
          xAxis = padding
          xFactor = (this.canvas.height - padding * 2) / minX
        } else {
          xAxis = (this.canvas.height - 2 * padding) / (Math.abs(minY / maxY) + 1) + padding
          xFactor = (this.canvas.height - padding * 2) / (maxY - minY)
        }
        if (maxY >= 0 && minY >= 0) {
          yAxis = padding
          yFactor = (this.canvas.width - padding * 2) / maxY
        } else if (maxY <= 0 && minY <= 0) {
          yAxis = this.canvas.width - padding
          yFactor = (this.canvas.width - padding * 2) / minY
        } else {
          yAxis = (this.canvas.width - 2 * padding) / (Math.abs(maxX / minX) + 1) + padding
          yFactor = (this.canvas.width - padding * 2) / (maxX - minX)
        }
        const factor = Math.min(xFactor, yFactor)
        return ({xAxis, yAxis, factor})
      },
      //отрисовка осей
      drawAxis(xAxis, yAxis, factor, padding) {
        this.ctx.beginPath();
        this.ctx.moveTo(0, xAxis);
        this.ctx.lineTo(this.canvas.width, xAxis)
        this.ctx.fillText("x (1000 км)", this.canvas.width - 55, xAxis - 5);
        this.ctx.moveTo(yAxis, 0);
        this.ctx.lineTo(yAxis, this.canvas.height)
        this.ctx.fillText("y (1000 км)", yAxis + 5, 15);
        this.ctx.stroke();
        this.ctx.translate(yAxis, xAxis);
        const xAxisStep = this.canvas.width / 20;
        const yAxisStep = this.canvas.height / 20;
        this.ctx.fillText("0", 5, 15);
        for (let i = xAxisStep; i < this.canvas.width - padding - yAxis || i < yAxis - padding; i += xAxisStep) {
          this.ctx.moveTo(i, -3)
          this.ctx.lineTo(i, 3)
          this.ctx.fillText((i / factor).toFixed(2), i, 15);
          this.ctx.moveTo(-i, -3)
          this.ctx.lineTo(-i, 3)
          this.ctx.fillText((-i / factor).toFixed(2), -i, 15);
        }
        for (let i = yAxisStep; i < this.canvas.width - padding - xAxis || i < xAxis - padding; i += yAxisStep) {
          this.ctx.moveTo(-3, i)
          this.ctx.lineTo(3, i)
          this.ctx.fillText((i / factor).toFixed(2), -30, i);
          this.ctx.moveTo(-3, -i)
          this.ctx.lineTo(3, -i)
          this.ctx.fillText((-i / factor).toFixed(2), -30, -i);
        }
        this.ctx.stroke();
        this.ctx.scale(1, -1);
      },
      drawFillCircle(x, y, radius, factor) {
        this.ctx.beginPath();
        this.ctx.arc(x * factor, y * factor, radius, 0, Math.PI * 2);
        this.ctx.fill();
        this.ctx.stroke();
      },
      drawLineWithPointTo(x, y, radius, factor) {
        this.ctx.lineTo(x * factor, y * factor);
        this.ctx.stroke();
        this.drawFillCircle(x, y, 2, factor)
        this.ctx.moveTo(x * factor, y * factor);
      },
    }
  }
</script>

<style>
    .plotter {
        padding-top: 30px;
    }
</style>

