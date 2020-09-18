<template>
  <view class="jingxuan">
    <!-- swiper组件 -->
      <qui-page-banner 
       :banner-list="threads" 
       :swiper-config="swiperConfig" 
        ref="banner"
				@bannerMore='bannerMore'
				@shareClick='shareClick'
				></qui-page-banner>

      <!-- 生成海报组件 -->
      <qui-create-poster
          ref="poster"  
          :subTitle="subTitle" 
          :headerImg="headerImg"
          :username="username"
          ></qui-create-poster>
     <view class="mask" v-if="shareShow" @click="closeShare">
       <view class="wxShareTip">
         <img src="/static/sharePoint.png" alt class="sharePoint" />
         <img src="/static/shareKnow.png" alt class="shareKnow" />
       </view>
     </view>
    <!--分享弹框-->
    <uni-popup ref="sharePopup" type="bottom">
      <view class="popup-share">
        <view class="popup-share-content">
          <button class="popup-share-button" open-type="share" v-if="shareType=='mpwx'"></button>
          <view v-for="(item, index) in bottomData" :key="index" class="popup-share-content-box">
            <view class="popup-share-content-image">
              <view class="popup-share-box" @click="shareContent(index)">
                <qui-icon
                  class="content-image"
                  :name="item.icon"
                  size="46"
                  color="#777777"
                ></qui-icon>
              </view>
            </view>
            <text class="popup-share-content-text">{{ item.text }}</text>
          </view>
        </view>
        <view class="popup-share-content-space"></view>
        <text class="popup-share-btn" @click="cancel('share')">{{ c.cancel }}</text>
      </view>
    </uni-popup>
  </view>
</template>

<script>
import { status } from '@/library/jsonapi-vuex/index';
import { mapState, mapMutations } from 'vuex';
import { DISCUZ_REQUEST_HOST } from '@/common/const';
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
  data() {
    return {
			threadId: '', // 主题id
			isApproved:0,
     shareShow: false, // h5微信分享
		 isWeixin: false,
		 isPhone: false,
		 browser: 0, // 0为小程序，1为除小程序之外的设备
		 contentVal: '', // 这是分享需要传的标题
		 shareLogo: '', // 这是分享需要传的图片
		 desc: '', // 这是分享需要传的描述
    bottomData: [], // 分享方式
    shareType:'',
    subTitle:'',
    headerImg:'',
    username:'',
      nowTime:new Date(),
      interval:30,   //时间间隔，获取当前时间至interval天前，之间的数据，测试值为30，上线时根据需求更改，上一次定为24小时的数据
      createdAtBegin:'2020-05-31 00:00:00',    //发表时间大于
      createdAtEnd:'',  //发表时间小于
      threadType: '3', // 主题类型 0普通 1长文 2视频 3图片（'' 不筛选）
      threadEssence: '', // 筛选精华 '' 不筛选 yes 精华 no 非精华
      threadFollow: 0, // 关注的主题 传当前用户 ID
      pageSize: 20, // 每页20条数据
      pageNum: 1, // 当前页数
      threadsStatusId: 0,
      threads: [],
      isFirst:false,
      swiperConfig: {
        previousMargin: '75rpx',
        nextMargin: '75rpx',
      },
    };
  },
	computed:{
		// topic详情页语言包
		t() {
		  return this.i18n.t('topic');
		},
		// core公共变量语言包
		c() {
		  return this.i18n.t('core');
		},
	},

