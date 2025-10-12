<template>
	<div class="game-page">
		<h1>CyberBall</h1>
		<div class="main-page">
			<div class="players-wrap" @mousemove="onPlayerTurn">
				<Player
					v-for="(position, idx) in gridArea"
					:key="idx"
					:ref="setPlayerRef"
					:style="position"
					:playerStyle="playerStyle[idx]"
					:playerName="player.name[idx]"
					:isLeftWard="player.isLeftWard[idx]"
					:title="`分数：${score.playerCounts[idx]}`"
					@moveBall="onThrowBall(idx)"
					@playerLoaded="idx===0?throwBall(0, 0):null"
				/>
			</div>
		</div>

		<Ball :posx="ball.pos[0]" :posy="ball.pos[1]" :moveDur="ball.moveDur" />

		<ScoreBoard :throwNum="score.throwCounts" :score="scoreRank" />
		<!-- 结束模态框 -->
		<div v-if="gameOver" class="end-overlay">
			<div class="end-modal">
				<h2>游戏结束</h2>
				<p>{{ endMsg }}</p>
				<div class="survey-placeholder">
					<!-- 问卷星链接占位：将 surveyUrl 设置为你的问卷地址即可显示按钮 -->
					<a v-if="surveyUrl" :href="surveyUrl" target="_blank" rel="noopener">参与问卷</a>
					<span v-else>问卷链接占位</span>
				</div>
				<NavButton destination="/" text="返回" />
			</div>
		</div>

		<div id="menu-btn">
			<NavButton destination="/" text="返回" />
		</div>
	</div>
</template>

<script>
import Player from "../components/Player.vue";
import Ball from "../components/Ball.vue";
import NavButton from "../components/NavButton.vue";
import ScoreBoard from "../components/ScoreBoard.vue";
import Utils from "../utils/index.js";

const fontSize = 16;
const BALL_DX = 67;
const BALL_DY = 5;
const AI_DELAY_T = 2000;
const cacheVals = {
	playerPos: {},
};

