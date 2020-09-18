<template>
  <view class="follow">
      <qui-content
        class="ivideo"
        v-for="(item, index) in threads"
        :ref="'myVideo' + index"
        :key="index"
        :currentindex="index"
        :pay-status="(item.price > 0 && item.paid) || item.price == 0"
        :user-name="item.user.username"
        :is-real="item.user.isReal"
        :theme-image="item.user.avatarUrl"
        :theme-btn="item.canHide || ''"
        :theme-reply-btn="item.canReply || ''"
        :them-pay-btn="item.price > 0"
        :user-groups="item.user && item.user.groups"
        :theme-time="item.createdAt"
        :theme-content="item.type == 1 ? item.title : item.firstPost.contentHtml"
        :thread-type="item.type"
        :media-url="item.threadVideo && item.threadVideo.media_url"
        :is-great="item.firstPost.isLiked"
        :theme-like="item.firstPost.likeCount"
        :theme-comment="item.postCount - 1"
        :tags="[item.category]"
        :images-list="item.firstPost.images"
        :theme-essence="item.isEssence"
        :video-width="item.threadVideo && item.threadVideo.width"
        :video-height="item.threadVideo && item.threadVideo.height"
        :video-id="item.threadVideo && item.threadVideo._jv.id"
        :cover-image="item.threadVideo && item.threadVideo.cover_url"
        :duration="item.threadVideo && item.threadVideo.duration"
        :is-deleted="item.isDeleted"
        :scroll-top="scrollTop"
        :thread-position="
          item.location ? [item.location, item.address, item.longitude, item.latitude] : []
        "
        @click="handleClickShare(item._jv.id)"
        @handleIsGreat="
          handleIsGreat(
            item.firstPost._jv.id,
            item.firstPost.canLike,
            item.firstPost.isLiked,
            item.firstPost.likeCount,
          )
        "
        @commentClick="commentClick(item._jv.id)"
        @contentClick="contentClick(item)"
        @backgroundClick="contentClick(item)"
        @headClick="headClick(item.user._jv.id)"
        @videoPlay="handleVideoPlay"
        @scrollheight="scrpllsip"
      ></qui-content>
      <qui-load-more :status="loadingType" ></qui-load-more>
			<uni-popup ref="popupContent" type="bottom">
			  <view class="popup-share">
			    <view class="popup-share-content">
			      <button class="popup-share-button" open-type="share" plain="true"></button>
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
			          <!-- <image :src="item.icon" class="content-image" mode="widthFix" /> -->
			        </view>
			        <text class="popup-share-content-text">{{ item.text }}</text>
			      </view>
			    </view>
			    <view class="popup-share-content-space"></view>
			    <text class="popup-share-btn" @click="cancel('share')">{{ i18n.t('home.cancel') }}</text>
			  </view>
			</uni-popup>
			<qui-toast ref="toast"></qui-toast> 
  </view>
</template>

<script>
import { status } from '@/library/jsonapi-vuex/index';
import forums from '@/mixin/forums';
import user from '@/mixin/user';
// #ifdef H5
import wxshare from '@/mixin/wxshare-h5';
import appCommonH from '@/utils/commonHelper';
import loginAuth from '@/mixin/loginAuth-h5';
// #endif
import { mapMutations, mapState } from 'vuex';

const sysInfo = uni.getSystemInfoSync();

let navbarHeight;
// #ifdef H5
navbarHeight = sysInfo.statusBarHeight; /* uni-nav-bar的高度 */
// #endif
// #ifdef MP-WEIXIN
navbarHeight = sysInfo.statusBarHeight + 44; /* uni-nav-bar的高度 */
// #endif
const navBarTransform = `translateY(-${navbarHeight}px)`;

