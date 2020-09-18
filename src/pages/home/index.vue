<template>
  <qui-page ref="quiPage" :data-qui-theme="theme" @pageLoaded="handlePageLoaded" :header="false">
    <view class="content">
      <view class="view-content">
        <qui-page-home
          v-if="showHome"
          ref="home"
          :nav-theme="theme"
          :style="{ display: show_index === 0 ? 'block' : 'none' }"
          @handleClickShare="handleClickShare"
        ></qui-page-home>
				<qui-page-jingxuan
				 ref="jingxuan"
				 :style="{ display: show_index === 1 ? 'block' : 'none' }"
				 @handleClickShare="handleClickShare"
				>
				</qui-page-jingxuan>
        <qui-page-notice
          :nav-theme="theme"
          ref="quinotice"
          :style="{ display: show_index === 3 ? 'block' : 'none' }"
        ></qui-page-notice>
        <qui-page-my
          ref="quimy"
          :style="{ display: show_index === 4 ? 'block' : 'none' }"
        ></qui-page-my>
      </view>
      <view class="tabBar">
        <qui-footer @click="cut_index" :bottom="detectionModel() ? 100 : 0"></qui-footer>
      </view>
    </view>
  </qui-page>
</template>

<script>
import forums from '@/mixin/forums';
import user from '@/mixin/user';
import { mapState, mapMutations } from 'vuex';
import detectionModel from '@/mixin/detectionModel';

export default {
  mixins: [forums, user, detectionModel],
  data() {
    return {
			show:false,
      nowThreadId: 0, // 点击主题ID
      showHome: false,
      tagId: 0, // 标签ID
      currentTab: 'home',
			currentTabIndex:1,
    };
  },
  computed: {
    ...mapState({
      forumError: state => state.forum.error,
      footerIndex: state =>
        state.footerTab.footerIndex ? parseInt(state.footerTab.footerIndex, 10) : 0,
    }),
    loading() {
      return this.forumError.loading;
    },
    show_index: {
      get() {
        const index = this.$store.state.footerTab.footerIndex;
        return index;
      },
      set(index) {
        if (this.forums.set_site) {
          const title = [
            this.forums.set_site.site_name,
						this.i18n.t('home.jingxuan'),
            this.i18n.t('notice.notice'),
            this.i18n.t('profile.mine'),
          ];
          uni.setNavigationBarTitle({
            title: title[index],
          });
        }
      },
    },
  },
  onLoad() {
    // #ifdef MP-WEIXIN
    wx.showShareMenu({
      withShareTicket: true,
      menus: ['shareAppMessage', 'shareTimeline'],
    });
    // #endif
    if (!this.loading && !this.showHome) {
      this.handlePageLoaded();
    }

    uni.$on('notiRead', () => {
      this.getUserInfo(true);
    });
  },
  // 下拉刷新
  onPullDownRefresh() {
		// 首页下拉刷新
    if (this.show_index === 0) {
       if(this.currentTabIndex==1){
      this.$refs.home.threads = [];
      this.$refs.home.pageNum = 1;
			this.$refs.home.isResetList = true;
      // this.$refs.home.loadThreadsSticky();
      this.$refs.home.loadThreads();
			console.log("首页发现下拉刷新")
			}else if(this.currentTabIndex==2){
			     this.$refs.home.followRefresh();
			     console.log("首页关注下拉刷新")
			}else{
			    this.$refs.home.hautiRefresh();
			     console.log("首页话题下拉刷新")
			}
    }
   // 精选页面刷新
   if (this.show_index === 1) {
       console.log("精选页面刷新")
       this.show=false;
       setTimeout(() => {
          this.show=true;
          setTimeout(() => {
         //  this.show=true;
          this.$refs.jingxuan.Reset();
       }, 200);
       }, 200);
     
   
   }
   //信息页面刷新
    if (this.show_index === 3) {
      console.log("信息页面刷新")
     //  this.$refs.quinotice.Refresh();
     this.$refs.quinotice.dialogList = [];
     this.$refs.quinotice.pageNum = 1;
     this.$refs.quinotice.ontrueGetList();
   }
   //  我的页面刷新
    if (this.show_index === 4) {
      console.log("我的页面刷新")
     // this.$refs.quimy.dialogList = [];
      this.$refs.quimy.refreshNum();
   }
    // 停止下拉刷新动画
    uni.stopPullDownRefresh();
  },
  // 监听页面滚动
  onPageScroll(event) {
    this.$refs.home.scroll(event);
  },
  // 上拉加载
  onReachBottom() {
    this.$refs.home.pullDown();
  },
	// // 唤起小程序原声分享
	// onShareAppMessage(res) {
	//   // 来自页面内分享按钮
	//   if (res.from === 'button') {
	//     const threadShare = this.$store.getters['jv/get'](`/threads/${this.threadId}`);
	//     return {
	//       title: threadShare.type === 1 ? this.thread.title : this.thread.firstPost.summaryText,
	//     };
	//   }
	//   return {
	//     title: this.forums.set_site.site_name,
	//   };
	// },
	// // 分享到朋友圈
	// onShareTimeline() {
	//   const threadShare = this.$store.getters['jv/get'](`/threads/${this.nowThreadId}`);
	//   return {
	//     title: threadShare.type === 1 ? this.thread.title : this.thread.firstPost.summaryText,
	//     query: `id=${this.nowThreadId}`,
	//     imageUrl: this.shareLogo,
	//   };
	// },
	
	
  // 唤起小程序原声分享
  onShareAppMessage(res) {
    // 来自页面内分享按钮
    if (res.from === 'button' && res.target.id !== 'top') {
      const threadShare = this.$store.getters['jv/get'](`/threads/${this.nowThreadId}`);
      return {
        title: threadShare.type === 1 ? threadShare.title : threadShare.firstPost.summaryText,
        path: `/pages/topic/index?id=${this.nowThreadId}`,
      };
    }
    return {
      title: this.forums.set_site.site_name,
    };
  },
  // 分享到朋友圈
  onShareTimeline() {
    return {
      title: this.forums.set_site.site_name,
      query: '',
    };
  },
  onShow() {
    // #ifdef MP-WEIXIN
    if (this.$refs.quiPage) {
      this.$store.dispatch('session/setAuth', this.$refs.quiPage.$refs.auth);
      // this.$refs.quiPage.open();
    }
    // #endif
    if (
      !this.$store.getters['session/get']('isLogin') &&
      ['quinotice', 'quimy'].indexOf(this.currentTab) >= 0
    ) {
      uni.navigateTo({
        url: 'pages/home/index',
      });
      return;
    }

    if (this.currentTab === 'quinotice' && this.$refs[this.currentTab]) {
      this.$nextTick(() => {
        this.$refs[this.currentTab].getUnreadNoticeNum();
      });
    }
    if (this.currentTab === 'quimy' && this.$refs[this.currentTab]) {
      this.$nextTick(() => {
        this.$refs[this.currentTab].refreshNum();
      });
    }
    // 其他页面返回
    if (this.forums.set_site) {
      const title = [
        this.forums.set_site.site_name,
				this.i18n.t('home.jingxuan'),
        this.i18n.t('notice.notice'),
        this.i18n.t('profile.mine'),
      ];
      uni.setNavigationBarTitle({
        title: title[this.show_index],
      });
    }
    // #ifdef H5
    // const index = window.location.href.split('?')[1];
    // if (index) {
    //   this.setFooterIndex(parseInt(index, 10));
    // }
    // #endif
  },
  methods: {
		// 监听qui-page-home子组件传来的当前tabbar(发现，关注，话题) index值
		tabIndex(childValue){
		   this.currentTabIndex=childValue;
		   console.log(this.currentTabIndex)
		 },
    ...mapMutations({
      setFooterIndex: 'footerTab/SET_FOOTERINDEX',
    }),
    // 切换组件
    cut_index(e, type, isTabBar) {
      uni.setStorage({
        key: 'page',
        data: '/pages/home/index',
      });
			// 如果是精选页，不跳转至登录页面
			if(e.id==2){
			  this.show=true;    //用v-if来彻底销毁与重建精选页
			  setTimeout(() => {
			     this.$refs.jingxuan.firstEnter()        
			  }, 200);
			 
			}else{
			      this.show=false; 
						}
			// 如果切换到消息，切换时，请求一次数据，这个功能也可以不要
			if(e.id==4){
			  this.$refs.quinotice.dialogList=[]
			  this.$refs.quinotice.ontrueGetList();
			} 
      const tabs = ['home', 'jingxuan','fabu','quinotice', 'quimy'];
      this.currentTab = tabs[type];
      if (
        !this.$store.getters['session/get']('isLogin') &&
        ['quinotice', 'quimy'].indexOf(this.currentTab) >= 0
      ) {
        this.$store.getters['session/get']('auth').open();
        this.currentTab = 'home';
        this.setFooterIndex(0);
        return;
      }

      this.show_index = type;
      // if (isTabBar.indexOf(type) === -1) {
      //   this.$refs[this.currentTab].ontrueGetList();
      //   isTabBar.push(type);
      // }
    },
    // 点击分享事件
    handleClickShare(e) {
      this.nowThreadId = e;
			console.log('this.nowThreadId'+this.nowThreadId)
    },
    handlePageLoaded() {
      this.showHome = true;
    },
  },
  onUnload() {
    uni.$off('notiRead');
  },
};
</script>

