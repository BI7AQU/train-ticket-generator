<template>
<!-- 红票（3代软质车票）） -->
  <div
    v-show="activeTab == 'ticket2D'"
    class="flex flex-col md:flex-row gap-4 justify-between w-full py-4"
    :style="{ minHeight: canvasHeight}"
  >
    <div class="ticket-container">
      <div class="canvas-aspect">
        <canvas ref="ticketCanvas" :width="canvasWidth" :height="canvasHeight"></canvas>
      </div>
    </div>
    <div class="ticket-container">
      <div class="canvas-aspect">
        <canvas ref="ticketBackCanvas" :width="canvasWidth" :height="canvasHeight"></canvas>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, watch } from 'vue'
import { pinyin } from 'pinyin-pro'
import QRCode from 'qrcode'
import {
  drawCustomText,
  getTextWidth,
  drawParagraph,
  drawTrapezoid,
  drawRoundRect,
} from '@/utils/canvas'
import { maskedId, getStationSpacing } from '@/utils/common'
import CRHImage from '@/assets/img/redTicket.png'
import Tabs from '@/components/common/Tabs.vue'

const props = defineProps({
  ticketInfo: {
    type: Object,
    required: true,
  },
})

const activeTab = ref('ticket2D')

const protrusionHeight = 40
const protrusionWidth = 10
const canvasWidth = 876
const canvasHeight = 539
const leftOffset = 75
let topOffset = 45

const ticketCanvas = ref<HTMLCanvasElement>()
const ticketBackCanvas = ref<HTMLCanvasElement>()

const destroyCanvas = () => {
  const canvas = ticketCanvas.value
  if (!canvas) return
  const ctx = canvas.getContext('2d')
  ctx?.clearRect(0, 0, canvas.width, canvas.height)
}

const drawTicket = () => {
  destroyCanvas()
  const canvas = ticketCanvas.value
  if (!canvas) return
  const ctx = canvas.getContext('2d') as CanvasRenderingContext2D

  ctx.fillStyle = '#fff'
  ctx.fillRect(0, 0, canvas.width, canvas.height)

  const backgroundImage = new Image()
  backgroundImage.src = CRHImage
  backgroundImage.onload = () => {
    const paddingX = canvasWidth - 10
    const paddingY = canvasHeight - 10
    const imgWidth = canvasWidth - 2 * paddingX
    const imgHeight = canvasHeight - 2 * paddingY
    const centerX = paddingX
    const centerY = paddingY

    ctx.globalAlpha = 1.0
    ctx.drawImage(backgroundImage, centerX, centerY, imgWidth, imgHeight)

    drawTicketDetails(canvas, ctx)
  }
}

