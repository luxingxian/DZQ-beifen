<template>
  <view class="banner-container">
    <swiper :style="{width: '100vw', height: '100vh'}" 
      :previous-margin="swiperConfig.previousMargin"
      :next-margin="swiperConfig.nextMargin"
      @change="swiperChange" 
      @animationfinish="animationfinish">
      <swiper-item v-for="(item, i) in bannerList" :key="i" class="item"  >
        <!-- 1.当前展示为第1项时，bannerList最后一项和第二项的justifyContent值分别为flex-end和flex-start，其余项值为center -->
        <!-- 2.当前展示为最后一项时，bannerList倒数第2项和第1项的justifyContent值分别为flex-end和flex-start，其余项值为center -->
        <!-- 3.当前展示为其他项（非第1和最后1项）时，bannerList当前项的前1项和后1项的justifyContent值分别为flex-end和flex-start，其余项值为center -->
        <!-- 4.padding值也需要根据不同项设定不同值，但理同justifyContent -->
      <view class="item-box"
       :style="{
              transform: curIndex===i?'scale(1)':'scale(0.85)',
              transitionDuration: '.3s',
              transitionTimingFunction: 'ease',
            }" 
        @click="toDetails(item._jv.id)"
      >
             <view class="image-container" 
		        	:class="[curIndex===0?((i===listLen-1)?'item-left':(i===1?'item-right':'item-center')):(curIndex===listLen-1?(i===0?'item-right':(i===listLen-2?'item-left':'item-center')):(i===curIndex-1?'item-left':(i===curIndex+1?'item-right':'item-center')))]">
              <image  fade-show='true'  :src="item.firstPost.images[0].url" 
              class="slide-image" 
           
             />
            </view>
            <!-- 滚动 -->
         	<!-- <scroll-view scroll-y="true" class="desc-wrap hideAndShowDesc"  >
                   <text class="desc">{{item.firstPost.content}}</text>
	        	</scroll-view> -->

              
              <!-- 不滚动 -->
               <view class="desc-wrap hideAndShowDesc" >
                      <!-- 使用富文本来解析HTML内容 -->
                      <rich-text :nodes="item.firstPost.contentHtml | changeStr" class="desc"></rich-text>
                      <!-- <view class="desc">{{item.firstPost.contentHtml | changeStr}}</view> -->
               </view>
	      

           <view  class="foot" style="display:flex;justify-content: space-between;position: absolute;bottom:20rpx;left:30rpx;right:40rpx">
          <view
            :class="[
              'themeItem__footer__themeType1__item',
              'themeItem__footer__themeType1__great',
              'themeItem__footer__themeType1__greated',
            ]"
            @click.stop="threadLikeClick(
                item.firstPost.images[0].type_id,
                item.firstPost.canLike,
                item.firstPost.isLiked,
              )"
          >
            <qui-icon class="foot-qui-icon" name="icon-liked" size="28" v-if="item.firstPost.isLiked"></qui-icon>
            <qui-icon class="foot-qui-icon" name="icon-like" size="28" v-else></qui-icon>
            {{ item.firstPost.isLiked ? t.giveLikeAlready : t.like }}
            {{item.firstPost.likeCount}}
          </view>

          <view
            class="themeItem__footer__themeType1__item themeItem__footer__themeType1__comment"
           
          >
           <qui-icon class="foot-qui-icon" name="icon-comments" size="28" ></qui-icon>
            {{ t.comment }}
            {{item.postCount-1 }}
          </view>
           <view
            class="themeItem__footer__themeType1__item themeItem__footer__themeType1__share"
            @click.stop="handleClickShare(item)"
          >
            <qui-icon class="foot-qui-icon" name="icon-share1" size="28" color="#AAA"></qui-icon>
            {{ t.share }}
          </view>
        </view>
      </view>
    
      </swiper-item>
       <qui-toast ref="toast"></qui-toast>
    </swiper>
     <view class="getmore" v-if="flag">{{type==1?'加载更多~':'没有更多数据了~'}}</view>
  
  </view>
</template>
<script>
	import { mapState, mapMutations } from 'vuex';
	import user from '@/mixin/user';
	import forums from '@/mixin/forums';
	import detectionModel from '@/mixin/detectionModel';
	// #ifdef H5
	import wxshare from '@/mixin/wxshare-h5';
	import loginAuth from '@/mixin/loginAuth-h5';
	// #endif
	// #ifndef MP-WEIXIN
	import appCommonH from '@/utils/commonHelper';
	// #endif
	import uniPopupDialog from '@/components/uni-popup/uni-popup-dialog';
	import { getCurUrl } from '@/utils/getCurUrl';
	

