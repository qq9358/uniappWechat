<template>
	<view class="view-index">
		<view class="view-swiper">
			<swiper class="swiper" :indicator-dots="swiper.indicatorDots" :autoplay="swiper.autoplay" :interval="swiper.interval" :duration="swiper.duration">
				<swiper-item v-for="(imageItem, index) in images" :key="index"><image :src="imageItem" class="image-swiper"></image></swiper-item>
			</swiper>
		</view>
		<view class="view-introduction" @click="onIntroduction">
			<view class="introduction-title">
				<text>{{ introductionTitle }}</text>
			</view>
			<view class="introduction-active">
				<text class="active-text">详情</text>
				<view class="iconfont icon-arrow"></view>
			</view>
		</view>
		<uni-popup ref="introductionPopup" type="bottom">
			<view class="view-popup">
				<view class="popup-header"><text>IntroductionTitle</text></view>
				<view class="popup-body">
					<view class="body-content"><text>IntroductionBody</text></view>
				</view>
			</view>
		</uni-popup>
		<view class="view-categorys">
			<view v-for="category in categorys" :key="category.id" class="view-category">
				<view class="category-header">
					<view class="iconfont icon-menpiao"></view>
					<text>{{ category.name }}</text>
				</view>
				<view class="category-body">
					<view v-for="ticketType in ticketTypes" :key="ticketType.id" class="view-items">
						<view class="item-content">
							<view class="item-name">{{ ticketType.name }}</view>
							<view class="item-status">
								<text>{{ ticketType.travelDateText }}</text>
								<text :style="{ color: getRefundColor(ticketType) }">{{ ' ' + ticketType.refundText + ' ' }}</text>
								<text>无需取票</text>
							</view>
							<view class="item-explain" @click="onShowDescription(ticketType)"><text>新品 | 购买须知></text></view>
						</view>
						<view class="item-active">
							<view class="view-price">
								<text class="text-price">￥{{ ticketType.price }}</text>
							</view>
							<view class="view-book" @click="onBuy(ticketType)"><button>立即预订</button></view>
						</view>
					</view>
				</view>
			</view>
		</view>
		<ticket-type-description
			v-model="showDescription"
			:show-buy="true"
			:ticketTypeId="selectedTicketType.id"
			:ticketTypeName="selectedTicketType.name"
			:price="selectedTicketType.price"
			:shouldRead="selectedTicketType.shouldReadDescription"
		></ticket-type-description>
	</view>
</template>

<script>
import dayjs from 'dayjs';
import { mapState, mapMutations } from 'vuex';
import { mobileMixin } from './../../mixins/mobileMixin.js';
import ticketTypeService from './../../services/ticketTypeService.js';
import scenicService from './../../services/scenicService.js';
import settingService from './../../services/settingService.js';
import memberService from '@/services/memberService.js';
import scenicImage1 from '@/static/image/scenic1.jpg';
import scenicImage2 from '@/static/image/scenic2.jpg';

