<template>
  <view class="my">
    <!-- #ifdef MP-WEIXIN -->
    <uni-nav-bar
      :title="i18n.t('profile.mine')"
      fixed="true"
      :color="checked ? '#fff' : '#000'"
      :background-color="checked ? '#2e2f30' : '#fff'"
      status-bar
    ></uni-nav-bar>
    <!-- #endif -->
    <scroll-view
      scroll-y="true"
      scroll-with-animation="true"
      show-scrollbar="false"
      class="scroll-y"
    >
      <view class="my-info">
        <view class="my-info__box">
          <view class="my-info__box__detail">
            <qui-avatar :user="userInfo" :is-real="userInfo.isReal" />
            <qui-cell-item
              :title="userInfo.username || ''"
              :brief="userInfo.groupsName"
              :border="false"
              class="my-info__box__detail-username"
            ></qui-cell-item>
          </view>
        </view>
        <view class="my-info__introduction" v-if="userInfo.signature">
          {{ userInfo.signature }}
        </view>
      </view>
      <view class="my-tabs">
        <qui-tabs :values="items" @clickItem="onClickItem" :brief="true" :current="-1"></qui-tabs>
      </view>
      <view class="my-items">
        <view class="my-items__wrap">
          <navigator url="/pages/my/profile" hover-class="none">
            <qui-cell-item :title="i18n.t('profile.myprofile')" arrow></qui-cell-item>
          </navigator>
          <navigator
            url="/pages/my/wallet"
            hover-class="none"
            v-if="forums.paycenter ? forums.paycenter.wxpay_close : ''"
          >
            <qui-cell-item :title="i18n.t('profile.mywallet')" arrow></qui-cell-item>
          </navigator>
          <navigator url="/pages/my/favorite" hover-class="none">
            <qui-cell-item :title="i18n.t('profile.myfavorite')" arrow></qui-cell-item>
          </navigator>
          <navigator url="/pages/my/shield" hover-class="none">
            <qui-cell-item
              :title="i18n.t('profile.myshield')"
              arrow
              :border="false"
            ></qui-cell-item>
          </navigator>
        </view>
        <view class="my-items__wrap">
          <navigator url="/pages/site/index" hover-class="none">
            <qui-cell-item :title="i18n.t('profile.circleinfo')" arrow></qui-cell-item>
          </navigator>
          <navigator url="/pages/site/search" hover-class="none">
            <qui-cell-item
              :title="i18n.t('profile.search')"
              arrow
              :border="
                forums.other &&
                (forums.other.can_create_invite || forums.other.can_invite_user_scale)
                  ? true
                  : false
              "
            ></qui-cell-item>
          </navigator>
          <navigator
            url="/pages/invite/index"
            hover-class="none"
            v-if="forums.other && forums.other.can_invite_user_scale"
          >
            <qui-cell-item
              :title="i18n.t('profile.inviteFriends')"
              arrow
              :border="forums.other && forums.other.can_create_invite ? true : false"
              :class-item="'invite-friends'"
            ></qui-cell-item>
          </navigator>
          <navigator
            v-if="forums.other && forums.other.can_create_invite"
            url="/pages/manage/index"
            hover-class="none"
          >
            <qui-cell-item
              :title="i18n.t('profile.circlemanagement')"
              arrow
              :border="false"
            ></qui-cell-item>
          </navigator>
        </view>

        <view class="my-items__wrap">
          <qui-cell-item :title="i18n.t('profile.theme')" slot-right :border="false">
            <u-switch @change="changeCheck" v-model="checked" active-color="#57cdd4"></u-switch>
          </qui-cell-item>
        </view>

        <!-- 小程序和微信内：无感模式不展示退出并解绑按钮，其他模式展示退出并解绑按钮
             微信外：展示退出登录按钮 -->
        <view class="logout">
          <!-- #ifdef MP-WEIXIN -->
          <qui-button
            size="large"
            type="warn"
            @click="exitAndUnbind"
            v-if="forums && forums.set_reg && forums.set_reg.register_type !== 2"
          >
            {{ i18n.t('user.noBind') }}
          </qui-button>
          <!-- #endif -->
          <!-- #ifdef H5-->
          <qui-button
            size="large"
            type="warn"
            @click="exitAndUnbind"
            v-if="isWeixin && forums && forums.set_reg && forums.set_reg.register_type !== 2"
          >
            {{ i18n.t('user.noBind') }}
          </qui-button>
          <qui-button size="large" type="warn" @click="logout" v-if="!isWeixin">
            {{ i18n.t('user.logout') }}
          </qui-button>
          <!-- #endif -->
        </view>
      </view>
    </scroll-view>
    <uni-popup ref="popup" type="center">
      <uni-popup-dialog
        type="warn"
        :content="i18n.t('user.loginOutTips')"
        :before-close="true"
        @close="handleClickCancel"
        @confirm="handleClickOk"
      ></uni-popup-dialog>
    </uni-popup>
  </view>
</template>

<script>
import { THEME_DEFAULT, THEME_DARK, SITE_PAY } from '@/common/const';
import forums from '@/mixin/forums';
import user from '@/mixin/user';
import appCommonH from '@/utils/commonHelper';
import uniPopupDialog from '@/components/uni-popup/uni-popup-dialog';
import { mapState, mapMutations } from 'vuex';
// #ifdef H5
import { setCookie } from '@/utils/setCookie';
// #endif

