<template>
	<view class="canvasWrap">
		<canvas :style="{backgroundColor: canvasBgColor}" @click="canvasClick" @touchstart="onTouchstart"
			@touchmove="onTouchmove" @touchend="onTouchend" :width="cvWidth + 'px'" :height="cvHeight + 'px'"
			canvas-id="firstCanvas" id="firstCanvas"></canvas>

		<view class="toolsWrap">
			<view class="tool_btns">
				<view class="tool_btn_item" @click="clearCanvas">清空</view>
				<view :style="{backgroundColor: eraserstatus == true?'gray':'royalblue'}" class="tool_btn_item"
					@click="eraserCanvas">橡皮</view>
				<view class="tool_btn_item" @click="saveAsPic">保存</view>
				<view @click="openColor" class="pickColor_btn" :style="{backgroundColor:penColor}"></view>
			</view>
			<view class="tool_pen">
				<text>宽度：</text>
				<slider :value="penWidth" min="1" max="100" step="1" style="width: 72vw;" show-value
					@change="updatePenWidth" />
			</view>
			<view class="undo_redo">
				<button :disabled="historyStack.length==0?true:false" @click="undo">撤销</button>
				<button :disabled="redoStack.length==0?true:false" @click="redo">恢复</button>
			</view>
		</view>
		<uv-pick-color ref="pickerColor" @confirm="confirm"></uv-pick-color>
	</view>
</template>

