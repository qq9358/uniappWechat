<template>
  <view class="no-data">
    <view type="flex" justify="center" align="center" :style="{ height: contentHeight + 'px' }">
      <view>
        <text class="iconfont icon-wushuju no-data-icon" />
        <view class="no-data-text">暂无数据</view>
      </view>
    </view>
  </view>
</template>

<script>
export default {
  name: "no-data",
  props: {
    contentHeight: {
      type: Number,
      default: 500
    }
  }
};
</script>

<style lang="scss">
.no-data {
  color: dimgrey;
  text-align: center;

  &-icon {
    font-size: 40px;
  }

  &-text {
    font-size: 13px;
    margin-top: 5px;
  }
}
</style>