const drawTicketDetails = (canvas: HTMLCanvasElement, ctx: CanvasRenderingContext2D) => {
  // 票面信息绘制

  // 左上角红色 ID
  ctx.fillStyle = 'rgba(255, 0, 0, 0.5)'
  ctx.font = ' 42px Arial'
  ctx.fillText(props.ticketInfo.redId, 80, 92)

  // 检票口
  ctx.font = '32px SimSun'
  if (props.ticketInfo.checkGate !== '') {
    const checkGateText = '检票:' + props.ticketInfo.checkGate
    const checkGateWidth = getTextWidth(ctx, checkGateText)
    drawCustomText(ctx, checkGateText, canvasWidth - checkGateWidth - 100, 85)
  } 
  
  // 始发站、车次、目的地
  const startStation = props.ticketInfo.startStation
  const endStation = props.ticketInfo.endStation
  ctx.font = '45px SimHei'
  let left = 100
  if (startStation.length == 5) left = 70
  drawCustomText(ctx, startStation, left, topOffset + 90, getStationSpacing(startStation))
  left = canvasWidth / 2 + 120
  if (endStation.length == 5) left = canvasWidth / 2 + 80
  drawCustomText(ctx, endStation, left, topOffset + 90, getStationSpacing(endStation))

  ctx.font = '30px FangSong'
  let startStationPinyin = pinyin(startStation, { toneType: 'none' }).replace(/ /g, '')
  let endStationPinyin = pinyin(endStation, { toneType: 'none' }).replace(/ /g, '')
  if (startStationPinyin)
    startStationPinyin = startStationPinyin[0].toUpperCase() + startStationPinyin.slice(1)
  if (endStationPinyin)
    endStationPinyin = endStationPinyin[0].toUpperCase() + endStationPinyin.slice(1)
  const startStationPinyinWidth = getTextWidth(ctx, startStationPinyin)
  const endStationPinyinWidth = getTextWidth(ctx, endStationPinyin)

  drawCustomText(ctx, startStationPinyin, 200 - startStationPinyinWidth / 2, topOffset + 120, -1)
  drawCustomText(ctx, endStationPinyin, canvasWidth / 2 + 220 - endStationPinyinWidth / 2, topOffset + 120, -1)

  ctx.font = '42px Simsun'
  const trainNumber = props.ticketInfo.trainNumber
  const trainNumberWidth = getTextWidth(ctx, trainNumber) + 2 * trainNumber.length
  drawCustomText(ctx, trainNumber, canvasWidth / 2 - trainNumberWidth / 2, topOffset + 97, 2)
  const arrowStartX = canvasWidth / 2 - 63
  const arrowStartY = topOffset + 108
  const arrowLength = 126
  const arrowHeight = 5
  const arrowWidth = 15

  ctx.beginPath()
  ctx.moveTo(arrowStartX, arrowStartY)
  ctx.lineTo(arrowStartX + arrowLength, arrowStartY)
  ctx.lineTo(arrowStartX + arrowLength - arrowWidth, arrowStartY - arrowHeight)
  ctx.stroke()

  ctx.font = '30px Simsun'
  drawCustomText(ctx, '站', 270, topOffset + 85)
  drawCustomText(ctx, '站', canvasWidth - 155, topOffset + 85)

  // 日期、时间
  ctx.font = '40px Simhei'
  const date = new Date(props.ticketInfo.date)
  const year = date.getFullYear().toString()
  const month =
    date.getMonth() + 1 > 9 ? (date.getMonth() + 1).toString() : '0' + (date.getMonth() + 1)
  const day = date.getDate() > 9 ? date.getDate().toString() : '0' + date.getDate()
  drawCustomText(ctx, year, leftOffset, topOffset + 160, -2)
  drawCustomText(ctx, month, leftOffset + 108, topOffset + 160, -2)
  drawCustomText(ctx, day, leftOffset + 170, topOffset + 160, -2)
  drawCustomText(ctx, props.ticketInfo.time, leftOffset + 238, topOffset + 160, -2)

  // 座位号、车厢号、座位类型
  const seatCarriage = props.ticketInfo.seatCarriage.toString().padStart(2, '0')
  let seatNumber = props.ticketInfo.seatNumber.toString().padStart(3, '0')
  drawCustomText(ctx, seatCarriage, canvasWidth / 2 + 105, topOffset + 160, -2)
  // 如果最后一位是字母，进行处理
  if (seatNumber === '000') {
    ctx.font = '32px FangSong GB2312'
    drawCustomText(ctx, '无座', canvasWidth / 2 + 167, topOffset + 157, -3)
  }
  else if (isNaN(parseInt(seatNumber.slice(-1)))) {
    const seatNumberLast = seatNumber.slice(-1)
    seatNumber = seatNumber.slice(0, -1)
    drawCustomText(ctx, seatNumber, canvasWidth / 2 + 167, topOffset + 160, -3)
    ctx.font = '32px SimSun'
    drawCustomText(ctx, seatNumberLast, canvasWidth / 2 + 167 + 38, topOffset + 157, -3)
  } 
    else {
    drawCustomText(ctx, seatNumber, canvasWidth / 2 + 167, topOffset + 160, -3)
  }

  ctx.font = '28px FangSong GB2312'
  const seatType = props.ticketInfo.seatType
  const seatTypeWidth = getTextWidth(ctx, seatType)
  drawCustomText(ctx, seatType, 650 - seatTypeWidth / 2, topOffset + 190)

  ctx.font = '21px SimSun'
  drawCustomText(ctx, '年', leftOffset + 80, topOffset + 153)
  drawCustomText(ctx, '月', leftOffset + 145, topOffset + 153)
  drawCustomText(ctx, '日', leftOffset + 208, topOffset + 153)
  drawCustomText(ctx, '开', leftOffset + 335, topOffset + 153)
  drawCustomText(ctx, '车', leftOffset + 505, topOffset + 153)
  if(seatNumber !== '000'){
  drawCustomText(ctx, '号', leftOffset + 584, topOffset + 153)
  }

  // 票价、额外信息
  ctx.font = '40px SimSun'
  drawCustomText(ctx, '¥', leftOffset + 10, topOffset + 195)
  ctx.font = '40px Simhei'
  const price = (props.ticketInfo.price || 0).toFixed(1).toString()
  const priceWidth = getTextWidth(ctx, price)
  drawCustomText(ctx, price, leftOffset + 45, topOffset + 195, -2)
  ctx.font = '21px FangSong'
  drawCustomText(ctx, '元', leftOffset + 37 + priceWidth, topOffset + 190)
  ctx.font = '32px SimSun'
  if (props.ticketInfo.isStudent) {
    const studentText = '学'
    const studentWidth = getTextWidth(ctx, studentText)
    drawCustomText(ctx, studentText, canvasWidth / 2 - studentWidth - 60, topOffset + 190)

    const discountText = '折'
    const discountWidth = getTextWidth(ctx, discountText)
    drawCustomText(ctx, discountText, canvasWidth / 2 - discountWidth - 10, topOffset + 190)
  } 
  else if (props.ticketInfo.isDiscount) {
    const discountText = '折'
    const discountWidth = getTextWidth(ctx, discountText)
    drawCustomText(ctx, discountText, canvasWidth / 2 - discountWidth - 20, topOffset + 190)
  }

  ctx.font = '20px Times New Roman'

  // 仅供报销使用
  ctx.font = '32px SimSun'
  drawCustomText(ctx, '限乘当日当次车', leftOffset + 5, 270)

  // 仅供纪念使用
  drawCustomText(ctx, '仅供纪念使用', 340, 305)

  ctx.font = '40px Maxim Sans Regular'
  // 身份证号码和姓名
  const id = maskedId(props.ticketInfo.passengerId)
  const idWidth = getTextWidth(ctx, id)
  drawCustomText(ctx, id, leftOffset, 370, -3)
  ctx.font = '36px SimSun'
  drawCustomText(ctx, props.ticketInfo.passengerName, leftOffset + idWidth + 10 - 48, 370)

  // 虚线框
  const dashWidth = 500
  const dashHeight = 74
  const dashLeft = 88
  ctx.font = ' 28px SimSun'
  ctx.setLineDash([13.3, 4.4])
  ctx.strokeRect(dashLeft, 384, dashWidth, dashHeight)
  ctx.setLineDash([])
  const text1 = '买票请到12306 发货请到95306'
  const text1Width = getTextWidth(ctx, text1)
  const text2 = '中国铁路祝您旅途愉快'
  const text2Width = getTextWidth(ctx, text2)
  drawCustomText(ctx, text1, dashLeft + dashWidth / 2 - text1Width / 2, 414)
  drawCustomText(ctx, text2, dashLeft + dashWidth / 2 - text2Width / 2, 449)

  // 二维码，真实车票二维码使用AES加密
  const qrCodeText = props.ticketInfo.qrCodeId
  const qrCodeWidth = 185
  const qrCodeOptions = {
    width: qrCodeWidth,
    errorCorrectionLevel: 'H',
    maskPattern: 5,
    margin: 0,
    color: {
      dark: '#000000b0', // 二维码颜色
      light: '#FFFFFF00', // 背景透明
    },
  }
  QRCode.toDataURL(qrCodeText, qrCodeOptions as any, (err, url) => {
    if (err) throw err
    const qrImage = new Image()
    qrImage.src = url
    qrImage.onload = () => {
      ctx.drawImage(qrImage, dashLeft + dashWidth + 44, 303, qrCodeWidth, qrCodeWidth)
    }
  })

  // 车票ID和售票点
  ctx.font = ' 30px FangSong'
  const TicketID = props.ticketInfo.id
  const IDWidth = getTextWidth(ctx, TicketID)
  const bottomOffset = 5
  drawCustomText(ctx, props.ticketInfo.id, leftOffset, canvasHeight - 55 + bottomOffset)
    ctx.font = ' 30px FangSong GB2312'
  drawCustomText(ctx, props.ticketInfo.ticketOffice + '售', leftOffset + IDWidth + 30, canvasHeight - 55 + bottomOffset)

}