export default {
  components: { uniPopupDialog },
  mixins: [forums, user, appCommonH],
  data() {
    return {
      items: [
        { title: this.i18n.t('profile.topic'), brief: '0' },
        { title: this.i18n.t('profile.following'), brief: '0' },
        { title: this.i18n.t('profile.followers'), brief: '0' },
        { title: this.i18n.t('profile.likes'), brief: '0' },
      ],
      current: 0,
      checked: false,
      // #ifdef H5
      isWeixin: false, // 默认不是微信浏览器
      // #endif
      register_type: 2, // 默认注册模式为无感模式
      site_mode: '', // 站点模式
    };
  },
  computed: {
    ...mapState({
      footerIndex: state =>
        state.footerTab.footerIndex ? parseInt(state.footerTab.footerIndex, 10) : 0,
    }),
    userId() {
      return this.$store.getters['session/get']('userId');
    },
    userInfo() {
      const userInfo = this.$store.getters['jv/get'](`users/${this.userId}`);
      userInfo.groupsName = userInfo.groups ? userInfo.groups[0].name : '';
      this.setNum(userInfo);
      return userInfo;
    },
  },
  created() {
    if (this.forums && this.forums.set_reg) {
      this.site_mode = this.forums.set_site.site_mode;
    }
    // #ifdef H5
    const { isWeixin } = appCommonH.isWeixin();
    this.isWeixin = isWeixin;
    // #endif
  },
  methods: {
    ...mapMutations({
      setFooterIndex: 'footerTab/SET_FOOTERINDEX',
    }),
    changeCheck(e) {
      getApp().globalData.themeChanged(e ? THEME_DARK : THEME_DEFAULT);
    },
    onClickItem(e) {
      uni.navigateTo({
        url: `/pages/profile/index?current=${e.currentIndex}&userId=${this.userId}`,
      });
    },
    exitAndUnbind() {
      this.$refs.popup.open();
    },
    logout() {
      this.$store.dispatch('session/logout').then(() => {
        setCookie('token', '', -1);
        window.location.reload();
      });
    },
    handleClickOk() {
      this.$store.dispatch('jv/delete', `users/${this.userId}/wechat`).then(() => {
        this.handleClickCancel();
        // #ifdef MP-WEIXIN
        this.$store.dispatch('session/logout').then(() => {
          uni.clearStorage();
          if (this.site_mode !== SITE_PAY) {
            this.setFooterIndex(parseInt(0, 10));
          }
          if (this.site_mode === SITE_PAY && this.user && !this.user.isPaid) {
            uni.redirectTo({
              url: '/pages/site/info',
            });
          }
        });
        // #endif
        // #ifdef H5
        this.$store.dispatch('session/logout').then(() => {
          setCookie('token', '', -1);
          window.location.reload();
        });
        // #endif
      });
    },
    handleClickCancel() {
      this.$refs.popup.close();
    },
    // 设置粉丝点赞那些数字
    setNum(res) {
      this.items[0].brief = res.threadCount || 0;
      this.items[1].brief = res.followCount || 0;
      this.items[2].brief = res.fansCount || 0;
      this.items[3].brief = res.likedCount || 0;
    },
    // 组件初始化数据
    ontrueGetList() {
      this.checked = this.theme !== THEME_DEFAULT;
    },
    // 获取最新主题数那些
    refreshNum() {
      const userId = this.$store.getters['session/get']('userId');
      const params = {
        include: 'groups,wechat',
      };
      this.$store.dispatch('jv/get', [`users/${userId}`, { params }]);
    },
  },
};
</script>

<style lang="scss" scoped>
@import '@/styles/base/variable/global.scss';
@import '@/styles/base/theme/fn.scss';
/* #ifdef H5 */
$height: calc(100vh - 120rpx);
/* #endif */

/* #ifdef MP-WEIXIN */
$height: calc(100vh - 260rpx);
/* #endif */
.my-items__wrap {
  padding-left: 40rpx;
  margin-top: 30rpx;
  background: --color(--qui-BG-2);
  border-bottom: 2rpx solid --color(--qui-BOR-ED);
  transition: $switch-theme-time;
}
.my-items {
  padding-bottom: 60rpx;
}
.my-info {
  padding: 40rpx;
  padding-top: 30rpx;
  font-size: $fg-f4;
  background: --color(--qui-BG-2);
  transition: $switch-theme-time;
}
.my-info__box {
  display: flex;
  justify-content: space-between;
}
.my-info__introduction {
  margin-top: 40rpx;
  color: --color(--qui-FC-333);
  word-break: break-all;
  transition: $switch-theme-time;
}
.my-info__box__detail {
  position: relative;
  display: flex;
  flex-direction: row;
  width: 100%;
  font-size: $fg-f4;
  box-sizing: border-box;
}
.my-info__box__detail-username {
  padding-left: 20rpx;
}
.my-tabs {
  background: --color(--qui-BG-2);
  transition: $switch-theme-time;
}
.scroll-y {
  max-height: $height;
}
.logout {
  margin: 30rpx 30rpx 0;
  text-align: center;
  border-radius: 7rpx;
}
</style>