export default {
   components: { uniPopupDialog },
   mixins: [
     user,
     forums,
     // #ifdef H5
     wxshare,
     loginAuth,
     // #endif
     detectionModel,
   ],
   // #ifndef MP-WEIXIN
   utils: [appCommonH],
   // #endif
  props: {
	bannerList: {
		type: Array,
		default () {
			return []
		}
	},
	swiperConfig: {
		type: Object,
		default () {
			return {
				previousMargin: '75rpx',
				nextMargin: '75rpx'
			}
		}
	},
	scaleX: {
		type: String,
		default: (634 / 550).toFixed(4)
	},
	scaleY: {
		type: String,
		default: (378 / 328).toFixed(4)
  },


  },
  computed:{
       t() {
      return this.i18n.t('topic');
    },
      thread() {
      const thread = this.$store.getters['jv/get'](`threads/${this.threadId}`);
      if (thread.rewardedUsers) {
        this.rewardedUsers = thread.rewardedUsers;
      }
      if (thread.firstPost) {
        this.likedUsers = thread.firstPost.likedUsers;
      }
      return thread;
    },
	listLen () {
		return this.bannerList.length
	}
  },

  data () {
    return {    
 
      curShareId:0,  //点击分享时，的文章id
      curIndex: 0,
      descIndex: 0,
      isDescAnimating: false,
      likedUsers: [],
      flag:false,   //是否显示正在加载或没有更多数据
      type:1,          //1有数据，正在加载 2.没有数据
    }
  },
  filters: {
       changeStr(str) {
        var spanReg=/<span[^>]*>.*?<\/span>/ig;
        // return str.replace(spanReg, '').replace(/\s+/g, "").replace(/<br>/g, "");
        return str.replace(spanReg, '').replace(/\s+/g, "");
        }
     },
  methods: {
   // 分享按钮点击事件
   handleClickShare(item) {
     this.$emit('shareClick',item);
   },

    // 是否加载更多
    show(){
      var that=this;
      setTimeout(() => {
        that.flag=true;   
        // 有更多数据时，就加载
        if(that.type==1){
					that.$emit('bannerMore')
        }
        
      }, 200);
    },


  // 点赞按钮点击事件
    threadLikeClick(postId, canLike, isLiked) {
      if (!this.$store.getters['session/get']('isLogin')) {
        // #ifdef MP-WEIXIN
        this.$store.getters['session/get']('auth').open();
        // #endif
        // #ifdef H5
        if (!this.handleLogin(getCurUrl())) {
          return;
        }
        // #endif
      }
      this.postOpera(postId, '1', canLike, isLiked);
    },
        // post操作调用接口（包括type 1主题点赞，3删除回复，4回复点赞）
    postOpera(id, type, canStatus, isStatus, post) {
      if (type === '1' && !canStatus) {
        // 没有主题点赞权限
        this.$refs.toast.show({ message: this.t.noThreadLikePermission });
        return;
      }
      if (type === '4' && !canStatus) {
        // 没有评论点赞权限
        this.$refs.toast.show({ message: this.t.noReplyLikePermission });
        return;
      }
      const jvObj = {
        type: 'posts',
        id,
      };
      let params = {};
      if (type === '1') {
        params = {
          _jv: jvObj,
          isLiked: !isStatus,
        };
      } else if (type === '3') {
        params = {
          _jv: jvObj,
          isDeleted: !isStatus,
        };
      } else if (type === '4') {
        params = {
          _jv: jvObj,
          isLiked: !isStatus,
        };
      }
      this.$store
        .dispatch('jv/patch', params)
        .then(data => {
          console.log(data)
          if (type === '1') {
            // 主题点赞
            this._updateLikedUsers(data.isLiked);
          } else if (type === '2') {
            if (data.isDeleted) {
              uni.navigateBack({
                delta: 1,
              });

              // uni.navigateBack({
              //   url: `/pages/home/index`,
              // });
            } else {
              // 主题删除失败
            }
          } else if (type === '3') {
            const postArr = post;
            postArr.isDeleted = data.isDeleted;
            post = postArr;
          } else if (type === '4') {
            console.log("111")
            const postArr = post;
            postArr.isLiked = data.isLiked;
            postArr.likeCount = data.likeCount;
            post = postArr;
          }
        })
        .catch(err => {
          console.log(err);
        });
    },
    _updateLikedUsers(liked = true) {
      this.isLiked = liked;
      // 主题点赞
      if (this.isLiked) {
        this.likedUsers.unshift(this.user);
        // this.thread.firstPost._jv.relationships.likedUsers.data.unshift({
        //   type: this.user._jv.type,
        //   id: this.user._jv.id,
        // });
      } else {
        this.likedUsers.forEach((value, key, item) => {
          value.id == this.user.id && item.splice(key, 1);
        });
        // this.thread.firstPost._jv.relationships.likedUsers.data.forEach((value, key, item) => {
        //   value.id == this.user.id && item.splice(key, 1);
        // });
      }
      this.$forceUpdate();
    },

    toDetails(id){
     let url=`/pages/topic/index?id=${id}`;
     uni.navigateTo({
        url,
      });
    },

    // 切换swiper
    swiperChange (e) {
      const that = this
      this.curIndex = e.mp.detail.current;
      if(this.curIndex==this.bannerList.length-1 ){
        this.show();
      }else{
         this.flag=false;
      }
      this.isDescAnimating = true
      let timer = setTimeout(function () {
        that.descIndex = e.mp.detail.current
        clearTimeout(timer)
      }, 150)
    },
    animationfinish (e) {
      this.isDescAnimating = false
    },

    getBannerDetail (index) {
	  uni.showLoading({
		title: '将前往详情页面',
		duration: 2000,
		mask: true
	  })
    }
  }
}
</script>
<style lang="scss" scoped>
.getmore{
  position: absolute;
  top: 50%;
  width: 25px;
  display: flex;
  align-items: center;
  justify-content: end;
  right: 0;
  bottom: 50%;
  font-size: 14px;
  color: rgb(170, 170, 170);
  }