const drawTicketBack = () => {
  const canvas = ticketBackCanvas.value
  if (!canvas) return
  const ctx = canvas.getContext('2d') as CanvasRenderingContext2D

  ctx.fillStyle = '#fff'
  ctx.fillRect(0, 0, canvas.width, canvas.height)

  // 矩形
  drawRoundRect(ctx, 20, 10, canvasWidth - 40, canvasHeight - 20, 0, 'rgba(255, 255, 255, 1)')

  // 乘车须知
  ctx.font = '35px SimSun'
  const text = '乘车须知：'
  const textWidth = getTextWidth(ctx, text)
  const color = [0, 0, 0]
  drawCustomText(ctx, text, 40, 80, 0, color)

  ctx.font = '25px SimSun'
  const paragraph =
    '☆请妥善保管车票。☆请凭车票和本人有效身份证件原件乘车，如改签、变更到站或退票请提前办理。☆正在开发中，敬请期待！'

  const paragraphs = paragraph
    .split('☆')
    .filter((p) => p !== '')
    .map((p) => '☆' + p)
  drawParagraph(ctx, paragraphs, 40, 45, 140, 35, canvasWidth - 90, -1, color)
}

onMounted(() => {
  drawTicket()
  drawTicketBack()
})

watch(
  () => props.ticketInfo,
  () => {
    drawTicket()
  },
  { deep: true },
)
</script>

<style scoped>
.ticket-container {
  @apply flex justify-center items-center w-full;
}
.canvas-aspect {
  width: 100%;
  aspect-ratio: 876 / 539;
  position: relative;
}
.canvas-aspect canvas {
  width: 100%;
  height: 100%;
  display: block;
  position: absolute;
  left: 0;
  top: 0;
}
</style>
