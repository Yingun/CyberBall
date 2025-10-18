<template>
	<div class="game-page">
		<h1>CyberBall</h1>
		<div class="pre-begin-page">
			<div class="input-name-wrap">
				<label for="input-name">昵称：</label>
				<input
					type="text"
					placeholder="请输入昵称"
					id="input-name"
					v-model="playerName"
				/>
			</div>
			<div>
				<PlayerImg ref="player00" :style="playerStyle" />
			</div>
			<div class="input-color-wrap">
				<div class="row">
					<label for="input-hue">色调：</label>
					<input
						class="color-bar"
						type="range"
						max="360"
						id="input-hue"
						v-model="playerHue"
					/>
				</div>
				<div class="row">
					<label for="input-gray">灰度：</label>
					<input
						class="color-bar"
						type="range"
						id="input-gray"
						v-model="playerGray"
					/>
				</div>
			</div>
			<label class="row">
				请选择玩家数量：
				<input type="number" v-model="playerNumInput" min="3" max="9" />
			</label>

			<div>
				<button @click="onSummit">开始游戏</button>
			</div>
		</div>

		<div id="menu-btn">
			<NavButton destination="/" text="返回" />
		</div>
	</div>
</template>

<script>
import NavButton from "../components/NavButton.vue";
import PlayerImg from "../components/PlayerImg.vue";
export default {
	components: {
		PlayerImg,
		NavButton,
	},
	data() {
		return {
			playerName: "player",
			playerHue: 180,
			playerGray: 0,
			playerNumInput: 3,
			playerNum: 3,
			isStore: false,
		};
	},
	computed: {
		playerStyle() {
			return `filter: sepia(1) saturate(20) hue-rotate(${this.playerHue}deg) grayscale(${this.playerGray}%)`;
		},
	},
	methods: {
		saveConfigs() {
			// 不再持久化到本地/Store，改为通过路由参数传递
			const q = new URLSearchParams({
				mode: "game",
				name: String(this.playerName || ""),
				hue: String(this.playerHue),
				gray: String(this.playerGray),
				num: String(this.playerNum),
			}).toString();
			this.$router.push(`/game?${q}`);
		},
		loadConfigs() {
			// 不从本地/Store读取，使用默认初值
			this.isStore = false;
			this.playerNum = this.playerNumInput;
		},
		checkNameInput() {
			const name = this.playerName;
			if (!name) {
				alert("请输入昵称！");
				return true;
			} else if (name.length > 10) {
				alert("名字太长了！");
				return true;
			}
		},
		checkPlayerNum() {
			const num = this.playerNumInput;
			if (num > 9 || num < 3 || num % 1 !== 0) {
				alert("请输入3-9之间的整数！");
				this.playerNumInput = this.playerNum;
				return false;
			} else {
				this.playerNum = this.playerNumInput;
				return true;
			}
		},
		onSummit() {
			if (!this.checkNameInput() && this.checkPlayerNum()) {
				// 仅保留带参数的跳转，避免覆盖昵称等信息
				this.saveConfigs();
			}
		},
	},
	mounted() {
		// 清空本地持久化数据，避免刷新后沿用旧信息
		try {
			const keys = ["playerName","playerHue","playerGray","playerNum","isSave","cb_topPlayerLimit"];
			keys.forEach(k => localStorage.removeItem(k));
		} catch(e) {}
		this.loadConfigs();
	},
	updated() {
		if (this.playerNum !== this.playerNumInput && this.playerNumInput) {
			this.checkPlayerNum();
		}
	},
};
</script>

<style scoped>
#input-name {
	width: 5rem;
	font-size: smaller;
}

.pre-begin-page {
	display: flex;
	flex-direction: column;
	row-gap: 2rem;
}
</style>