export default {
  mixins: [
    forums,
    user,
    // #ifdef  H5
    wxshare,
    appCommonH,
    loginAuth,
    // #endif
  ],

  props: {
    navTheme: {
      type: String,
      default: '',
    },
  },
  data() {
    return {
			followList: [], //用户关注列表中的被关注用户的id，根据id,去获取其最新的文案
			createdAtBegin: '2020-05-31 00:00:00', //发表时间大于
			createdAtEnd: '', //发表时间小于
			followpageSize: 20, // 每页10条数据
			followpageNum: 1, // 当前页数
      navBarTransform,
      // suspended: false, // 是否吸顶状态
      checkoutTheme: false, // 切换主题  搭配是否吸顶使用
      threadType: '', // 主题类型 0普通 1长文 2视频 3图片（'' 不筛选）
      threadEssence: '', // 筛选精华 '' 不筛选 yes 精华 no 非精华
      threadFollow: 0, // 关注的主题 传当前用户 ID
      show: false,
      ifNeedConfirm: true,
      top: 500,
      filterSelected: { label: this.i18n.t('topic.whole'), value: '' }, // 筛选类型
      loadingType: 'more', // 上拉加载状态
      hasMore: false, // 是否有更多
      pageSize: 20, // 每页10条数据
      pageNum: 1, // 当前页数
			
      isLiked: false, // 主题点赞状态
      showSearch: true, // 筛选显示搜索
      navbarHeight, // 顶部导航栏的高度
      headerShow: true, // 是否显示标题图(背景+logo)，不显示标题图时，分类切换栏需要固定顶部
      navTop: 128, // 切换分类导航的top
      navHeight: 0, // 切换分类导航的高度
      nowThreadId: '', // 当前点击主题ID
      filterTop: '', // 筛选弹窗的位置
      shareShow: false, // h5内分享提示信息
      shareTitle: '', // h5内分享复制链接
      isWeixin: '', // 是否是微信浏览器内
      filterList: [
        {
          title: this.i18n.t('home.filterPlate'),
          data: [],
        },
        {
          title: this.i18n.t('home.filterType'),
          data: [
            { label: this.i18n.t('home.all'), value: '', selected: true },
            { label: this.i18n.t('home.text'), value: '0', selected: false },
            { label: this.i18n.t('home.invitation'), value: '1', selected: false },
            { label: this.i18n.t('home.video'), value: '2', selected: false },
            { label: this.i18n.t('home.picture'), value: '3', selected: false },
          ],
        },
        {
          title: this.i18n.t('home.confirmText'),
          data: [
            { label: this.i18n.t('home.all'), value: '', selected: true },
            { label: this.i18n.t('home.essence'), value: '1', selected: false },
            { label: this.i18n.t('home.followed'), value: '2', selected: false },
          ],
        },
      ],
      threads: [],
      location: {},
      sticky: [], // 置顶帖子内容
      shareBtn: 'icon-share1',
      tabIndex: 1, // 选中标签栏的序列,默认显示第一个
      isResetList: false, // 是否重置列表
      bottomData: [
        {
          text: this.i18n.t('home.generatePoster'),
          icon: 'icon-poster',
          name: 'wx',
          id: 1,
        },
        {
          text: this.i18n.t('home.wxShare'),
          icon: 'icon-wx-friends',
          name: 'wx',
          id: 2,
        },
      ],
      threadsStatusId: 0,
      categories: [],
      playIndex: null,
      scrollTop: 0,
      surl: '', // 公安网备案信息地址
      observer: null,
      scrollnumber: '',
      scrollindex: '',
      switchscroll: false,
    };
  },
  computed: {
    ...mapState({
      categoryId: state => state.session.categoryId,
      categoryIndex: state => state.session.categoryIndex,
      footerIndex: state =>
        state.footerTab.footerIndex ? parseInt(state.footerTab.footerIndex, 10) : 0,
    }),
    setIndex: {
      get() {
        const index = this.$store.state.footerTab.footerIndex;
        return index;
      },
      set(index) {
        if (index === 1 || index === 2) {
          this.headerShow = true;
        }
      },
    },
  },
  created() {
    // #ifdef  H5
    this.isWeixin = appCommonH.isWeixin().isWeixin;
    this.isPhone = appCommonH.isWeixin().isPhone;
    if (this.forums.set_site) {
      const recordNumber = this.forums.set_site.site_record_code.replace(/[^\d]/g, '');
      this.surl = `http://www.beian.gov.cn/portal/registerSystemInfo?recordcode=${recordNumber}`;
    }
    // #endif
    // 发布帖子后首页追加最新帖子
    this.$u.event.$on('addThread', thread => this.threads.unshift(thread));
    // 详情页删除帖子后在首页删除
    this.$u.event.$on('deleteThread', thread =>
      this.threads.forEach((item, index) => {
        if (item._jv.id === thread) {
          this.threads.splice(index, 1);
        }
      }),
    );
    // 编辑删除图片后首页删除图片
    this.$u.event.$on('deletedImg', res => {
      this.threads.forEach(item => {
        if (item._jv.id === res.threadId) {
          item.firstPost.images.splice(res.index, 1);
        }
      });
    });
    // 置顶列表添加数据当详情页置顶时
    this.$u.event.$on('stickyThread', data => {
      this.sticky.unshift(data);
      this.threads.forEach((item, index) => {
        if (item._jv.id === data._jv.id) {
          this.threads.splice(index, 1);
        }
      });
    });
    // 详情页取消置顶时置顶列表中删除该置顶
    this.$u.event.$on('cancelSticky', data => {
      this.sticky.forEach((item, index) => {
        if (item._jv.id === data._jv.id) {
          this.sticky.splice(index, 1);
        }
      });
    });

    // 详情页编辑增加图片时首页增加图片
    this.$u.event.$on('refreshImg', res => {
      // eslint-disable-next-line no-restricted-syntax
      for (const index in this.threads) {
        if (this.threads[index]._jv.id === res.threadId) {
          const images = this.$store.getters['jv/get'](`posts/${res.id}`);
          this.threads[index].firstPost.images = images.attachments;
          this.$forceUpdate();
          break;
        }
      }
    });

    // 详情页编辑定位
    this.$u.event.$on('updateLocation', (id, res) => {
      this.threads.forEach((item, index) => {
        if (item._jv.id === id) {
          this.threads[index].latitude = res.latitude || '';
          this.threads[index].location = res.location || '';
          this.threads[index].longitude = res.longitude || '';
          this.threads[index].address = res.address || '';
        }
      });
    });
    // h5微信分享
    // #ifdef H5
    this.wxShare({
      title: this.forums.set_site ? this.forums.set_site.site_name : '',
      desc: this.forums.set_site ? this.forums.set_site.site_introduction : '',
      logo:
        this.forums.set_site && this.forums.set_site.site_logo
          ? this.forums.set_site.site_logo
          : '',
    });
    // #endif
    this.ontrueGetList();
    uni.$on('logind', () => {
      this.ontrueGetList();
    });
  },
  destroyed() {
    uni.$off('logind');
    // #ifdef H5
    uni.$off('updateIndex');
    uni.$off('updateNoticePage');
    uni.$off('updateMy');
    // #endif
  },
  mounted() {
    this.$u.event.$on('tagClick', tagId => {
      this.isResetList = true;
      this.loadCategories();
      this.setCategoryId(tagId);
      this.setCategoryIndex(this.getCategorieIndex(tagId));
      // 首页主题置顶列表
      this.loadThreadsSticky();
      // 首页主题内容列表
      this.getFollows();
    });

    if (this.footerIndex === 0) {
      // console.log('fffffffffffff');
      this.$uGetRect('#navId').then(rect => {
        this.navTop = rect.top;
        // console.log(this.navTop, 'this.navTopthis.navTopthis.navTopthis.navTop');
        this.navHeight = rect.height;
      });
    }

    if (this.forums.set_site) {
      uni.setNavigationBarTitle({
        title: this.forums.set_site.site_name,
      });
    }

    // #ifdef H5
    uni.$on('updateIndex', () => {
      this.headerShow = true;
    });
    uni.$on('updateNoticePage', () => {
      // console.log('99999');
      this.headerShow = true;
    });
    uni.$on('updateMy', () => {
      // console.log('我的我的');
      this.headerShow = true;
    });

    // #endif
    // uni.$on('onpullDownRefresh', () => {
    //   this.navBarTransform = 'none';
    // })
  },
  methods: {
		// 刷新关注的数据
		followRefresh(){
		   this.$refs.follow.Refresh();
		},
		// 刷新话题的数据
		hautiRefresh(){
		   this.$refs.huati.Refresh();
		},
		gotosearch() {
		  const url = `/pages/site/search`;
		  uni.navigateTo({
		    url,
		  });
		},
    ...mapMutations({
      setCategoryId: 'session/SET_CATEGORYID',
      setCategoryIndex: 'session/SET_CATEGORYINDEX',
    }),
    topMargin() {
      return ';';
    },
    scrpllsip(e, index) {
      // console.log(e, index);
      this.scrollnumber = e;
      this.scrollindex = index;
      this.switchscroll = true;
    },
    scroll(event) {
      // if (this.footerIndex === 0) {
      this.scrollTop = event.scrollTop;
      if (Math.abs(this.scrollnumber) && this.switchscroll) {
        this.num = Math.abs(this.scrollTop) - Math.abs(this.scrollnumber);
        if (this.num >= 10 || this.num <= -10) {
          // console.log('视频暂停播放');
          this.$refs[`myVideo${this.scrollindex}`][0].pauseVideo();
          this.switchscroll = false;
        }
      }
      // #ifdef MP-WEIXIN
      if (!this.navbarHeight) {
        return;
      }

      if (event.scrollTop + this.navbarHeight + 20 >= this.navTop) {
        this.headerShow = false;
        this.navBarTransform = 'none';
      } else {
        this.headerShow = true;
        this.navBarTransform = `translate3d(0, -${this.navbarHeight}px, 0)`;
      }
      // #endif

      // #ifdef H5
      if (event.scrollTop >= this.navTop) {
        this.headerShow = false;
        // console.log('falsefalsefalse');
        this.navBarTransform = 'none';
      } else {
        this.headerShow = true;
        // console.log('truetruetruetrue');
        this.navBarTransform = `translate3d(0, -${this.navbarHeight}px, 0)`;
      }
      // #endif
      // }
    },
    // 滑动到顶部
    toUpper() {
      this.headerShow = true;
    },
    // 初始化选中的选项卡
    getCategorieIndex(tagId) {
      for (let i = 0, len = this.categories.length; i < len; i += 1) {
        if (+this.categories[i]._jv.id === +tagId) {
          return i;
        }
      }
    },
	
    
    // 筛选分类里的搜索
    searchClick() {
      uni.navigateTo({
        url: '/pages/site/search',
      });
      this.show = false;
    },
    // 点击置顶跳转到详情页
    stickyClick(id) {
      uni.navigateTo({
        url: `/pages/topic/index?id=${id}`,
      });
    },
    // 内容部分点击评论跳到详情页
    commentClick(id) {
      uni.navigateTo({
        url: `/pages/topic/index?id=${id}`,
      });
    },
    // 内容部分点击跳转到详情页
    contentClick(thread) {
      if (thread.canViewPosts) {
        uni.navigateTo({
          url: `/pages/topic/index?id=${thread._jv.id}`,
        });
      } else {
        this.$refs.toast.show({ message: this.i18n.t('home.noPostingTopic') });
      }
    },
    // 点击头像调转到个人主页
    headClick(id) {
      uni.navigateTo({
        url: `/pages/profile/index?userId=${id}`,
      });
    },
    
    // #ifdef H5
    closeShare() {
      this.shareShow = false;
    },
    // #endif
    
    // 取消按钮
    cancel() {
      this.$refs.popupContent.close();
    },
    // 点赞调取用户信息取消弹框
    close() {
      this.$refs.auth.close();
    },
    // 首页内容部分分享按钮弹窗
    handleClickShare(id) {
     this.$emit('handleClickShare',id)
		 
    },
   
	 
    // 首页导航栏分类列表数据
    loadCategories() {
      this.$store.dispatch('jv/get', ['categories', {}]).then(data => {
        const resData = [...data] || [];
        this.categories = [
          // {
          //   _jv: {
          //     id: 0,
          //   },
          //   name: this.i18n.t('home.all'),
          // },
          ...resData,
        ];
        const categoryFilterList = [
          {
            label: this.i18n.t('home.all'),
            value: 0,
            selected: true,
          },
        ];
        resData.forEach(item => {
          categoryFilterList.push({
            label: item.name,
            value: item._jv.id,
            selected: false,
          });
        });

        this.filterList[0].data = categoryFilterList;
      });
    },
    // 首页置顶列表数据
    loadThreadsSticky() {
      const params = {
        'filter[isSticky]': 'yes',
        'filter[isApproved]': 1,
        'filter[isDeleted]': 'no',
        'filter[categoryId]': this.categoryId,
        include: ['firstPost'],
      };
      this.$store.dispatch('jv/get', ['threads', { params }]).then(data => {
        this.sticky = [...data];
      });
    },
		//查询用户的关注列表
		getFollows() {
			this.threads = [];
			this.followList = [];
			const info = {
				'filter[type]': 1,
				include: [
					'toUser',
				],
			};
			const threadsAction = status.run(() =>
				this.$store.dispatch('jv/get', ['follow', {
					info
				}]),
			);
			this.threadsStatusId = threadsAction._statusID;
			return threadsAction.then((res) => {
				if (res.length <= 0) {
					this.loadingType = 'nomore'
				}
				res.forEach(element => {
					//  获取到此用户的关注列表
					this.followList.push(element.to_user_id)
				});
		
		
				this.loadThreads();
			});
		},
		
		// 首页内容部分数据请求
		loadThreads() {
			this.followList.forEach(element => {
				const params = {
					'filter[userId]': element,
					'filter[isSticky]': 'no',
					'filter[isApproved]': 1,
					'filter[isDeleted]': 'no',
					'filter[type]': 3,
					'filter[createdAtBegin]': this.createdAtBegin, //发表时间大于，
					'filter[createdAtEnd]': this.createdAtEnd, //发表时间小于
					'filter[isEssence]': this.threadEssence,
					'filter[pageSize]': this.pageSize,
					'filter[pageNum]': this.pageNum,
					include: [
						'user',
						'user.groups',
						'firstPost',
						'firstPost.images',
					],
				};
				const threadsAction = status.run(() =>
					this.$store.dispatch('jv/get', ['threads', {
						params
					}]),
				);
				this.threadsStatusId = threadsAction._statusID;
				return threadsAction.then((res) => {
					// this.loadingType = res.length === this.pageSize ? 'more' : 'nomore';
					delete res._jv;
					if (this.isResetList) {
						this.fictitiousList = res;
					} else {
						this.fictitiousList = [...this.fictitiousList, ...res];
					}
		
					// 对数据进行处理，按照时间进行倒叙排列
					this.fictitiousList.sort((a, b) => {
						return Math.round(new Date(a.updatedAt) / 1000) < Math.round(new Date(b.updatedAt) / 1000) ? 1 : -1
					});
		
		
					setTimeout(() => {
		
						this.threads = this.fictitiousList;
		
						//   this.fictitiousList.forEach((element,index) => {
						//      if(index <= 5){
						//        console.log(index)
						//        this.threads.push(element);
						//        this.fictitiousList.splice(index,1)
						//      }else {
						//        return;
						//      }
						//    });
		
						// // this.fictitiousList[i].splice(i,1)
						this.loadingType = 'nomore';
					}, 500);
		
					this.isResetList = false;
				});
		
			});
		
		},
		//下拉刷新
		Refresh() {
			this.getFollows();
		},
		// 上拉加载
		pullDowns() {
			console.log("1111")
			// this.loadThreads();
		},
		// 组件初始化请求接口
		ontrueGetList() {
			this.isResetList = true;
			// 首页主题内容列表
			this.getFollows();
		},
		
		
    // 首页内容部分数据请求
    // loadThreads() {
    //   const params = {
    //     'filter[isSticky]': 'no',
    //     'filter[isApproved]': 1,
    //     'filter[isDeleted]': 'no',
    //     'filter[categoryId]': 1,  //因为项目没有分类机制，所以去掉分类字段(默认为1)
    //     'filter[type]': this.threadType,
    //     'filter[isEssence]': this.threadEssence,
    //     'page[number]': this.pageNum,
    //     'page[limit]': this.pageSize,
    //     include: [
    //       'user',
    //       'user.groups',
    //       'firstPost',
    //       'firstPost.images',
    //       'category',
    //       'threadVideo',
    //     ],
    //   };
    //   if (this.threadType !== null) {
    //     params['filter[type]'] = this.threadType;
    //   }
    //   params['filter[fromUserId]'] = this.threadFollow;

    //   const threadsAction = status.run(() =>
    //     this.$store.dispatch('jv/get', ['threads', { params }]),
    //   );

    //   this.threadsStatusId = threadsAction._statusID;

    //   return threadsAction.then(res => {
    //     this.loadingType = res.length === this.pageSize ? 'more' : 'nomore';
    //     delete res._jv;
    //     if (this.isResetList) {
    //       this.threads = res;
    //     } else {
    //       this.threads = [...this.threads, ...res];
    //     }
    //     this.isResetList = false;
    //   });
    // },
		
		
		
    // 内容部分点赞按钮点击事件
    handleIsGreat(id, canLike, isLiked) {
      if (!this.$store.getters['session/get']('isLogin')) {
        uni.setStorage({
          key: 'page',
          data: '/pages/home/index',
        });
        // #ifdef MP-WEIXIN
        this.$store.getters['session/get']('auth').open();
        // #endif
        // #ifdef H5
        if (!this.handleLogin()) {
          return;
        }
        // #endif
      }
      const params = {
        _jv: {
          type: 'posts',
          id,
        },
        isLiked: isLiked !== true,
      };
      this.$store.dispatch('jv/patch', params);
    },
    // 视频禁止同时播放
    handleVideoPlay(index, e) {
      this.switchscroll = e;
      if (this.playIndex !== index && this.playIndex !== null) {
        this.$refs[`myVideo${this.playIndex}`][0].pauseVideo();
      }
      this.playIndex = index;
    },
  },
};
</script>
<style lang="scss">
@import '@/styles/base/variable/global.scss';
@import '@/styles/base/theme/fn.scss';
/* #ifdef H5 */
$padding-bottom: 180rpx;
/* #endif */
/* #ifdef MP-WEIXIN */
$padding-bottom: 160rpx;
/* #endif */
.home {
  // min-height: 100vh;
  color: --color(--qui-FC-333);
  background-color: --color(--qui-BG-1);
}
.status_bar {
    height: var(--status-bar-height);
    width: 100%;
    background-color: --color(--qui-BG-2);
}
.top_view {
    height: var(--status-bar-height);
    width: 100%;
    position: fixed;
    background-color: --color(--qui-BG-2);
    top: 0;
    z-index: 999;
}
.status-bar {
  position: fixed;
  z-index: 2;
  transition: 0.2s;
}
.nav {
  position: fixed;
  top: var(--status-bar-height);
  text-align: center;
  z-index: 1;
  width: 100%;
  overflow: hidden;
  background: --color(--qui-BG-2);
  /* #ifdef MP-WEIXIN */
  border-bottom: 2rpx solid --color(--qui-BOR-ED);
  /* #endif */
  transition: box-shadow 0.2s, -webkit-transform 0.2s;

  &__box {
    position: absolute;
    right: 0;
    z-index: 2;
    display: block;
    float: right;
    width: 80rpx;
    height: 97rpx;
    background: --color(--qui-BG-2);
    &__icon {
      margin-left: 24rpx;
      line-height: 100rpx;
    }
  }
}