<script setup>
	import { onReady } from '@dcloudio/uni-app'
	import { ref } from 'vue';


	const ctx = ref() // canvas上下文
	const canvasBgColor = ref('#eeeeee') // canvas背景色
	const points = ref([]); // 路径点集合
	const penWidth = ref(10) // 画笔宽度
	const penColor = ref('#000000') // 画笔颜色 
	const pickerColor = ref() //颜色选择器颜色

	const windowWidth = ref()
	const windowHeight = ref()
	const cvWidth = ref()
	const cvHeight = ref()
	
	const uniPlatform = ref(uni.getSystemInfoSync().uniPlatform)
	onReady(() => {
		
		ctx.value = uni.createCanvasContext('firstCanvas')
		ctx.value.lineCap = 'round';
		ctx.value.lineJoin = 'round';

		uni.getSystemInfo({
			success(res) {
				windowWidth.value = res.windowWidth
				windowHeight.value = res.windowHeight
				cvWidth.value = res.windowWidth * res.devicePixelRatio
				cvHeight.value = res.windowHeight * res.devicePixelRatio

				ctx.value.setFillStyle(canvasBgColor.value); // 设置背景色，保存图片时是透明背景
				ctx.value.fillRect(0, 0, cvWidth.value, cvHeight.value); // 用矩形填充整个 Canvas
			}
		})
	})

	// === begin 撤销 和 恢复 ===
	/* 每画一笔，在onTouchend时用uni.canvasToTempFilePath保存为一张图片
	 */
	const historyStack = ref([])
	const redoStack = ref([])

	const undo = () => {
		if (historyStack.value.length > 0) {
			let lastDraw = historyStack.value.pop()
			redoStack.value.push(lastDraw)
			redraw(historyStack.value[historyStack.value.length - 1])
		}
	}
	const redo = () => {
		if (redoStack.value.length > 0) {
			let nextDraw = redoStack.value.pop();
			historyStack.value.push(nextDraw)
			redraw(historyStack.value[historyStack.value.length - 1])
		}
	}
	const redraw = (imgUrl) => {
		ctx.value.drawImage(imgUrl, 0, 0, windowWidth.value, windowHeight.value)
		ctx.value.draw()
	}
	// === end 撤销 和 恢复 ===

	// 定期清理冗余点的时间间隔（毫秒）
	const clearPointsInterval = 500;
	// 最后一次清理冗余点的时间戳
	let lastClearPointsTimestamp = 0;
	const confirm = (e) => {
		penColor.value = e.hex
	}
	const openColor = () => {
		pickerColor.value.open()
	}

	const canvasClick = (e) => {
		let startX = e.detail.x
		let startY = e.detail.y
		let clickPointColor = penColor.value
		if (eraserstatus.value == true) {
			clickPointColor = canvasBgColor.value
		}
		ctx.value.setLineWidth(penWidth.value);
		ctx.value.setStrokeStyle(clickPointColor);
		ctx.value.beginPath();
		ctx.value.arc(startX, startY, penWidth.value / 2, 0, 2 * Math.PI);
		ctx.value.closePath();
		ctx.value.setFillStyle(clickPointColor);
		ctx.value.fill();
		ctx.value.draw(true)
	}
	const onTouchstart = (e) => {
		let startX = e.changedTouches[0].x;
		let startY = e.changedTouches[0].y;
		let startPoint = {
			X: startX,
			Y: startY,
			width: penWidth.value,
			color: penColor.value,
		};
		points.value.push(startPoint);
		ctx.value.setLineWidth(penWidth.value);
		ctx.value.setStrokeStyle(penColor.value);
		if (eraserstatus.value == true) {
			// 模拟橡皮擦效果，清除指定区域内容Y
			startPoint.color = canvasBgColor.value
			ctx.value.setStrokeStyle(startPoint.color);
		}

		//每次触摸开始，开启新的路径
		ctx.value.beginPath();
	}
	const onTouchmove = (e) => {
		let moveX = e.changedTouches[0].x;
		let moveY = e.changedTouches[0].y;
		let movePoint = {
			X: moveX,
			Y: moveY,
			width: penWidth.value,
			color: penColor.value,
		};
		if (eraserstatus.value == true) {
			// 模拟橡皮擦效果，清除指定区域内容Y
			movePoint.color = canvasBgColor.value
		}
		points.value.push(movePoint);
		// historyStack.value.push(points.value)
		if (points.value.length >= 2) {
			draw()
		}
		// 定期清理冗余点
		const currentTimestamp = Date.now();
		if (currentTimestamp - lastClearPointsTimestamp > clearPointsInterval) {
			lastClearPointsTimestamp = currentTimestamp;
			// 保留最后一段路径，清除之前的路径
			const lastPath = points.value.slice(-2);
			points.value = lastPath;
		}
	}
	const draw = () => {
		let point1 = points.value[0];
		let point2 = points.value[1];

		points.value.shift()

		//通过在两个坐标点之间插值，获取它们之间的所有点，然后在每个点绘制圆
		let dx = point2.X - point1.X;
		let dy = point2.Y - point1.Y;
		let distance = Math.sqrt(dx * dx + dy * dy);
		let step = 3; // 控制描点的间隔
		for (let t = 0; t < distance; t += step) {
			let x = point1.X + (t / distance) * dx;
			let y = point1.Y + (t / distance) * dy;
			// 绘制圆
			ctx.value.beginPath();
			ctx.value.arc(x, y, point1.width / 2, 0, 2 * Math.PI);
			ctx.value.closePath();
			ctx.value.setFillStyle(point1.color);
			ctx.value.fill();
		}

		// 将路径连接起来并绘制
		ctx.value.beginPath();
		ctx.value.moveTo(point1.X, point1.Y)
		ctx.value.lineTo(point2.X, point2.Y);
		ctx.value.stroke();
		ctx.value.draw(true);
	}
	const onTouchend = () => {
		points.value = []
		redoStack.value = []
		uni.canvasToTempFilePath({
			canvasId: 'firstCanvas',
			quality: 1,
			destWidth: cvWidth.value,
			destHeight: cvHeight.value,
			success: (res) => {
				historyStack.value.push(res.tempFilePath)
			},
		})
	}

	const clearCanvas = () => {
		ctx.value.clearRect(0, 0, cvWidth.value, cvHeight.value);
		ctx.value.draw(true);
		points.value = []; // 清空当前路径，而不是整个历史路径
	};

	const eraserstatus = ref(false)
	const eraserCanvas = () => {
		eraserstatus.value = !eraserstatus.value
	}

	const updatePenWidth = (e) => {
		penWidth.value = e.detail.value;
	};

	
	const saveAsPic = () => {
		let scopeVal = 'scope.writePhotosAlbum'
		if(uniPlatform.value == 'mp-toutiao'){
			scopeVal = 'scope.album'
		}
		uni.getSetting({
			success(set_res) {
				if (set_res.authSetting[scopeVal]) {// 已经获得授权，直接存储
					saveCanvasToImg()
				}
				else if (set_res.authSetting[scopeVal] === undefined) {
					// 第一次运行，授权未定义，可以直接保存，系统会一次性询问用户权限
					saveCanvasToImg()
				} else {
					// 用户拒绝授权后，打开设置页可以看到授权提示开关
					uni.openSetting({
						success(open_res) {
							if (open_res.authSetting[scopeVal]) {
								saveCanvasToImg()
							}
							else { // 用户拒绝授权
								uni.showToast({
									title: '请开启图片访问权限',
									icon: 'none'
								})
							}
						},
						fail() {
							uni.showToast({
								title: '设置失败',
								icon: 'none'
							})
						}
					})
				}
			},
			fail() {
				uni.showToast({
					title: '设置失败',
					icon: 'none'
				})
			}
		})
	}

	function saveCanvasToImg() {
		uni.canvasToTempFilePath({
			canvasId: 'firstCanvas',
			quality: 1,
			destWidth: cvWidth.value,
			destHeight: cvHeight.value,
			success: (res) => {
				// res.tempFilePath 是图片文件的临时路径
				// 这里可以处理保存图片的逻辑，例如上传到服务器或本地存储
				uni.saveImageToPhotosAlbum({
					filePath: res.tempFilePath,
					success() {
						uni.showToast({
							title: '已保存到系统相册',
							icon: 'none'
						})
					},
				})
			}
		});
	}
</script>

<style scoped>
	.canvasWrap {
		display: flex;
		flex-direction: column;
		position: relative;
	}

	#firstCanvas {
		width: 100vw;
		height: 100vh;
	}

	.toolsWrap {
		width: 100vw;
		position: absolute;
		bottom: 0;
		left: 0;
		padding: 50rpx 20rpx;
		box-sizing: border-box;
		background-color: white;
		z-index: 999;
	}

	.toolsWrap .tool_btns {
		display: flex;
		justify-content: space-around;
	}

	.toolsWrap .tool_btns .tool_btn_item {
		display: flex;
		align-items: center;
		background-color: royalblue;
		padding: 8rpx 20rpx;
		border-radius: 16rpx;
		color: white;
	}

	/* 颜色选择器 */
	.toolsWrap .tool_btns .pickColor_btn {
		width: 50rpx;
		height: 50rpx;
		padding: 8rpx 20rpx;
		background-color: aqua;
	}

	.toolsWrap .tool_pen {
		display: flex;
		align-items: center;
	}

	.toolsWrap .tool_pen text {
		font-size: 16px;
	}

	.toolsWrap .undo_redo {
		display: flex;
		align-items: center;
		justify-content: center;
	}
</style>