const today = dayjs();
const tomorrow = dayjs().addDays(1);
export default {
	mixins: [mobileMixin],
	props: {
		publicSaleFlag: {
			type: [Boolean, String],
			default: true
		}
	},
	data() {
		return {
			href: 'https://uniapp.dcloud.io/collocation/pages?id=easycom',
			swiper: {
				indicatorDots: true,
				autoplay: true,
				interval: 2000,
				duration: 500
			},
			introductionTitle: '景区特色  景区简介',
			categorys: [
				{
					id: 1,
					name: '景区门票',
					items: [
						{
							id: 1,
							name: '欢乐票',
							price: 12.34
						},
						{
							id: 2,
							name: '开发票',
							price: 23.45
						}
					]
				},
				{
					id: 2,
					name: '餐饮',
					items: [
						{
							id: 1,
							name: '河粉',
							price: 5.5
						},
						{
							id: 2,
							name: '米饭',
							price: 3
						}
					]
				}
			],
			images: [scenicImage1, scenicImage2],
			scenic: {},
			ticketTypes: [],
			selectedTicketType: {},
			origin: 'http://localhost:8080',
			shareImgUrl: '',
			groundId: 1,
			showDescription: false,
			screenWidth: 0
		};
	},
	computed: {
		swipeHeight() {
			return (this.screenWidth / 640) * 360;
		}
	},
	async onLoad() {
		/* #ifndef MP */
		await memberService.loginFromWeChatAsync({
			code: '',
			state: ''
		});
		/* #endif */
		/* #ifdef MP */
		await memberService.loginFromMiniAsync();
		/* #endif */
		let systemInfo = await uni.getSystemInfo();
		this.screenWidth = systemInfo[1].screenWidth;

		await settingService.getSettingsFromWeChatAsync();

		const getTicketTypeTask = ticketTypeService
			.getTicketTypesForWeiXinSaleAsync({
				publicSaleFlag: this.publicSaleFlag,
				groundId: ''
			})
			.then(ticketTypes => {
				for (const ticketType of ticketTypes) {
					ticketType.travelDateText = this.getTravelDateText(ticketType);
					ticketType.refundText = this.getRefundText(ticketType);
				}

				this.ticketTypes = ticketTypes;
			});

		const getScenicTask = scenicService
			.getScenicAsync()
			.then(scenic => {
				if (scenic.photoList && scenic.photoList.length > 0) {
					this.images = scenic.photoList.map(p => p.url);
					this.shareImgUrl = this.images[0];
				}

				if (scenic.openTime && scenic.closeTime) {
					const today = dayjs().toDateString();
					const openTime = dayjs(`${today} ${scenic.openTime}:00`);
					const closeTime = dayjs(`${today} ${scenic.closeTime}:00`);
					const now = dayjs();
					if (now.isBefore(openTime)) {
						scenic.openText = `未开园 ${scenic.openTime}开园`;
					} else if (now.isBetween(openTime, closeTime)) {
						scenic.openText = `开放中 ${scenic.closeTime}闭园`;
					} else {
						scenic.openText = `已闭园 明日${scenic.openTime}开园`;
					}
				}

				this.scenic = scenic;
			})
			.then(() => {
				let shareUrl = `${this.origin}/tickettype/${this.publicSaleFlag}`;
				if (this.groundId) {
					shareUrl += `?groundId=${this.groundId}`;
				}
				// settingService.configWxJsApi().then(() => {
				// 	settingService.configWxShareUrl(shareUrl, this.shareImgUrl);
				// });
			});

		try {
			await Promise.all([getTicketTypeTask, getScenicTask]);
			this.pageLoaded = true;
		} finally {
		}
	},
	methods: {
		onIntroduction() {
			this.$refs.introductionPopup.open();
		},
		onBook() {
			uni.navigateTo({
				url: '../ticket/buy-ticket'
			});
		},
		onBuy(ticketType) {
			if (ticketType.shouldReadDescription) {
				this.onShowDescription(ticketType);
			} else {
				uni.navigateTo({
					url: '/pages/ticket/buy-ticket?ticketTypeId=' + ticketType.id
				})
			}
		},
		onShowDescription(ticketType) {
			this.selectedTicketType = ticketType;
			this.showDescription = true;
			// this.$refs.itemExplainPopup.open();
		},
		getTravelDateText(ticketType) {
			const startTravelDate = dayjs(ticketType.startTravelDate);
			let travelDateText = '';
			if (startTravelDate.isSameOrBefore(today)) {
				travelDateText = '今日';
			} else if (startTravelDate.isSameOrBefore(tomorrow)) {
				travelDateText = '明日';
			} else {
				travelDateText = startTravelDate.format('MM月DD日');
			}

			return `最早可订${travelDateText}票`;
		},
		getRefundText(ticketType) {
			if (ticketType.allowRefund === false) {
				return '不可退';
			}

			return ticketType.refundLimited ? '有条件退' : '随时退';
		},
		getRefundColor(ticketType) {
			if (ticketType.allowRefund === false) {
				return '#ff2f39';
			}
			return ticketType.refundLimited ? '#ffae13' : '#099fde';
		},
		...mapMutations(['setGroundId'])
	}
};
</script>

<style lang="scss">
.view-index {
	font-size: 14px;

	.view-swiper {
		.swiper {
			height: 232px;

			.swiper-item {
				height: 100%;
			}

			.uni-bg-red {
				background-color: #ff0000;
			}

			.uni-bg-green {
				background-color: #4cd964;
			}

			.uni-bg-blue {
				background-color: #007aff;
			}

			.image-swiper {
				width: 100%;
			}
		}
	}

	.view-introduction {
		display: flex;
		padding: 13px 15px;
		justify-content: space-between;
		font-size: 15px;
		background-color: #fff;
		.introduction-active {
			display: flex;
			justify-content: space-between;
			// align-items: center;
			color: #999;
			.icon-arrow {
				font-size: 20px;
			}
		}
	}
	.view-categorys {
		.view-category {
			margin-top: 10px;
			background: #fff;
			.category-header {
				display: flex;
				padding: 8px 15px;
				border-bottom: 1px solid #dbdbdb;
				font-size: 17px;
				align-items: center;
				.icon-menpiao {
					padding: 1px 3px 0px 3px;
					background-color: #ff7d13;
					border-radius: 25px;
					line-height: 1.5;
					color: #ffffff;
					font-size: 12px;
					height: 19px;
					margin-right: 7px;
				}
			}
			.category-body {
				.view-items {
					padding: 10px 13px 10px 0px;
					margin-left: 15px;
					display: flex;
					justify-content: space-between;
					border-bottom: 1px solid #dbdbdb;
					.item-content {
						.item-name {
							margin-bottom: 5px;
							padding-right: 0;
							line-height: 18px;
							font-size: 14px;
							color: #000;
							text-overflow: ellipsis;
							overflow: hidden;
						}
						.item-status {
							color: #099fde;
							font-size: 11px;
							padding: 0px 0px 7px 0px;
						}
						.item-explain {
							font-size: 12px;
							color: #999;
							height: 21px;
							line-height: 21px;
						}
					}
					.item-active {
						.view-price {
							font-size: 20px;
							font-style: normal;
							color: #f40;
							text-align: right;
							overflow: hidden;
							word-break: break-all;
							font-weight: 400;
							.text-price {
							}
						}
					}
				}
			}
		}
	}
}
</style>
