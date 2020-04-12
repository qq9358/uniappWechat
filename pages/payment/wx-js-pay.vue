<template>
	<view>
		<view class="in-order">
			<view class="in-order-countdown">
				支付剩余时间
				<text>{{ countdownText }}</text>
			</view>
			<view class="in-order-fee">
				<text class="in-order-fee-text">¥</text>
				<text>{{ totalMoney }}</text>
			</view>
			<view class="in-order-title">{{ product }}</view>
		</view>
		<view class="in-pay-method">
			<view class="in-pay-method-pay-type">
				<view class="iconfont icon-weixin"></view>
				<text>微信支付</text>
			</view>
		</view>
		<view class="in-button"><button type="primary" size="large" :loading="saving" @click="pay">确认支付</button></view>
	</view>
</template>

<script>
import dayjs from 'dayjs';
import paymentService from './../../services/paymentService.js';
import commonService from './../../services/commonService.js';
import weiXinJsSdkHelper from './../../utils/weiXinJsSdkHelper.js';

export default {
	data() {
		return {
			countdownText: '',
			totalMoney: 0,
			timer: -1,
			queryTimer: -1,
			saving: false,
			shouldConfirm: true,
			listNo: ''
		};
	},
	computed: {
		product() {
			return uni.getStorageSync('Pay:Product');
		}
	},
	async onLoad(event) {
		try {
			this.listNo = event.listNo;
			const order = await paymentService.getNetPayOrderAsync(this.listNo);
			this.totalMoney = order.payMoney;

			if (order.expireSeconds <= 0) {
				this.shouldConfirm = false;
				this.$router.push({ name: 'tickettype' });
				uni.redirectTo({
					url: '/pages/index/index'
				});
				return;
			}

			const endTime = dayjs('2018-01-01 00:00:00');
			let countdown = endTime.clone().addSeconds(order.expireSeconds);
			const formatStr = order.expireSeconds > 3600 ? 'HH:mm:ss' : 'mm:ss';
			this.countdownText = countdown.format(formatStr);
			this.timer = setInterval(() => {
				countdown = countdown.addSeconds(-1);
				this.countdownText = countdown.format(formatStr);
				if (countdown.isSame(endTime)) {
					paymentService.getNetPayOrderAsync(this.listNo).then(result => {
						this.clear();
						this.shouldConfirm = false;
						if (result.paySuccess) {
							uni.redirectTo({
								url: '/pages/order/order-detail?listNo=' + this.listNo
							});
						} else {
							this.$dialog
								.alert({
									title: '支付超时订单已取消'
								})
								.then(() => {
									uni.redirectTo({
										url: '/pages/index/index'
									});
								});
						}
					});
				}
			}, 1000);
		} catch (err) {
			console.log(err);
		}
	},
	onHide(to, from, next) {
		console.log(to);
		console.log(from);
		console.log(next);
		if (!this.shouldConfirm || to.meta.shouldNotConfirm) {
			this.clear();
			next();
			return;
		}

		this.$dialog
			.confirm({
				title: '你的支付尚未完成，是否取消支付？',
				showCancelButton: true,
				confirmButtonText: '继续支付',
				cancelButtonText: '取消支付'
			})
			.then(() => {
				next(false);
			})
			.catch(() => {
				this.clear();
				this.shouldConfirm = false;
				next({
					name: 'orderdetail',
					params: { listNo: this.listNo },
					replace: true
				});
			});
	},
	methods: {
		async pay() {
			try {
				this.saving = true;

				const payArgs = await paymentService.jsApiPayAsync(this.listNo);
				await weiXinJsSdkHelper.jsApiPay(payArgs);

				this.$toast.loading({
					duration: 0,
					message: '查询支付结果...',
					className: 'van-toast-big'
				});

				this.queryTimer = setInterval(() => {
					paymentService.getNetPayOrderAsync(this.listNo).then(result => {
						if (result.paySuccess) {
							this.clear();
							this.shouldConfirm = false;
							this.$router.replace({
								name: 'orderdetail',
								params: { listNo: this.listNo }
							});
						}
					});
				}, 3000);
			} catch (err) {
				if (err.err_msg == 'get_brand_wcpay_request:cancel') return;

				commonService.logError(err);
			} finally {
				this.saving = false;
			}
		},
		clear() {
			clearInterval(this.timer);
			clearInterval(this.queryTimer);
		}
	}
};
</script>

<style lang="scss" scoped>
.in-order {
	background-color: #fff;
	text-align: center;
	padding: 15px 15px;

	&-countdown {
		font-size: 13px;
		color: #999;
		line-height: 1;
	}

	&-fee {
		margin-top: 10px;
		line-height: 1;
		font-size: 30px;
		letter-spacing: -0.44px;

		&-text {
			font-size: 20px;
			margin-right: 8px;
		}

		text {
			font-weight: 700;
		}
	}

	&-title {
		margin-top: 3px;
		font-size: 11px;
		color: #999;
	}
}

.in-pay-method {
	margin: 15px 0;
	background-color: #fff;

	&-pay-type {
		padding: 0 20px;
		height: 70px;
		line-height: 70px;
		display: flex;
		align-items: center;

		view {
			vertical-align: middle;
			font-size: 25px;
			margin-right: 20px;
			color: rgb(9, 187, 7);
		}

		text {
			display: inline-block;
			vertical-align: middle;
			font-size: 18px;
			font-weight: 600;
		}
	}
}

.in-button {
	padding: 0 15px;

	button {
		border-radius: 5px;
		height: 50px;
		line-height: 39px;
		font-size: 16px;
		background: #19a0f0;
	}
}
</style>
