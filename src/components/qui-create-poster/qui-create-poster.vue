<template>
	<view class="content" v-if="isShow" @click.stop="close">
		<view v-if="showtip" class='tipToast' >长按图片即可保存！</view>
		<canvas  :style="{ width: canvasW + 'px', height: canvasH + 'px' }" canvas-id="my-canvas"   class="isCan"></canvas>
		<scroll-view scroll-y="true" style="max-height:90vh;width:100vw;position:fixed;left: 0;overflow:auto;" :class="canvasH>500?'alitem':'creatIMg'" v-if="tempImgShow">
			<img class="showImg" @longpress="longpress" :style="{ width: canvasW + 'px', height: canvasH + 'px' }" :src="tempImg" style="display: block;margin: 0 auto;border-radius:6px" />
		</scroll-view>
		<!-- <view v-if="tempImgShow" class="fixedBox">
			<view class="boxLeft"><button class="flexBtn btnLeft" hover-class="btnHover"    @click="longpress">长按图片保存</button></view>
			<view class="boxRight"><button class="flexBtn btnRight" hover-class="btnHover"  @click="close">关闭</button></view>
		</view> -->
	</view>
</template>

<script>
// import uniNoticeBar from '@/components/uni-notice-bar/uni-notice-bar.vue'
// export default {
//     components: {uniNoticeBar}
// }
	export default{
		props:{
			headerImg:{
				type: String,
				// default: 'https://pages.70e.com/storage/attachments/2020/08/08/L2TvT8xarUAOOTG475C4b36fv0tUE44p84xE1V9W_thumb.jpeg'
			},
			subTitle:{
				type: String,
			
			},
			username:{
				type: String,
			},
		},
        // watch:{
        //         username(newValue,oldValue) {
        //              if(newValue>6){
		// 	      	    this.name= newValue.split(0,6)+"...."
		// 	         }else{
		// 				 this.name= newValue
		// 			 }
        //     }
        // },
		data(){
			return {
				logo:'[大文案]',
				tempImgShow: false,
			    tempImg: '',
				// canvasW: 335,
				// canvasH: 534,
				canvasW: 270,
				canvasH: 430,
				ctx: null,
				isShow: false,
				strList:[],
				row:0,
				showtip:false,
				imgHeight:180,
			    imgWeight:270,
				wHeight:'',
				name:'',
			}
		},

		methods:{
	    stop(){
	        document.body.style.overflow='hidden';//禁止页面划动
	    },
	     move(){
	         document.body.style.overflow='';//出现滚动条
	      },
			// 绘制长图，
			upDateHeight(){
				let currentRow=Math.ceil((this.canvasH-this.imgHeight)/uni.upx2px(40))
				// 图片存在下（文字大于剩下高度的行，改变canvas高度）
				if(this.headerImg !=''){
					if(this.row >=currentRow){
						console.log("行数超过剩余canvas高度")
						this.canvasH = this.canvasH + (uni.upx2px(40) * (this.row - currentRow+4))
					}else {
						this.canvasH=430;
					}
				 }else{
					currentRow=Math.ceil(this.canvasH/uni.upx2px(40))
				// 图片不存在下（文字大于整体高度的行，改变canvas高度）
					 if(this.row >= currentRow){
                          this.canvasH = this.canvasH + (uni.upx2px(40) * (this.row -currentRow+1))
					}else {
						this.canvasH=430;
					}
				   }
			   console.log( this.canvasH)
		   },
			// 点击关闭按钮
			close(){
			   uni.hideLoading();
			   this.isShow=false;
			   this.tempImg="";
			   this.move();
			},

			//显示
			showCanvas(){
				// this.subTitle='';
				// this.headerImg='';
				// this.row=0
				this.strList=[];
			   this.isShow = true
				this.__init();
			},
			//初始化画布
			async __init(){
				
				uni.showLoading({
				    title: '正在生成海报...',
					mask: true
				})
				this.ctx = uni.createCanvasContext('my-canvas',this)
				// this.canvasW = uni.upx2px(550);
				// this.canvasH = uni.upx2px(860);
				//设置画布背景透明
				this.ctx.setFillStyle('rgba(255, 255, 255, 0)')
				//设置画布大小
				this.ctx.fillRect(0,0,this.canvasW,this.canvasH)
				//绘制圆角背景
				this.drawRoundRect(this.ctx, 0, 0, this.canvasW, this.canvasH,uni.upx2px(18),'#FFFFFF')
				// var hW = uni.upx2px(550);
				// var hH = uni.upx2px(350);
			    var hW = this.imgWeight;
				var hH = this.imgHeight;
				
				
				if(this.headerImg !=''){
					// if(this.subTitle=='分享图片'){
					//   hH=this.canvasH
					// }
	               //获取标题图片
				 let headerImg = await this.getImageInfo(this.headerImg);
				//绘制标题图
				this.drawRoundImg(this.ctx,headerImg.path,((this.canvasW-hW) / 2),((this.canvasW-hW) / 2),hW,hH,uni.upx2px(16))

				// 有图片下，文字高度铺满剩余的canvas高度，则不需要设置top值
				console.log(this.row)
				console.log(this.canvasH-this.imgHeight)
				console.log(this.row * uni.upx2px(40) )
				console.log(this.row * uni.upx2px(40) >=this.canvasH-this.imgHeight)
				 if(this.row * uni.upx2px(40) >=this.canvasH-this.imgHeight){
                     hH=0
					}else{
					// 有图片下，文字高度没有铺满剩余的canvas高度，则文字在剩余canvas高度正中间
					 hH=this.imgHeight+((this.canvasH - this.imgHeight-uni.upx2px(40))-(this.row * uni.upx2px(40)))/2;
					 console.log((this.canvasH - this.imgHeight-uni.upx2px(40))-(this.row * uni.upx2px(40)))
					}
				}else{
					console.log(this.row)
					// 没有图片下，文字高度铺满整个canvas高度，则不需要设置top值
					if(this.row * uni.upx2px(40) >=this.canvasH){
                     hH=0
					}else{
					// 没有图片下，文字高度没有铺满整个canvas高度，则文字在canvas正中间
					 hH=(this.canvasH - uni.upx2px(40)- (this.row * uni.upx2px(40)))/2;
					}
				}
			

				//绘制内容
				this.spliceStr(this.subTitle);
				this.ctx.setFontSize(12);
				this.ctx.setFillStyle('#666');
				this.ctx.textAlign="center";
				this.strList.forEach((element,index) => {
	             this.ctx.fillText(element,this.canvasW/2,(
					((this.canvasW-hW) / 2) + hH + uni.upx2px(40 * (index+0.6) )))
				
				});
				
				// 用户名称
				this.ctx.setFontSize(12);
				this.ctx.textAlign="left";
				this.ctx.setFillStyle('#666');
				this.ctx.fillText(this.username,(((this.canvasW-hW)/2)+uni.upx2px(20) ),(((this.canvasW-hW) / 2) + this.canvasH-uni.upx2px(40)))

				//大文案名称
				this.ctx.setFontSize(12);
				this.ctx.textAlign="right";
				this.ctx.setFillStyle('#666');
				this.ctx.fillText(this.logo,(((this.canvasW+hW)/2 )-uni.upx2px(10) ),(((this.canvasW-hW) / 2)  + this.canvasH-uni.upx2px(40)))
	
				//延迟渲染
				setTimeout(()=>{
					this.ctx.draw(true,()=>{
						uni.hideLoading();
						this.creatImage()
					})
				},500)
			},

			//图片
			drawRoundImg(ctx, img, x, y, width, height, radius){
				ctx.beginPath()
				ctx.save()
				// 左上角
				 ctx.arc(x, y , radius, Math.PI, Math.PI * 1.5)
				// 右上角
				ctx.arc(x + width, y + radius, radius, Math.PI * 1.5, Math.PI * 2)
				// 右下角
				ctx.arc(x + width , y + height , radius, 0, Math.PI * 0.5)
				// 左下角
				ctx.arc(x , y + height , radius, Math.PI * 0.5, Math.PI)
				ctx.stroke()
				ctx.clip()
				ctx.drawImage(img, x, y, width, height);
				ctx.restore()
				ctx.closePath()
			},
			
		  //截取字符串，按照br换号，截取
		   spliceStr(str){
			//   去掉话题span标签及内容 
			//  var spanReg = /<span[^>]*>(.|\n)*<\/span>/gi;
			 var spanReg=/<span[^>]*>.*?<\/span>/ig;
            //  仅去掉话题span标签保留其内容 
			// var spanReg = /(<\/?span.*?>)/gi;

			str=str.replace(spanReg, '').replace(/\s+/g, "").replace(/<p>/g,'').replace(/<\/p>/g,'');   //去除空格已经话题@等内容
			console.log(str)
			if(str.indexOf('<br>') !=-1){
			 let array=str.split('<br>');
			 var reg = new RegExp("<br>","g");
			  array.forEach(element => {
				if(element.length > 17){
				    for(let x=0;x<=Math.ceil(element.length/17);x++){
						 if(element.substr(x*17,17)!=""){
                            this.row=this.row+1;
					       this.strList.push(element.substr(x*17,17))
						 }
				    }
				 }
				 else{
					 if(element!=""){
                     	this.strList.push(element);
				        this.row=this.row+1;
					 } 

				 }               
			 });
			}else{
			let length=str.length,
			 row= Math.ceil(length/17)+1;
			 for(let i=0;i<row;i++){
				 if(str.substr(i*17,17)!=""){
                    this.row=this.row+1;
                     this.strList.push(str.substr(i*17,17))
				 }


			 }
  
			}
			console.log( this.row)
		},


		
			//矩形
			drawRoundRect(ctx, x, y, width, height, radius, color){
				ctx.save();
				ctx.beginPath();
				ctx.setFillStyle(color); 
				ctx.setStrokeStyle(color)
				// ctx.setLineJoin('round');  //交点设置成圆角
				ctx.setLineWidth(radius);
				ctx.strokeRect(x + radius/2, y + radius/2, width - radius , height - radius );
				ctx.fillRect(x + radius, y + radius, width - radius * 2, height - radius * 2);
				ctx.stroke();
				ctx.closePath();
			},
			//获取图片
			getImageInfo(imgSrc){
				return new Promise((resolve, reject) => {
					uni.getImageInfo({
						src: imgSrc,
						success: (image) => {
							resolve(image);
							// this.imgHeight=image.height;
							// this.imgWeight=image.width;
							console.log('获取图片成功')
							console.log(image)
						},
						fail: (err) => {
							reject(err);
							console.log('获取图片失败',err)
						}
					});
				});
			},
		
		// 保存图片
		longpress() {
			uni.saveImageToPhotosAlbum({
				filePath: this.tempImg,
				success: function(res) {
					uni.showToast({
						title: '保存相册成功'
					});
					console.log('save success');
				},
				fail(res) {
					console.log(res);
					if (res.errMsg == 'saveImageToPhotosAlbum:fail auth deny') {
						uni.showModal({
							title: '您需要授权相册权限',
							success(res) {
								if (res.confirm) {
									uni.openSetting({
										success(res) {},
										fail(res) {
											console.log(res);
										}
									});
								}
							}
						});
					}
				}
			});
		},
	
			//生成图片
			creatImage(){
				var that = this
				uni.canvasToTempFilePath({
					canvasId: 'my-canvas',
					quality: 1,
					fileType:"jpg",
					success: (res) => {
						console.log('生成图片'+res);
						// uni.hideLoading();
						that.tempImg = res.tempFilePath;
						setTimeout(() => {
							that.tempImgShow = true;
							that.showtip=true;
							that.stop();
							setTimeout(() => {
								that.showtip=false;
							}, 2000);
						}, 400);
					},
					fail: (res) => {
					 uni.showToast({
						title: '生成图片失败！请稍后再试'
					  });
					},
				},this);
			}
		}
	}