.scrolled .nav {
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}
.filter-modal .filter-modal__content {
  max-height: 500rpx;
}

.sticky {
  // margin: 20rpx auto;
  border-top: 2rpx solid --color(--qui-BOR-ED);
  border-bottom: 2rpx solid --color(--qui-BOR-ED);
}
.sticky__box {
  background: --color(--qui-BG-2);
}

.sticky__isSticky {
  display: flex;
  width: 710rpx;
  height: 80rpx;
  margin-left: 30rpx;
  font-size: $fg-f3;
  line-height: 80rpx;
  background: --color(--qui-BG-2);
  border-bottom: 2rpx solid --color(--qui-BOR-ED);
  // border-radius: 6rpx;
  // box-shadow: 0rpx 2rpx 4rpx rgba(0, 0, 0, 0.05);
  &__box {
    // display: block;
    width: 62rpx;
    height: 35rpx;
    margin-top: 27rpx;
    // margin-left: 20rpx;
    font-size: $fg-f1;
    line-height: 35rpx;
    color: --color(--qui-FC-777);
    text-align: center;
    background: --color(--qui-BG-777);
    border-radius: 6rpx;
    transition: $switch-theme-time;
  }
  &__count {
    width: 572rpx;
    height: 100%;
    margin-top: 27rpx;
    margin-left: 21rpx;
    overflow: hidden;
    line-height: 35rpx;
    color: --color(--qui-FC-333);
    text-overflow: ellipsis;
    white-space: nowrap;
    &__text {
      display: flex;
      flex-direction: row;
      align-items: center;
    }
  }
}
.sticky__isSticky:last-child {
  border-bottom: none;
}
.horizonal-tab .active {
  color: --color(--qui-BG-HIGH-LIGHT);
}
.scroll-tab {
  z-index: 100;
  height: 100rpx;
  // text-align: center;
  white-space: nowrap;
  border-bottom: 2rpx solid --color(--qui-BOR-EEE);
}
.scroll-tab-item {
  z-index: 1;
  display: inline-block;
  margin: 20rpx 30rpx;
  font-size: $fg-f3;
  line-height: 77rpx;
  color: --color(--qui-FC-777);
}
.active .scroll-tab-line {
  color: --color(--qui-BG-HIGH-LIGHT);
  border-bottom: 4rpx solid --color(--qui-BG-HIGH-LIGHT);
}
.uni-tab-bar .active {
  font-size: $fg-f4;
  font-weight: bold;
  color: --color(--qui-BG-HIGH-LIGHT);
}
.main {
  padding-bottom: $padding-bottom;
  background: --color(--qui-BG-1);
	margin-top: 100rpx;
}