export default {
	components: {
		ScoreBoard,
		NavButton,
		Player,
		Ball,
	},
	data() {
		return {
			score: {
				throwCounts: 0,
				playerCounts: [0, 0, 0, 0, 0, 0, 0, 0, 0],
			},
			// 游戏结束与限制相关
			gameOver: false,
			totalPassLimit: 30,
			topPlayerIdx: null,
			topPlayerLimit: null,
			endMsg: "",
			trainingMode: false,
			// 结束面板问卷链接占位，后续可设置为问卷星链接
			surveyUrl: "https://www.baidu.com",
			player: {
				num: 4,
				numInput: 4,
				turnDur: 0,
				filter: "sepia(1) saturate(20)",
				hue: 180,
				gray: 0,
				name: ["player", "player1", "player2", "player3", "player4", "player5", "player6", "player7", "player8"],
				isLeftWard: [
					false,
					false,
					true,
					false,
					true,
					true,
					false,
					true,
					false,
					true,
				],
				refs: [],
			},
			ball: {
				pos: [-10, -10],
				moveDur: 0,
				owner: undefined,
			},
		};
	},
	computed: {
		scoreRank() {
			const [first, second, third] = Utils.getTopIdx(this.score.playerCounts);
			return [
				{
					player: this.player.name[first],
					value: this.score.playerCounts[first],
				},
				{
					player: this.player.name[second],
					value: this.score.playerCounts[second],
				},
				{
					player: this.player.name[third],
					value: this.score.playerCounts[third],
				},
			];
		},

		// player相关
		playerStyle() {
			const getStyleStr = (hue, gray, turnDur) =>
				`filter: ${this.player.filter} hue-rotate(${hue}deg) grayscale(${gray}%); transition:${turnDur}s linear`;

			const newList = [
				getStyleStr(this.player.hue, this.player.gray, this.player.turnDur),
			];

			for (let i = 1; i < this.player.num; i++) {
				if (Number(this.player.num) === 3 && i === 1) {
					// 小明：蓝色（增强着色：先sepia再高饱和度，再转色相，略调亮度与对比）
					newList.push(`filter: sepia(1) saturate(12) hue-rotate(190deg) brightness(0.95) contrast(1.15) grayscale(0%); transition:0s linear`);
				} else if (Number(this.player.num) === 3 && i === 2) {
					// 小红：粉色（增强着色：先sepia再高饱和度，再转色相，略调亮度与对比）
					newList.push(`filter: sepia(1) saturate(12) hue-rotate(330deg) brightness(1.0) contrast(1.10) grayscale(0%); transition:0s linear`);
				} else {
					newList.push(getStyleStr(Math.random() * 360, Math.random() * 100, 0));
				}
			}

			return newList;
		},
		gridArea() {
			const getGridArea = (i) => {
				if (Utils.playerLayout.modeMap[this.player.num].includes(i)) {
					return `grid-area: ${Utils.playerLayout.rect.mode2[i]}`;
				} else {
					return `grid-area: ${Utils.playerLayout.rect.mode1[i]}`;
				}
			};

			const newList = [];
			for (let i = 0; i < this.player.num; i++) {
				newList.push(getGridArea(i));
			}
			return newList;
		},
	},
	methods: {
		loadConfigs() {
			this.player.name[0] =
				this.$store.getters.playerName || this.player.name[0];
			this.player.hue = this.$store.getters.playerHue || this.player.hue;
			this.player.gray = this.$store.getters.playerGray || this.player.gray;
			this.player.num = Number(this.$store.getters.playerNum) || this.player.num;
		},
		scoreCounter(idx) {
			if (this.gameOver) return;
			// 训练模式：仅计数，不做限制
			if (this.trainingMode) {
				this.score.throwCounts++;
				this.score.playerCounts[idx]++;
				return;
			}
			// 限制模式：正常计数，命中30即结束（getReceptorId已做保底，确保到30时玩家达标）
			this.score.throwCounts++;
			this.score.playerCounts[idx]++;
			const hitTotalLimit = this.score.throwCounts >= this.totalPassLimit;
			if (hitTotalLimit) {
				this.gameOver = true;
				this.ball.moveDur = 0;
				this.endMsg = `总传球数：${this.score.throwCounts}/${this.totalPassLimit}；你的接球数：${this.score.playerCounts[0]}`;
			}
		},
		watchWindowResize() {
			//监听窗口大小的改变，以更新球的位置
			window.addEventListener(
				"resize",
				() => {
					// 暂停游戏（暂停球的移动）
					cacheVals.playerPos = {};       // 清除缓存值（playerPos改变了）
					this.updateBallPosOnResize();   // TODO: 防抖
					// 继续游戏
				},
				false
			);
		},

		// player相关
		onPlayerTurn(e) {
			// 根据鼠标位置的x坐标，让人物控制的玩家转向跟随鼠标
			const x = e.clientX;
			// 未来扩展：让其他player也可以转身，但是这样还要设置他们的this.player.turnDur
			for (const idx of [0]) {
				const turnLeft = this.setPlayerDirection(x, idx);
				if (turnLeft === -1) break;

				// 当转身的player拥有球时，移动球
				if (this.ball.owner == idx) {
					const dx = turnLeft ? -BALL_DX : BALL_DX;
					this.shiftBallPos(dx, this.player.turnDur);
				}
			}
		},
		setPlayerRef(refItem) {
			if (refItem) {
				this.player.refs.push(refItem.$el.childNodes[0]);
			}
		},
		setPlayerDirection(x, idx) {
			if (!this.player.refs[idx]?.offsetLeft) return -1;
			const turnLeft = x < this.player.refs[idx].offsetLeft;
			if (turnLeft == this.player.isLeftWard[idx]) return -1;
			this.player.isLeftWard[idx] = turnLeft;
			return turnLeft;
		},
		getPlayerPos(idx) {
			// 优先读取缓存值
			if (cacheVals.playerPos[idx]) {
				return cacheVals.playerPos[idx];
			}
			// 发生过变化，获取最新的值
			const receptor = this.player.refs[idx];
			if (!receptor) {
				console.log("reference error!");
				return [undefined, undefined];
			}
			let x = receptor.getClientRects()[0]?.left;
			let y = receptor.getClientRects()[0]?.top;
			if (x !== undefined && y !== undefined) {
				cacheVals.playerPos[idx] = [x, y];  // 缓存起来
			}
			return [x, y];
		},

		getReceptorId(ballOwnerIdx) {
			// 根据当前的ball.owner，返回目标的ball.receptor的id。
			// 策略：
			// 1) 玩家(你)达上限后：AI 不再选择玩家(0)
			// 2) 玩家未达上限且剩余可计数传球不足以喂满玩家时：优先把球传给玩家(0)，确保到第30次时一定满足玩家上限
			const n = this.player.num;
			let candidates = [];
			for (let i = 0; i < n; i++) {
				if (i === ballOwnerIdx) continue; // 不能传给自己
				candidates.push(i);
			}
			if (this.trainingMode) {
				const r = Math.floor(Math.random() * candidates.length);
				return candidates[r];
			}
			const playerLimitReached =
				this.topPlayerIdx === 0 &&
				this.topPlayerLimit !== null &&
				this.score.playerCounts[0] >= this.topPlayerLimit;
			if (playerLimitReached) {
				candidates = candidates.filter((i) => i !== 0);
			} else {
				// 未达上限：检查“保底”条件
				const remainingTotal = this.totalPassLimit - this.score.throwCounts; // 含本次之后还剩多少可计数传球
				const remainingToPlayer = this.topPlayerLimit - this.score.playerCounts[0];
				// 若剩余可计数传球数不足以喂满玩家，则本次AI强制传给玩家(0)
				if (remainingTotal <= remainingToPlayer && candidates.includes(0)) {
					return 0;
				}
			}
			// 兜底：若意外为空，回退为除自己外的任意目标
			if (candidates.length === 0) {
				for (let i = 0; i < n; i++) {
					if (i !== ballOwnerIdx) candidates.push(i);
				}
			}
			const r = Math.floor(Math.random() * candidates.length);
			return candidates[r];
		},
		applyAIIdentityForThree() {
			// 当为3人局时，固定AI的名字
			if (Number(this.player.num) === 3) {
				this.player.name[1] = "小红";
				this.player.name[2] = "小明";
			}
		},
		getBallDx(idx) {
			let dx = BALL_DX;
			if (this.player.isLeftWard[idx]) {
				dx = 0;
			}
			return dx;
		},

		// ball相关
		onThrowBall(idx) {
			// 只有当[用户持有球]的时候，按钮才起效果
			if (this.ball.owner == 0) {
				this.throwBall(idx);
			}
		},
		async aiThrowBall() {
			if (this.gameOver) return;
			const receptorId = this.getReceptorId(this.ball.owner);
			await Utils.sleep(AI_DELAY_T);
			if (this.gameOver) return;
			this.throwBall(receptorId);
		},
		throwBall(receptorId = 0, ballMoveDur = 0.5) {
			// 把球从一个玩家扔到另一个玩家手中
			if (this.gameOver) return "game over";
			this.ball.moveDur = ballMoveDur;
			if (receptorId == this.ball.owner) return "don't need to move the ball";
			this.setBallPos(receptorId);
			this.ball.owner = receptorId;
			this.scoreCounter(receptorId);
			if (this.gameOver) return "game over";

			// 如果球到其他玩家手中，执行一个判断...
			if (this.ball.owner > 0) {
				this.aiThrowBall();
			}
		},
		setBallPos(receptorId) {
			// 根据新的player的坐标设置球的位置
			const dx = this.getBallDx(receptorId);
			const [playerX, playerY] = this.getPlayerPos(receptorId);
			if (!playerX || !playerY) return;
			// 设置球相对于player的位置
			const posX = (playerX + dx) / (window.innerWidth / 100);
			const posY = (playerY - BALL_DY) / fontSize;
			this.ball.pos = [posX, posY];
		},
		resetBallPos() {
			// 初始化球的位置：在中间
			this.ball.pos = [
				50 - 32 / 2 / (window.innerWidth / 100),
				(window.innerHeight - 32) / fontSize / 2,
			];
		},
		updateBallPosOnResize() {
			// 根据窗口变化更新球的位置
			this.ball.moveDur = 0;
			if (!this.ball.owner) {
				this.resetBallPos();
			}
			this.setBallPos(this.ball.owner);
		},
		shiftBallPos(dx, moveDur) {
			// 根据dx值平移球的位置
			this.ball.moveDur = moveDur;
			this.ball.pos[0] += dx / (window.innerWidth / 100);
		},
		// 仅限制玩家本人（index=0），AI 不受限制
		initTopPlayerLimit() {
			try {
				this.topPlayerIdx = 0;
				// 按当前时间秒数决定：偶数秒=10，奇数秒=3
				const even = (new Date().getSeconds() % 2 === 0);
				this.topPlayerLimit = even ? 10 : 3;
			} catch (e) {
				console.warn("initTopPlayerLimit failed", e);
				this.topPlayerIdx = 0;
				this.topPlayerLimit = 3;
			}
		},
		restartGame() {
			// 重置分数与状态，并重新生成玩家接球上限
			this.gameOver = false;
			this.endMsg = "";
			this.score.throwCounts = 0;
			for (let i = 0; i < this.score.playerCounts.length; i++) {
				this.score.playerCounts[i] = 0;
			}
			this.ball.moveDur = 0;
			this.ball.owner = undefined;
			this.resetBallPos();
			cacheVals.playerPos = {};
			this.initTopPlayerLimit();
		},
	},
	mounted() {
		this.loadConfigs();
		this.watchWindowResize();
		// 等 DOM/refs 就绪后计算顶端玩家上限
		this.$nextTick(() => {
			this.initTopPlayerLimit();
			this.applyAIIdentityForThree();
			// 训练模式由路由或 store 控制：?mode=training 或 store.getters.trainingMode === true
			this.trainingMode =
				(this.$route?.query?.mode === 'training') ||
				(this.$store?.getters?.trainingMode === true);
		});
	},
	beforeUpdate() {
		this.player.refs = [];
	},
};
</script>

<style>
.players-wrap {
	display: grid;
	grid-template-columns: 1fr 1fr 1fr 1fr;
	grid-template-rows: 1fr 1fr 1fr;
	align-items: center;
}
.end-overlay {
	position: fixed;
	inset: 0;
	background: rgba(0,0,0,0.5);
	display: flex;
	align-items: center;
	justify-content: center;
	z-index: 999;
}
.end-modal {
	background: #fff;
	padding: 20px 24px;
	border-radius: 12px;
	min-width: 260px;
	text-align: center;
	box-shadow: 0 10px 30px rgba(0,0,0,0.2);
}
.survey-placeholder{
	margin-top:12px;
}
.survey-placeholder a{
	display:inline-block;
	padding:8px 14px;
	border-radius:8px;
	background:#1677ff;
	color:#fff;
	text-decoration:none;
}
.survey-placeholder a:hover{
	background:#145fd1;
}
.end-modal button {
	margin-top: 12px;
	padding: 8px 14px;
	border: none;
	border-radius: 8px;
	background: #2b8a3e;
	color: #fff;
	cursor: pointer;
}
.end-modal button:hover {
	background: #237233;
}
</style>