</script>

<style scoped lang="scss">
.content{
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,.4);
  z-index: 9999;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}
.fixedBox {
	width: 100%;
	height: 100rpx;
	position: fixed;
	bottom: 60rpx;
	left: 0;
	display: flex;
	z-index: 1000;
	.boxLeft,
	.boxRight {
		width: 50%;
		height: 100rpx;
		display: flex;
		align-items: center;
		position: relative;
		z-index: 1000;
		justify-content: center;
		.flexBtn {
			position: relative;
			z-index: 1000;
			width: 200rpx;
			height: 60rpx;
			text-align: center;
			line-height: 60rpx;
			font-size: 24rpx;
			// border-bottom: 6rpx #f58365 solid;
			border-radius: 32rpx;
			color: white;
			background: linear-gradient(to left, #f58365, #ff188a);
		}
	}
}

.btnHover {
	height: 55rpx !important;
	border-bottom: 0 #f6be3c solid !important;
	// transform: translateY(3px) translateZ(0px) !important;
}
.isCan {
	// border: 6px solid white;
	border-radius: 10px;
	z-index: 999;
	position: fixed;
	left: 0;
	// right: 0;
	// bottom: 0;
	top: -99999px;
	margin: auto;
	background-size: 100%;
}
.tipToast{
	width:350rpx;
	height:100rpx;
	line-height:100rx;
	text-align: center;
	z-index:100000;
	background-color:#000;
	color:#fff;
	position:fixed;
	top:40%;
	left:200rpx;
}
</style>