// .scroll-y {
//   // max-height: calc(100vh - 497rpx);
//   // max-height: calc(100vh - 100rpx);
//   height: calc(100vh - 90rpx);
// }

.nav .filter-modal {
  position: absolute;
  z-index: 1000;
  width: 100%;
}
.sticky__isSticky__text {
  display: inline-block;
  width: 100%;
  height: 35rpx;
  overflow: hidden;
  line-height: 35rpx;
  text-overflow: ellipsis;
  word-break: break-all;
  white-space: nowrap;
}
.record {
  display: flex;
  width: 100%;
  height: 40rpx;
  margin-top: -$padding-bottom;
  font-size: $fg-f3;
  color: --color(--qui-FC-B2);
  text-align: center;
  justify-content: center;
  &__box {
    &-url {
      font-size: $fg-f3;
      color: --color(--qui-BG-HIGH-LIGHT);
    }
  }
  &__box1 {
    margin-left: 20rpx;
    line-height: 40rpx;
    &-url {
      font-size: $fg-f3;
      color: --color(--qui-BG-HIGH-LIGHT);
    }
  }
  &__box2 {
    line-height: 40rpx;
    &-url {
      font-size: $fg-f3;
      color: --color(--qui-BG-HIGH-LIGHT);
    }
  }
  a {
    color: --color(--qui-FC-B2);
    text-decoration: none;
  }
}
.copyright {
  width: 100%;
  height: 40rpx;
  font-size: $fg-f3;
  color: --color(--qui-FC-B2);
  text-align: center;
}
.wxcopyright {
  margin-top: -$padding-bottom;
  font-size: $fg-f3;
  color: --color(--qui-FC-B2);
  text-align: center;
}
.copyright_margin {
  margin-top: -$padding-bottom;
}
.search-box__content {
  padding: 10px 30px 11px 20px;
  text-align: center;
  background-color: var(--qui-BG-2);
}
.search {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  background-color: #f6f6f6;
  padding-left: 10px;
}
.search-input {
  width: 100%;
  height: 40px;
  line-height: 40px;
  text-align: left;
  margin-left: 5px;
}
.retureTop{
  display: flex;
  justify-content: center;
  align-items: center;
  background: #57cdd4;
  border-radius: 50%;
  position: fixed;
  right:30rpx;
  bottom:130rpx;
  width:80rpx;
  height:80rpx;
  
}
</style>
