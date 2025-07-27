<!-- eslint-disable @typescript-eslint/no-extra-non-null-assertion -->
<script setup lang="ts">
import { ref } from 'vue'

async function toBase64(file: File): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.readAsDataURL(file)
    reader.onload = () => resolve(reader.result as unknown as string)
    reader.onerror = (error) => reject(error)
  })
}

function dist(a: number, b: number, c: number, d: number): number {
  return Math.pow(Math.pow(c - a, 2) + Math.pow(d - b, 2), 1 / 2)
}

let fileName: string = ''
let loadedimg: HTMLImageElement | undefined = undefined
let boxes: [number, number, number, number, number][] = []
let currentbox = 0
const textvalue = ref('')

function redrawCanvas() {
  if (!loadedimg) return
  const canvas = document.getElementById('maincanva') as HTMLCanvasElement
  const context = canvas.getContext('2d')!!
  context.clearRect(0, 0, loadedimg.width, loadedimg.height)
  context.drawImage(loadedimg, 0, 0, loadedimg.width, loadedimg.height)
  let idx = 0
  for (const barycenter of boxes) {
    if (idx == currentbox) {
      context.strokeStyle = 'green'
    } else {
      context.strokeStyle = 'black'
    }
    context.lineWidth = 3
    context.strokeRect(
      barycenter[0],
      barycenter[1],
      barycenter[2] - barycenter[0],
      barycenter[3] - barycenter[1],
    )
    idx++
  }
}

function setValue() {
  console.log('Calling setvalue for', currentbox)
  boxes[currentbox][4] = parseInt(textvalue.value)
  currentbox = (currentbox + 1) % boxes.length
  const v = boxes[currentbox][4]
  textvalue.value = v == -1 ? '' : v.toString()
  redrawCanvas()
}

function loadImage(event: Event) {
  const file = (event.target as HTMLInputElement).files?.[0]
  if (!file) {
    return
  }
  fileName = file.name
  toBase64(file).then((b) => {
    const img = new Image()
    img.onload = function () {
      loadedimg = img
      boxes = []
      const canvas = document.getElementById('maincanva') as HTMLCanvasElement
      const context = canvas.getContext('2d')!!
      canvas.width = loadedimg.width
      canvas.height = loadedimg.height
      context.clearRect(0, 0, loadedimg.width, loadedimg.height)
      context.drawImage(loadedimg, 0, 0, loadedimg.width, loadedimg.height)
      const arr = context!!.getImageData(0, 0, img.width, img.height)
      const groups: [number, number][][] = []
      for (let j = 0; j < arr.height; j++) {
        for (let i = 0; i < arr.width; i++) {
          const idx = 4 * (i + j * arr.width)
          const r = arr.data[idx]
          const g = arr.data[idx + 1]
          const b = arr.data[idx + 2]
          const mask = 16 * 9 <= r && g <= 16 * 8 && b <= 16 * 8
          if (!mask) continue
          let found = false
          for (const group of groups) {
            if (found) break
            for (const elt of group) {
              if (dist(i, j, elt[0], elt[1]) < 15) {
                found = true
                group.push([i, j])
                break
              }
            }
          }
          if (!found) {
            groups.push([[i, j]])
          }
        }
      }
      for (const group of groups) {
        let xl = arr.width
        let yl = arr.height
        let xr = 0
        let yr = 0
        for (const elt of group) {
          const margin = 5
          const x = elt[0]
          const y = elt[1]
          xl = Math.max(Math.min(xl, x - margin), 0)
          xr = Math.min(Math.max(xr, x + margin), arr.width)
          yl = Math.max(Math.min(yl, y - margin), 0)
          yr = Math.min(Math.max(yr, y + margin), arr.width)
        }
        boxes.push([xl, yl, xr, yr, -1])
      }
      redrawCanvas()
    }

    img.src = b
  })
}

function exportData() {
  const text = boxes
    .sort((v, w) => v[4] - w[4])
    .map((arr) => arr[4] + ';' + arr.filter((_, i) => i != 4).join(','))
    .join('\n')
  const filename = `${fileName}.boxes.csv`

  const blob = new Blob([text], { type: 'text/plain' })
  const url = URL.createObjectURL(blob)

  const a = document.createElement('a')
  a.href = url
  a.download = filename
  a.click()

  URL.revokeObjectURL(url)
}
</script>

<template>
  <div class="main">
    <div class="drawer">
      <label for="files" class="btn">Select Image</label>
      <input id="files" type="file" accept="image/*" hidden @change="loadImage" />
      <div style="padding-left: 15px" />
      <button @click="exportData">Export data</button>
    </div>
    <div class="middle">
      <canvas id="maincanva"></canvas>
    </div>
    <div class="drawer">
      <span v-if="boxes.length > 0">
        Filling number for box {{ currentbox + 1 }} / {{ boxes.length }}
      </span>
      <div style="padding-left: 15px" />
      <input placeholder="Current box's number" v-model="textvalue" @keyup.enter="setValue" />
    </div>
  </div>
</template>

<style scoped>
.main {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  height: 100%;
}

.middle {
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
}

button,
.btn {
  background-color: blueviolet;
  color: white;
  padding: 10px;
  border-radius: 25px;
  font-size: 15px;
}
.drawer {
  padding: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
}
</style>