<style lang="scss" scoped>
@import '@/styles/base/variable/global.scss';
@import '@/styles/base/theme/fn.scss';
.view-content {
  width: 100%;
  height: calc(100vh - 90rpx);
}
.ioschoice {
  width: 100%;
  height: 100vh;
  padding-top: 240rpx;
}
.ioschoice-img {
  width: 140rpx;
  height: 140rpx;
  margin: 0 auto;
}
.ioschoice-img-icon {
  width: 100%;
  height: 100%;
}
.ioschoice-title {
  margin: 60rpx 0 20rpx;
  font-size: 34rpx;
  font-weight: bold;
  line-height: 45rpx;
  color: rgba(51, 51, 51, 1);
  text-align: center;
}
.ioschoice-content {
  font-size: 28rpx;
  font-weight: 400;
  line-height: 37rpx;
  color: rgba(170, 170, 170, 1);
  text-align: center;
}
.close-btn {
  width: 510rpx;
  height: 90rpx;
  margin: 50rpx auto 0;
  font-size: 28rpx;
  font-weight: 400;
  line-height: 90rpx;
  color: rgba(255, 255, 255, 1);
  text-align: center;
  background: #57cdd4;
  border: 2rpx solid 2px rgba(237, 237, 237, 1);
}
// my页面和notice页面样式渗透不进去的问题
/deep/ .my .cell-item,
/deep/ .notice-box .cell-item {
  padding-right: 40rpx;
}
/deep/ .no-border .cell-item {
  border: 0;
}
/deep/ .my-info__box__detail .cell-item__body {
  height: auto;
  align-items: flex-start;
}
/deep/ .my-tabs .qui-tabs__item--active:after {
  width: 0;
  height: 0;
}
/deep/ .my .qui-tabs__item__title {
  font-weight: normal;
  color: --color(--qui-FC-AAA);
}
/deep/ .my .qui-tabs__item__brief {
  font-weight: bold;
}
/deep/ .my-info__box__detail .cell-item__body__content-title {
  font-weight: bold;
}
</style>
