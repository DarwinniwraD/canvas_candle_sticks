<template>
  <div id="demobox" style="overflow:hidden">
    <img id="logo" src="../assets/logo.png" style="display:none">
    <div class="canvasPlayground"  :style="canvasPlaygroundStyle">
      <canvas id="test"></canvas>
      <canvas id="mask"></canvas>
    </div>
    <!-- <div>Lorem ipsum dolor sit amet, consectetur adipisicing elit. Officia aliquam inventore id reiciendis laudantium, at voluptatum nam, dicta in, cum dignissimos excepturi odit. Laborum, qui! Sunt, sapiente. Voluptate, officia totam.</div> -->
  </div>
</template>
<script>
import { timeFormat } from "d3-time-format";
import { scaleLinear } from "d3-scale";
import data from '../services/data.json';
export default {
  name: "HelloWorld",
  data() {
    return {
      margin: 20,
      barSpacing: 1,
      marginVolume: 5
    };
  },
  computed: {
    width() {
      return window.innerWidth
    },
    height() {
      return 240
    },
    stemWidth() {
      return this.barWidth * 0.15
    },
    barWidth(){
      return (this.width - this.margin*2)/data.length - this.barSpacing * 2
    },
    dpi(){
      return window.devicePixelRatio
    },
    canvasPlaygroundStyle(){
      return {
        'overflow':'hidden' , 
        'height': `${this.height}px`,
        'width': `${this.width}px`
      }
    }
  },
  mounted() {
    const cmpgt = (a, b) => a > b ? a : b
    const cmplt = (a, b) => a > b ? b : a
    const dateDomain = data.reduce((last, cur) => {
      last.dateLow = last.dateLow ? cmplt(last.dateLow, cur.date) : cur.date;
      last.dateHigh = last.dateHigh ? cmpgt(last.dateHigh, cur.date) : cur.date
      last.highestHigh = last.highestHigh ? cmpgt(last.highestHigh, cur.high) : cur.high
      last.lowestLow = last.lowestLow ? cmplt(last.lowestLow, cur.low) : cur.low;
      last.volumnHigh = last.volumnHigh ? cmpgt(last.volumnHigh, cur.volume) : cur.volume
      last.volumnLow = last.volumnLow ? cmplt(last.volumnLow, cur.volume) : cur.volume;
      return last
    }, {
      dateLow: null,
      dateHigh: null,
      highestHigh: null,
      lowestLow: null,
      volumnHigh: null,
      volumnLow: null
    })
    const calcDateX = scaleLinear()
      .domain([dateDomain.dateLow, dateDomain.dateHigh])
      .range([0, this.width - this.barWidth]);
    const calcRY = scaleLinear()
      .domain([dateDomain.lowestLow, dateDomain.highestHigh])
      .range([this.height - this.margin, this.margin]);

    const dateRange = this.width - this.margin * 2;
    const dOpenCloseMin = (d) => Math.min(d['close'], d['open'])
    const dOpenCloseMax = (d) => Math.max(d['close'], d['open'])
    const dIsDown = (d) => d.close < d.open
    const dStemColor = (d) => dIsDown(d) ? '#ff2121' : '#1bd31b'
    // const dBodyColor = (d) => dIsDown(d) ? '#ff2121' : '#fff'
    const dBodyColor = (d) => dIsDown(d) ? '#ff2121' : '#1bd31b'
    const canvas = document.getElementById('test');
    canvas.setAttribute('height', this.height * this.dpi);
    canvas.setAttribute('width', this.width * this.dpi);
    if (canvas.getContext) {
      const context = canvas.getContext('2d');
      for (let index = 0; index < data.length; index++) {
        const candleItemData = data[index];
        context.beginPath();
        const dateX = calcDateX(candleItemData.date);
        const dateY = calcRY(dOpenCloseMax(candleItemData));
        const dateY1 = calcRY(dOpenCloseMin(candleItemData));
        const stemColor = dStemColor(candleItemData)
        const bodyColor = dBodyColor(candleItemData)
        
        const centrailPoint = (this.barWidth / 2) - (this.stemWidth / 2)


        /* stem */
        context.beginPath();
        context.moveTo(dateX + centrailPoint, calcRY(candleItemData.high));
        context.lineTo(dateX + centrailPoint, calcRY(candleItemData.low));
        context.lineWidth = this.stemWidth
        context.strokeStyle = stemColor;
        context.stroke();
        // context.fillRect(dateX + centrailPoint, calcRY(candleItemData.high), this.stemWidth, calcRY(candleItemData.low) - calcRY(candleItemData.high));
        // context.fillStyle = stemColor;
        context.clearRect(dateX, dateY, this.barWidth, dateY1 - dateY);

        if (dIsDown(candleItemData)) {
          
          /* entity candle */
          context.beginPath()
          context.moveTo(dateX + centrailPoint, dateY);
          context.lineTo(dateX + centrailPoint, dateY1);
          context.lineWidth = this.barWidth;
          context.closePath();
          context.strokeStyle = bodyColor;
          context.stroke();
          // context.fillRect(dateX, dateY, this.barWidth, dateY1 - dateY);
          // context.fillStyle = bodyColor;
        } else {

          /* blank candle */
          context.strokeRect(dateX, dateY, this.barWidth, dateY1 - dateY);
          context.strokeStyle = stemColor;
        }

        
      }

      /* embeded image */
      // var rectBgImg = new Image();
      // var rectBgImg = document.getElementById('logo');
      // context.drawImage(rectBgImg, 0, 0);



      /***
       * 
       * draw k-line 
       *
       ***/

      //config
      context.lineWidth = 1;
	    context.lineJoin = 'round';
	    context.lineCap = 'round';
	    context.strokeStyle = 'blue';
      context.fillStyle = 'blue';
      //draw
      context.beginPath();
      context.moveTo(calcDateX(data[0].date), calcRY(data[0].high));
      let lineCursorPosition = 0;
      for (let index = 0; index < data.length - 2; index++) {
        lineCursorPosition = index
        const element = data[index];
        const nextSibling = data[index+1];
        const controlPonitX = (calcDateX(element.date) + calcDateX(nextSibling.date)) /2
        const controlPonitY = (calcRY(element.high) + calcRY(nextSibling.high)) /2
        context.quadraticCurveTo(calcDateX(element.date), calcRY(element.high), controlPonitX, controlPonitY);
        
      }
      // For the last 2 points
      context.quadraticCurveTo(calcDateX(data[lineCursorPosition].date), calcRY(data[lineCursorPosition].high), calcDateX(data[lineCursorPosition+1].date), calcRY(data[lineCursorPosition + 1].high));

      context.stroke();

      /* draw bar chart */
      for (let index = 0; index < data.length; index++) {
        const element = data[index];
        const dateX = calcDateX(element.date);
        const dateY = calcRY(element.high) + 100;
        context.strokeRect(dateX, dateY, this.barWidth, this.height - dateY);
        // context.strokeStyle = stemColor;
        context.strokeStyle = '#ff2121';
      }


      /* linear Gradient */
      // context.save();
      const lingrad = context.createLinearGradient(100, 150, 100, 170);
      lingrad.addColorStop(0, 'rgba(255,255,255, 0)');
      lingrad.addColorStop(1, 'rgba(0, 171, 235, 0.6)');
      context.beginPath();
      context.moveTo(100, 150);
      context.lineTo(180, 150);
      context.lineTo(200, 170);
      context.lineTo(80, 170);
      context.closePath();
      // context.strokeStyle = lingrad;
      // context.stroke();
      context.fillStyle = lingrad;
      context.fill();
      context.restore();




    }


    /* mask */

    const mask = document.getElementById('mask');
    mask.setAttribute('height', this.height * this.dpi);
    mask.setAttribute('width', this.width * this.dpi);
    if (mask.getContext) {
      const maskContext = mask.getContext('2d');

      /* cross line */
      maskContext.lineWidth = 1
      maskContext.strokeStyle = '#ff2121';
      maskContext.lineJoin = 'butt';
      maskContext.lineCap = 'butt';
      // maskContext.fill();
      // maskContext.fillStyle='#ff2121';
      const handleCrossline = e => {
        maskContext.save();
        maskContext.clearRect(0, 0, this.width, this.height);
        const canvasRecPos = canvas.getBoundingClientRect();
        // const x = e.touches[0].clientX - canvasRecPos.left
        const x = e.clientX - canvasRecPos.left
        const xlabel = new Date(parseInt(calcDateX.invert(x))).getHours()
        maskContext.textAlign = 'center'
        maskContext.font = '30px serif'
        maskContext.fillText(xlabel, x, 20);

        // const y = e.touches[0].clientY - canvasRecPos.top
        const y = e.clientY - canvasRecPos.top
        const ylabel = parseFloat(calcRY.invert(y)).toFixed(10)
        maskContext.textAlign = 'start'
        maskContext.font = '30px serif'

        y>40 ? maskContext.fillText(`$${ylabel}`, 10, y - 10) : maskContext.fillText(`$${ylabel}`, 10, y + 30);
        maskContext.setLineDash([5, 3]);/*dashes are 5px and spaces are 3px*/
        // horizational line
        maskContext.beginPath();
        maskContext.moveTo(x,0);
        maskContext.lineTo(x, this.height);
        maskContext.stroke();

        //vertical line
        maskContext.beginPath();
        maskContext.moveTo(0,y);
        maskContext.lineTo(this.width, y);
        maskContext.stroke();
        maskContext.restore();
      }

      // mask.addEventListener('touchmove', handleCrossline)
      mask.addEventListener('mousemove', handleCrossline)
      // mask.addEventListener('touchend', ()=>{
      //   maskContext.clearRect(0, 0, this.width, this.height)
      // })
      mask.addEventListener('mouseleave', ()=>{
        maskContext.clearRect(0, 0, this.width, this.height)
      })
    }

  },
  methods: {
  }
};
</script>
<!-- Add "scoped" attribute to limit CSS to this component only -->
<style>
body, html{
  margin: 0;
  padding: 0;
}
h1,
h2 {
  font-weight: normal;
}

ul {
  list-style-type: none;
  padding: 0;
}

li {
  display: inline-block;
  margin: 0 10px;
}

a {
  color: #42b983;
}
.canvasPlayground{
  position: relative;
}
#mask{
  position: absolute;
  left: 0;
  top: 0;
}
</style>