created(){
    // this.createdAtBegin=this.afterDate(this.interval);
    this.createdAtEnd=this.nowDate();
		// #ifndef MP-WEIXIN
		this.isWeixin = appCommonH.isWeixin().isWeixin; // 这是微信网页
		this.isPhone = appCommonH.isWeixin().isPhone; // 这是h5
		this.browser = 1;
		// #endif
},
  methods: {
		// 分享
		shareClick(list) {
			this.contentVal = list.firstPost.summaryText;
			this.desc = list.firstPost.sumaryText;
			this.shareLogo =
			  list.price == 0 && list.firstPost.images.length > 0
			    ? list.firstPost.images[0].thumbUrl
			    : '';
			this.threadId = list._jv.id;
			this.isApproved = list.isApproved;
			this.headerImg=list.firstPost.images[0].thumbUrl;
			this.subTitle=list.firstPost.contentHtml;
			this.username=list.user.username;
		  if (!this.$store.getters['session/get']('isLogin')) {
		    uni.setStorage({
		      key: 'page',
		      data: getCurUrl(),
		    });
		    // #ifdef MP-WEIXIN
		    this.$store.getters['session/get']('auth').open();
		    // #endif
		    // #ifdef H5
		    if (!this.handleLogin()) {
		      return;
		    }
		    // #endif
		    return;
		  }
		  // #ifdef MP-WEIXIN
			this.$emit('handleClickShare', this.threadId);
			this.mpwxsharePopup()
		  // this.$refs.sharePopup.open();
		  if (this.forums.set_site.site_mode === 'pay') {
		    this.bottomData.forEach((value, key, bottomData) => {
		      // this.bottomData.map((value, key, bottomData) => {
		      value.name === 'wxFriends' && bottomData.splice(key, 1);
		    });
		  }
		  // #endif
		  // #ifdef H5
		  if (this.isWeixin === true) {
		    // 微信内
		    this.shareShow = true;
		  } else {
				this.h5sharePopup();
		    // this.h5Share({
		    //   title: this.contentVal,
		    //   id: this.threadId,
		    //   url: 'pages/topic/index',
		    // });
		  }
		  // #endif
		},
		h5sharePopup(){
			this.bottomData=[
			     {
			        text: this.i18n.t('core.generatePoster'),
			        icon: 'icon-poster',
			        name: 'haibao',
			
			      },
			      {
			        text: this.i18n.t('core.generateLink'),
			        icon: 'icon-link',
			        name: 'lianjie',
			      },
			    ], 
				this.shareType='h5'
			 this.$refs.sharePopup.open();
		},
		mpwxsharePopup(){
			this.bottomData=[
		    {
		      text: this.i18n.t('core.generatePoster'),
		      icon: 'icon-poster',
		      name: 'wx',
		    },
		    {
		      text: this.i18n.t('core.wxShare'),
		      icon: 'icon-wx-friends',
		      name: 'wxFriends',
		    },
		  ], // 分享方式
				this.shareType='mpwx'
			 this.$refs.sharePopup.open();
		},
		// #ifdef H5
		closeShare() {
		  this.shareShow = false;
		},
		// #endif
		// 取消分享
		cancel() {
		  this.$refs.sharePopup.close();
		},
		sharePoster(){
		     this.$refs.poster.row=0;
		     this.$refs.poster.spliceStr(this.subTitle);
		    //  this.$refs.poster.changename(this.username);
		     this.$refs.poster.upDateHeight();
		     setTimeout(() => {
		          this.$refs.poster.showCanvas();
		        //  setTimeout(() => {
		        //   this.$refs.poster.showCanvas();
		        // }, 200);
		     }, 200);
		      
		   },
		// 内容部分分享海报,跳到分享海报页面
		shareContent(index) {
			// #ifdef MP-WEIXIN
			if (index === 0) {
			  if (this.isApproved == 0) {
			    uni.showToast({
			      icon: 'none',
			      title: this.i18n.t('topic.underReview'),
			      duration: 2000,
			    });
			  } else {
			    uni.navigateTo({
			      url: `/pages/share/topic?id=${this.threadId}`,
			    });
			  }
			}
			// #endif
			// #ifdef H5
			
			  if (this.isApproved == 0) {
			    uni.showToast({
			      icon: 'none',
			      title: this.i18n.t('topic.underReview'),
			      duration: 2000,
			    });
			  } else {
					if (index === 0) {
						 this.sharePoster();
					}else {
						this.h5Share({
						  title: this.contentVal,
						  id: this.threadId,
						  url: 'pages/topic/index',
						});
					}
				}
			// #endif
		  this.cancel();
		},
    
    //  当前时间  
    nowDate() {
      var date = this.nowTime;
      var YY = date.getFullYear() + '-';
      var MM = (date.getMonth() + 1 < 10 ? '0' + (date.getMonth() + 1) : date.getMonth() + 1) + '-';
      var DD = (date.getDate() < 10 ? '0' + (date.getDate()) : date.getDate());
      var hh = (date.getHours() < 10 ? '0' + date.getHours() : date.getHours()) + ':';
      var mm = (date.getMinutes() < 10 ? '0' + date.getMinutes() : date.getMinutes()) + ':';
      var ss = (date.getSeconds() < 10 ? '0' + date.getSeconds() : date.getSeconds());
      return YY + MM + DD +" "+hh + mm + ss;
    },

      //多少天之前的时间  
      afterDate(afterday){
        var afterdate = new Date(this.nowTime);  
        afterdate.setDate(this.nowTime.getDate()-afterday);  
        var YY = afterdate.getFullYear() + '-';
        var MM = (afterdate.getMonth() + 1 < 10 ? '0' + (afterdate.getMonth() + 1) : afterdate.getMonth() + 1) + '-';
        var DD = (afterdate.getDate() < 10 ? '0' + (afterdate.getDate()) : afterdate.getDate());
        var hh =(afterdate.getHours() < 10 ? '0' + afterdate.getHours() : afterdate.getHours()) + ':';;
        var mm = (afterdate.getMinutes() < 10 ? '0' + afterdate.getMinutes() : afterdate.getMinutes()) + ':';
        var ss = (afterdate.getSeconds() < 10 ? '0' + afterdate.getSeconds() : afterdate.getSeconds());
        return YY + MM + DD +" "+hh + mm + ss;
      },
    firstEnter(){
      if(this.pageNum>1){
        this.isFirst=false;
        return; 
      }else{
         this.isFirst=true;
         this.loadThreads()
      }
    },
		bannerMore(){
			this.pageNum+=1;
			this.loadThreads();
		},
    Reset(){
        this.isFirst=true;
        this.pageNum=1;
        this.threads=[];
        this.loadThreads();
    },
    
     loadThreads(){
      const params = {
        // 'filter[isSticky]':'no',
        'filter[isApproved]':1,
        'filter[isDeleted]':'no',
        // 'filter[categoryId]':0,
        'filter[type]':this.threadType,
        'filter[isEssence]':'yes',
        'page[number]':this.pageNum,
        'page[limit]':this.pageSize,
        'filter[createdAtBegin]': this.createdAtBegin,   //发表时间大于，
        'filter[createdAtEnd]': this.createdAtEnd,       //发表时间小于
        // '[filter[fromUserId]':0,
        include: [
          'user',
          'firstPost',
          'firstPost.images',
          'firstPost.likedUsers'
        ],
      };
      if (this.threadType !== null) {
        params['filter[type]'] = this.threadType;
      }
      params['filter[fromUserId]'] = this.threadFollow;
      const threadsAction = status.run( () => 
        this.$store.dispatch('jv/get', ['threads', { params }]),
      );
      // this.threadsStatusId = threadsAction._statusID;
      return threadsAction.then((res) => {
        if(res.length > 0){
           // 有数据且有图片为0精选数据
         res.forEach(element => {
          if(element.firstPost.images.length>0){
             this.threads.push(element)
          }
          //  隐藏加载更多 
          this.$refs.banner.flag=false;
        });
				// #ifdef H5
				this.wxShare({
				  title: this.contentVal,
				  desc: this.desc,
				  logo: this.shareLogo,
				});
				// #endif
        }else{
        //  为以后没有数据成为精选时，做准备 
        uni.showModal({
          title: '提示',
          content: '暂无数据,请稍后再试! 是否跳至首页?',
          showCancel:false,
          success: function (res) {
              if (res.confirm) {
                  uni.navigateBack({
                        delta: 1
                 });
              } 
          }
        });
        }
         if(res.length<this.pageSize) {
            this.$refs.banner.type=2;
         }else{
            this.$refs.banner.type=1;
         }
        delete res._jv;
      });
    },
    
  },
};
</script>
<style lang="scss" scoped>
.mask {
  position: fixed;
  top: 0;
  left: 0;
  z-index: 170;
  width: 100%;
  height: 100%;
  background: rgba(#000, 0.6);
}
.wxShareTip {
  position: fixed;
  top: 0;
  left: 0;
  z-index: 2222222222222;
  width: 100%;
  height: 100%;
  text-align: right;
  .sharePoint {
    display: inline-block;
    width: 70%;
    margin-top: 10rpx;
    margin-right: 30rpx;
  }
  .shareKnow {
    display: block;
    width: 35%;
    margin: 20vh auto 30rpx;
  }
}

</style>