.themeItem__footer__themeType1__item{
  font-family: -apple-system, blinkmacsystemfont, "Helvetica Neue", arial, "PingFang SC", "Hiragino Sans GB", stheiti, "Microsoft YaHei", "Microsoft JhengHei", "Source Han Sans SC", "Noto Sans CJK SC", "Source Han Sans CN", "Noto Sans SC", "Source Han Sans TC", "Noto Sans CJK TC", "WenQuanYi Micro Hei", simsun, sans-serif;
  font-size: 26rpx;
  font-weight: 400;
  line-height: 30rpx;
  color: #aaaaaa;
}
.foot-qui-icon{
  margin-right: 15rpx;
}
.item-box{
  height:70vh;
  width:96% !important;
  margin:0 auto;
  // border: 1px rgba(238, 238, 238, 0.808) solid;
  border-radius: 20rpx;
  position: relative;
  overflow: hidden;
 box-shadow: 1px 1px 8px 1px rgba(0,0,0,0.1);
  transform: translate3d(0, -4px, 0);
  background: var(--qui-BG-2);
}
.item{
  height: calc(100vh - 90rpx) !important;
  display: flex;
  align-items: center;
}
.banner-container {
  width: 100vw;
  height: 100vh;
  .image-container {
	box-sizing: border-box;
	width: 100%;
	// height: 100%;
	display: flex;
	.slide-image {
	  width: 100%;
	  height: 350rpx;
	  z-index: 200;
    border-radius: 20rpx 20rpx 0 0;
	}
  }
  .item-left {
	justify-content: flex-end;
	// padding: 56rpx 26rpx 0 0;
  }
  .item-right {
	justify-content: flex-start;
	// padding: 56rpx 0 0 26rpx;
  }
  .item-center {
	justify-content: center;
	// padding: 15rpx 0 0 0;
  }
  .desc-wrap {
    // box-sizing: border-box;
    // position: absolute;
    line-height: 50rpx;
    // left: 10rpx;
    // // right: 10rpx;
    // left: 0;
    // right: 0;
    // padding: 0 10rpx;
    // padding: 0 6rpx  0 10rpx;
    // top: 390rpx;
    // left: 22rpx;
    // right: 10rpx;
    // padding: 20rpx 10rpx 0 10rpx;
    // padding: 30rpx 40rpx;
    text-align: left;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    // bottom: 80rpx;
    display: -webkit-box;
    -webkit-line-clamp: 10;      
    -webkit-box-orient: vertical;
    padding: 30rpx 40rpx 0 30rpx;

    // position: absolute;
    // border: 1px red solid;
    // box-sizing: border-box;
    // width: 90%;
    // margin: 30rpx auto;
    // line-height: 50rpx; 
    // display: -webkit-box;
    // -webkit-line-clamp: 10;
    // -webkit-box-orient: vertical;
    // padding: 24rpx 66rpx 0;
    .title {
      width: 100%;
      height: 42rpx;
      line-height: 42rpx;
      color: #222222;
      font-size: 30rpx;
      font-weight: 600;
      text-align: left;
    }
    .desc {
      // margin-top: 4rpx;
      width: 100%;

      font-size: 28rpx;
      // text-align: center;
      white-space: break-spaces;
    }
  }
  @keyframes descAnimation {
    0% {
      opacity: 1;
    }
    25% {
      opacity: .5;
    }
    50% {
      opacity: 0;
    }
    75% {
      opacity: .5;
    }
    100% {
      opacity: 1;
    }
  }
  @-webkit-keyframes descAnimation {
    0% {
      opacity: 1;
    }
    25% {
      opacity: .5;
    }
    50% {
      opacity: 0;
    }
    75% {
      opacity: .5;
    }
    100% {
      opacity: 1;
    }
  }
  .hideAndShowDesc {
    animation: descAnimation .3s ease 1;
    -webkit-animation: descAnimation .3s ease 1;
  }
  // .foot .themeItem__footer__themeType1__item{
  //   font-size:14px !important;
  // }

}

</style>
