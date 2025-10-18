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
					@playerLoaded="onPlayerLoaded(idx)"
				/>
			</div>
		</div>

		<Ball :posx="ball.pos[0]" :posy="ball.pos[1]" :moveDur="ball.moveDur" />

		<ScoreBoard :throwNum="score.throwCounts" :score="scoreRank" />
		<!-- 开始前提示 -->
		<div v-if="showIntro" class="end-overlay">
			<div class="end-modal">
				<h2>玩法说明</h2>
				<div class="intro-text">
					<p v-if="!trainingMode">现在进入到正式的实验环节，在屏幕上你会看到三个人物进行传球游戏 </p>
					<p v-else>现在进入练习环节，在屏幕上你会看到三个人物进行传球游戏 </p>
					<ul>
						<li>屏幕底部的人物是你自己。</li>
						<li>另外两个人物是在其它房间进行实验的另外两个人</li>
						<li>你们之间会互相进行传球，每一次你可以选择将球传给其中一人 </li>
					</ul>
					<p>准备好了请点击“进入游戏” </p>
				</div>
				<button @click="startGame">进入游戏</button>
			</div>
		</div>
		<!-- 加载提示 -->
		<div v-if="showLoading && !trainingMode" class="end-overlay">
			<div class="end-modal">
				<h2 v-if="!showAlmostStart">正在等待其他玩家准备中……</h2>
				<h2 v-else>匹配就绪，游戏房间资源加载中</h2>
				<div class="loading-wrap">
					<div v-if="!showAlmostStart" class="loading-spinner"></div>
					<div v-else class="ready-indicator sk-fading-circle" aria-label="玩家已就绪">
						<span class="sk-circle1 sk-circle"></span>
						<span class="sk-circle2 sk-circle"></span>
						<span class="sk-circle3 sk-circle"></span>
						<span class="sk-circle4 sk-circle"></span>
						<span class="sk-circle5 sk-circle"></span>
						<span class="sk-circle6 sk-circle"></span>
						<span class="sk-circle7 sk-circle"></span>
						<span class="sk-circle8 sk-circle"></span>
						<span class="sk-circle9 sk-circle"></span>
						<span class="sk-circle10 sk-circle"></span>
						<span class="sk-circle11 sk-circle"></span>
						<span class="sk-circle12 sk-circle"></span>
					</div>
					<div class="loading-text">
						<span v-if="showAlmostStart">玩家已就绪，正在进入…</span>
						<span v-else>请稍候，游戏即将开始（约剩余 {{ loadingRemaining }} 秒）</span>
					</div>
				</div>
			</div>
		</div>
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
const AI_DELAY_MIN = 1000; // 1s
const AI_DELAY_MAX = 5000; // 5s
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
			topPlayerSchedule: [],
			lastPlayer0Throw: null,
			endMsg: "",
			trainingMode: false,
			showIntro: true,
			showLoading: false,
			waitSeconds: 6,
			// 结束面板问卷链接占位，后续可设置为问卷星链接
			surveyUrl: "",
			player: {
				num: 3,
				numInput: 3,
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
			// 等待页倒计时
			loadingRemaining: 0,
			loadingElapsed: 0,
			loadingTimer: null,
			showAlmostStart: false,
			effectiveWait: 0,
			// 三人局AI名字预设开关（最后5秒启用）
			aiPresetNamesEnabled: false,
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
			// 优先使用路由参数；无则回退到当前默认或 Store（仅内存）
			const q = this.$route?.query || {};
			const qName = q.name ? String(q.name) : null;
			const qHue = q.hue !== undefined ? Number(q.hue) : null;
			const qGray = q.gray !== undefined ? Number(q.gray) : null;
			const qNum = q.num !== undefined ? Number(q.num) : null;

			this.player.name[0] = qName && qName.length > 0 ? qName : (this.$store?.getters?.playerName || this.player.name[0]);
			if (!Number.isNaN(qHue) && qHue !== null) this.player.hue = qHue;
			else this.player.hue = this.$store?.getters?.playerHue || this.player.hue;

			if (!Number.isNaN(qGray) && qGray !== null) this.player.gray = qGray;
			else this.player.gray = this.$store?.getters?.playerGray || this.player.gray;

			// 玩家数量：合法范围 3-9
			let sNum = Number(this.$store?.getters?.playerNum);
			let setNum = (!Number.isNaN(qNum) && qNum >= 3 && qNum <= 9) ? qNum :
				(!Number.isNaN(sNum) && sNum >= 3 && sNum <= 9 ? sNum : this.player.num);
			this.player.num = setNum;
			this.player.numInput = setNum;
		},
		scoreCounter(idx) {
			if (this.gameOver) return;
			// 训练模式：仅计数，不做限制
			if (this.trainingMode) {
				this.score.throwCounts++;
				this.score.playerCounts[idx]++;
				if (idx === 0) {
					// 若命中计划中的接球时机，则消耗一项
					if (Array.isArray(this.topPlayerSchedule) && this.topPlayerSchedule.length > 0 && this.topPlayerSchedule[0] === this.score.throwCounts) {
						this.topPlayerSchedule.shift();
					}
					// 记录玩家0上次接球发生在第几次传球
					this.lastPlayer0Throw = this.score.throwCounts;
				}
				return;
			}
			// 限制模式：正常计数，命中30即结束（getReceptorId已做保底，确保到30时玩家达标）
			this.score.throwCounts++;
			this.score.playerCounts[idx]++;
			if (idx === 0) {
				// 若命中计划中的接球时机，则消耗一项
				if (Array.isArray(this.topPlayerSchedule) && this.topPlayerSchedule.length > 0 && this.topPlayerSchedule[0] === this.score.throwCounts) {
					this.topPlayerSchedule.shift();
				}
				// 记录玩家0上次接球发生在第几次传球
				this.lastPlayer0Throw = this.score.throwCounts;
			}
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
			// 计划驱动的均匀分配：在指定轮次把球传给玩家(0)，非计划轮次不传给玩家(0)
			const n = this.player.num;
			let candidates = [];
			for (let i = 0; i < n; i++) {
				if (i === ballOwnerIdx) continue; // 不能传给自己
				candidates.push(i);
			}
			// 训练模式：完全随机
			if (this.trainingMode) {
				const r = Math.floor(Math.random() * candidates.length);
				return candidates[r];
			}
			// 限制模式：按计划控制玩家0的接球时机
			const playerLimitReached =
				this.topPlayerIdx === 0 &&
				this.topPlayerLimit !== null &&
				this.score.playerCounts[0] >= this.topPlayerLimit;
			const nextThrow = this.score.throwCounts + 1;
			const hasSchedule = Array.isArray(this.topPlayerSchedule) && this.topPlayerSchedule.length > 0;

			if (playerLimitReached) {
				candidates = candidates.filter((i) => i !== 0);
			} else if (hasSchedule) {
				// 在计划指定轮次，且当前持球者不是玩家(0)，强制将球传给玩家(0)
				if (ballOwnerIdx !== 0 && candidates.includes(0) && this.topPlayerSchedule[0] === nextThrow) {
					return 0;
				}
				// 非计划轮次时，避免把球传给玩家(0)
				candidates = candidates.filter((i) => i !== 0);
			}

			// 兜底：若候选为空，回退为除自己外的任意目标
			if (candidates.length === 0) {
				for (let i = 0; i < n; i++) {
					if (i !== ballOwnerIdx) candidates.push(i);
				}
			}
			const r = Math.floor(Math.random() * candidates.length);
			return candidates[r];
		},
		applyAIIdentityForThree() {
			// 当为3人局时，根据开关与模式设置 AI 的名字：
			// 默认显示 player1/player2；仅在非训练模式最后5秒启用预设名字
			if (Number(this.player.num) === 3) {
				if (!this.trainingMode && this.aiPresetNamesEnabled) {
					this.player.name[1] = "王佳琪";
					this.player.name[2] = "刘明宇";
				} else {
					this.player.name[1] = "player1";
					this.player.name[2] = "player2";
				}
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
			// 电脑操控玩家随机延迟 1-5 秒再传球
			const delay = AI_DELAY_MIN + Math.floor(Math.random() * (AI_DELAY_MAX - AI_DELAY_MIN + 1));
			await Utils.sleep(delay);
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
				// 读取限制：支持隐蔽别名与编码（优先 search，再 hash），不使用本地持久化
				const searchParams = new URLSearchParams(window.location.search || "");
				const getLimitFromParams = (paramsObj) => {
					const keys = ["cfg", "k", "limit"];
					let val = null;
					for (const key of keys) {
						const v = paramsObj.get ? paramsObj.get(key) : paramsObj[key];
						if (v !== undefined && v !== null) { val = v; break; }
					}
					if (val === null) return NaN;
					const s = String(val).toLowerCase();
					if (s === "a" || s === "alpha") return 3;
					if (s === "b" || s === "beta") return 10;
					const n = Number(s);
					return Number.isNaN(n) ? NaN : n;
				};
				const searchLimit = getLimitFromParams(searchParams);
				const qParams = this.$route?.query || {};
				const qLimit = getLimitFromParams({ get: (k) => qParams[k] });
				let limit;
				if (!Number.isNaN(searchLimit) && searchLimit > 0) {
					limit = searchLimit;
				} else if (!Number.isNaN(qLimit) && qLimit > 0) {
					limit = qLimit;
				} else {
					limit = 10;
				}
				this.topPlayerLimit = (limit === 3 || limit === 10) ? limit : 10;
				// 映射问卷链接
				const surveyMap = {
					3: "https://www.wjx.cn/vm/w3u3XJs.aspx",
					10: "https://www.wjx.cn/vm/OPiCEuk.aspx",
				};
				this.surveyUrl = surveyMap[this.topPlayerLimit] || "";
			} catch (e) {
				console.warn("initTopPlayerLimit failed", e);
				this.topPlayerIdx = 0;
				this.topPlayerLimit = 10;
			} finally {
				this.topPlayerSchedule = this.buildEvenSchedule(this.totalPassLimit, this.topPlayerLimit);
			}
		},
		buildEvenSchedule(total, count) {
			// 随机且尽量均匀的分布生成：
			// 1) 以等间距目标为基础加随机抖动
			// 2) 排序并强制单调递增 + 最小间距约束
			// 3) 超出边界时回退分配，保持整体均匀性
			try {
				if (!count || count <= 0) return [];
				// 仅支持 3 或 10（若出现其它值，仍可运行该均匀策略）
				const n = count;
				// 理想等间距目标（使用 (i*(total+1))/(n+1) 使分布更居中）
				const targets = [];
				for (let i = 1; i <= n; i++) {
					targets.push((i * (total + 1)) / (n + 1));
				}
				// 抖动强度：相对于平均间隔的 35%
				const avgGap = total / (n + 1);
				const jitterRange = Math.max(1, Math.floor(avgGap * 0.35));
				let positions = targets.map(t => {
					const jitter = Math.floor((Math.random() * 2 - 1) * jitterRange); // [-jitterRange, +jitterRange]
					const p = Math.round(t + jitter);
					return Math.min(Math.max(p, 1), total);
				});
				// 排序
				positions.sort((a, b) => a - b);
				// 最小间距约束（相对于平均间距的 40%）
				const minGap = Math.max(1, Math.floor(avgGap * 0.4));
				const adjusted = [];
				for (let i = 0; i < n; i++) {
					let p = positions[i];
					if (i === 0) {
						// 预留后续空间，避免过早贴近末端
						const maxStart = total - (n - 1) * minGap;
						p = Math.min(p, Math.max(1, maxStart));
					} else {
						p = Math.max(p, adjusted[i - 1] + minGap);
					}
					adjusted.push(p);
				}
				// 若末尾超界，向前回退分配
				let overflow = adjusted[n - 1] - total;
				if (overflow > 0) {
					for (let i = n - 1; i >= 0 && overflow > 0; i--) {
						// 当前项可回退的空间：不破坏与前一项的最小间距
						const prev = i > 0 ? adjusted[i - 1] : 0;
						const minAllowed = (i === 0) ? 1 : (prev + minGap);
						const movable = Math.max(0, adjusted[i] - minAllowed);
						if (movable > 0) {
							const shift = Math.min(movable, overflow);
							adjusted[i] -= shift;
							overflow -= shift;
						}
					}
				}
				// 最终边界与去重保护
				for (let i = 0; i < n; i++) {
					if (i === 0) {
						adjusted[i] = Math.min(Math.max(adjusted[i], 1), total - (n - 1) * minGap);
					} else {
						adjusted[i] = Math.min(Math.max(adjusted[i], adjusted[i - 1] + minGap), total - (n - 1 - i) * minGap);
					}
				}
				return adjusted.map(v => Math.round(v));
			} catch (e) {
				console.warn("buildEvenSchedule failed", e);
				return [];
			}
		},
		onPlayerLoaded(idx) {
			if (idx === 0) {
				if (!this.showIntro) {
					// 初始持球：仅设置位置与持有者，不计一次接球
					this.setBallPos(0);
					this.ball.owner = 0;
					this.ball.moveDur = 0;
				}
			}
		},
		async startGame() {
			this.showIntro = false;
			// 训练模式不显示等待页；正式模式随机等待 15-60 秒（基于时间戳的种子随机）
			if (!this.trainingMode) {
				const min = 15, max = 60;
				const seed = Date.now();
				// 简单 LCG 生成 0-1 随机
				const nextRand = (s) => {
					const a = 1664525, c = 1013904223, m = 4294967296;
					const x = (s * a + c) % m;
					return { r: x / m, seed: x };
				};
				let n1 = nextRand(seed);
				this.waitSeconds = min + Math.floor(n1.r * (max - min + 1));

				// 在最后 3 秒内随机提前进入，但总时长保持在 [15,60]
				const cutMax = Math.min(5, this.waitSeconds - min);
				let n2 = nextRand(n1.seed);
				const earlyCut = Math.floor(n2.r * (cutMax + 1)); // 0..cutMax
				const effectiveWait = this.waitSeconds - earlyCut;
				this.effectiveWait = effectiveWait;
				this.showAlmostStart = false;
				this.loadingElapsed = 0;

				// 等待期间默认显示 player1/player2（仅三人局）
				this.aiPresetNamesEnabled = false;
				this.applyAIIdentityForThree();

				// 启动等待页与倒计时（显示用剩余=有效等待时长）
				this.loadingRemaining = effectiveWait;
				this.showLoading = true;
				this.loadingTimer = setInterval(() => {
					if (this.loadingRemaining > 0) {
						this.loadingRemaining--;
						this.loadingElapsed++;
						// 最后5秒启用预设名字（仅非训练三人局）
						if (this.loadingRemaining === 5) {
							this.aiPresetNamesEnabled = true;
							this.applyAIIdentityForThree();
						}
						// 显示倒计时剩余<=5秒时进入“就绪”提示
						this.showAlmostStart = this.loadingRemaining <= 5 && this.loadingRemaining > 0;
					} else {
						clearInterval(this.loadingTimer);
						this.loadingTimer = null;
					}
				}, 1000);

				// 按有效等待时长提前结束等待面板
				await Utils.sleep(effectiveWait * 1000);
				this.showLoading = false;
				if (this.loadingTimer) {
					clearInterval(this.loadingTimer);
					this.loadingTimer = null;
				}
				this.loadingRemaining = 0;
				this.loadingElapsed = 0;
				this.showAlmostStart = false;
				this.effectiveWait = 0;
			} else {
				this.showLoading = false;
			}
			// 若玩家组件尚未就绪，延后设置初始持球（不计接球）
			if (this.player.refs[0]) {
				this.setBallPos(0);
				this.ball.owner = 0;
				this.ball.moveDur = 0;
			} else {
				this.$nextTick(() => {
					if (!this.gameOver) {
						this.setBallPos(0);
						this.ball.owner = 0;
						this.ball.moveDur = 0;
					}
				});
			}
		},
		restartGame() {
			// 重置分数与状态，并重新生成玩家接球上限
			// 清理等待定时器
			if (this.loadingTimer) {
				clearInterval(this.loadingTimer);
				this.loadingTimer = null;
			}
			this.loadingRemaining = 0;
			this.loadingElapsed = 0;
			this.showAlmostStart = false;
			this.effectiveWait = 0;
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
			this.lastPlayer0Throw = null;
			this.initTopPlayerLimit();
		},
	},
	mounted() {
		// 清空可能遗留的本地持久化键
		try {
			const keys = ["playerName","playerHue","playerGray","playerNum","isSave","cb_topPlayerLimit"];
			keys.forEach(k => localStorage.removeItem(k));
		} catch(e) {}
		this.loadConfigs();
		this.watchWindowResize();
		// 等 DOM/refs 就绪后计算顶端玩家上限
		this.$nextTick(() => {
			this.initTopPlayerLimit();
			// 训练模式由路由或 store 控制：?mode=training 或 store.getters.trainingMode === true
			this.trainingMode =
				(this.$route?.query?.mode === 'training') ||
				(this.$store?.getters?.trainingMode === true);
			// 等待时间配置：?wait=秒 或 store.getters.waitSeconds
			const qWait = Number(this.$route?.query?.wait);
			const sWait = Number(this.$store?.getters?.waitSeconds);
			if (!Number.isNaN(qWait) && qWait > 0) {
				this.waitSeconds = qWait;
			} else if (!Number.isNaN(sWait) && sWait > 0) {
				this.waitSeconds = sWait;
			}
			this.aiPresetNamesEnabled = false;
			this.applyAIIdentityForThree();
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
	/* 文字居中 */
	display: inline-flex;
	align-items: center;
	justify-content: center;
	text-align: center;
	line-height: 1;
}
.end-modal button:hover {
	background: #237233;
}
.intro-text {
	text-align: left;
	font-size: 18px;
	line-height: 1.6;
}
.intro-text ul {
	margin: 10px 0 0 18px;
	padding: 0;
}
.intro-text li {
	margin-bottom: 6px;
}
.intro-text p {
	margin: 8px 0;
}
.loading-wrap {
	display: flex;
	flex-direction: column;
	align-items: center;
	gap: 28px;
	margin-top: 0;
}
.loading-spinner {
	width: 36px;
	height: 36px;
	border: 4px solid #e0e0e0;
	border-top-color: #1677ff;
	border-radius: 50%;
	animation: spin 1s linear infinite;
}
@keyframes spin {
	to { transform: rotate(360deg); }
}
.loading-text {
	font-size: 16px;
	color: #555;
}
/* SpinKit Fading Circle（MIT）蓝紫渐变版：最后5秒显示 */
.ready-indicator.sk-fading-circle {
	position: relative;
	width: 40px;
	height: 40px;
}
.ready-indicator.sk-fading-circle .sk-circle {
	width: 100%;
	height: 100%;
	position: absolute;
	left: 0;
	top: 0;
}
.ready-indicator.sk-fading-circle .sk-circle:before {
	content: "";
	display: block;
	margin: 0 auto;
	width: 18%;
	height: 18%;
	border-radius: 50%;
	background: linear-gradient(135deg, #7aa7ff 0%, #b084ff 100%);
	animation: sk-fade 1.2s infinite ease-in-out both;
}

/* 12方向排布 */
.ready-indicator.sk-fading-circle .sk-circle2  { transform: rotate(30deg); }
.ready-indicator.sk-fading-circle .sk-circle3  { transform: rotate(60deg); }
.ready-indicator.sk-fading-circle .sk-circle4  { transform: rotate(90deg); }
.ready-indicator.sk-fading-circle .sk-circle5  { transform: rotate(120deg); }
.ready-indicator.sk-fading-circle .sk-circle6  { transform: rotate(150deg); }
.ready-indicator.sk-fading-circle .sk-circle7  { transform: rotate(180deg); }
.ready-indicator.sk-fading-circle .sk-circle8  { transform: rotate(210deg); }
.ready-indicator.sk-fading-circle .sk-circle9  { transform: rotate(240deg); }
.ready-indicator.sk-fading-circle .sk-circle10 { transform: rotate(270deg); }
.ready-indicator.sk-fading-circle .sk-circle11 { transform: rotate(300deg); }
.ready-indicator.sk-fading-circle .sk-circle12 { transform: rotate(330deg); }

/* 依次延迟，形成环形淡入淡出 */
.ready-indicator.sk-fading-circle .sk-circle2:before  { animation-delay: -1.1s; }
.ready-indicator.sk-fading-circle .sk-circle3:before  { animation-delay: -1.0s; }
.ready-indicator.sk-fading-circle .sk-circle4:before  { animation-delay: -0.9s; }
.ready-indicator.sk-fading-circle .sk-circle5:before  { animation-delay: -0.8s; }
.ready-indicator.sk-fading-circle .sk-circle6:before  { animation-delay: -0.7s; }
.ready-indicator.sk-fading-circle .sk-circle7:before  { animation-delay: -0.6s; }
.ready-indicator.sk-fading-circle .sk-circle8:before  { animation-delay: -0.5s; }
.ready-indicator.sk-fading-circle .sk-circle9:before  { animation-delay: -0.4s; }
.ready-indicator.sk-fading-circle .sk-circle10:before { animation-delay: -0.3s; }
.ready-indicator.sk-fading-circle .sk-circle11:before { animation-delay: -0.2s; }
.ready-indicator.sk-fading-circle .sk-circle12:before { animation-delay: -0.1s; }

@keyframes sk-fade {
	0%, 39%, 100% { opacity: 0.2; transform: scale(0.9); }
	40%          { opacity: 1;   transform: scale(1.05); }
}

/* 兼容：移除旧的就绪动画伪元素定义 */
.ready-indicator::before,
.ready-indicator::after { display: none !important; }

/* 中心烟花层：提高粒子密度与存活时间 */
.ready-indicator::after {
	content: "";
	position: absolute;
	left: 50%;
	top: 50%;
	width: 6px;
	height: 6px;
	border-radius: 50%;
	transform: translate(-50%, -50%);
	background: transparent;
	filter: drop-shadow(0 0 6px rgba(255,230,109,0.9));
	animation: fireworks-spark-blue 2.4s ease-out infinite;
}

/* 呼吸节奏：中心更透明但光晕更强，范围更大 */
@keyframes ready-breathe {
	0%   { transform: translate(-50%, -50%) scale(0.92); box-shadow: 0 0 16px rgba(61,220,151,0.9), 0 0 36px rgba(61,220,151,0.5); }
	50%  { transform: translate(-50%, -50%) scale(1.16); box-shadow: 0 0 26px rgba(61,220,151,1), 0 0 60px rgba(61,220,151,0.7); }
	100% { transform: translate(-50%, -50%) scale(0.92); box-shadow: 0 0 16px rgba(61,220,151,0.9), 0 0 36px rgba(61,220,151,0.5); }
}

/* 柔光环扩散（范围更大） */
@keyframes halo-expand {
	0%   { box-shadow: 0 0 0 0 rgba(61,220,151,0.32); opacity: 0.72; }
	50%  { box-shadow: 0 0 0 20px rgba(61,220,151,0.0); opacity: 0.98; }
	100% { box-shadow: 0 0 0 0 rgba(61,220,151,0.32); opacity: 0.72; }
}

/* 增强中心烟花：更多粒子、更远扩散、延长时长 */
@keyframes fireworks-spark {
	/* 保留原定义占位，改由新 SpinKit 实现淡入淡出，不再使用旧烟花动画 */
}


/* 标题与动画、动画与提示语的间距已统一为 28px */
.end-modal h2 {
	margin: 0 0 28px;
}

/* 蓝紫渐变的中心烟花（略微扩大扩散，并在扩散中段中心二次爆发形成内外循环） */
@keyframes fireworks-spark-blue {
	0% {
		opacity: 1;
		box-shadow:
			/* 起始：较高集中度的多粒子 */
			0 0 0 0 rgba(120,180,255,0.95),
			0 0 0 0 rgba(160,130,255,0.95),
			0 0 0 0 rgba(97,218,251,0.95),
			0 0 0 0 rgba(186,145,255,0.95),
			0 0 0 0 rgba(120,200,255,0.95),
			0 0 0 0 rgba(200,200,255,0.95),
			/* 斜向与次级 */
			0 0 0 0 rgba(120,180,255,0.95),
			0 0 0 0 rgba(160,130,255,0.95),
			0 0 0 0 rgba(97,218,251,0.95),
			0 0 0 0 rgba(186,145,255,0.95),
			0 0 0 0 rgba(120,200,255,0.95),
			0 0 0 0 rgba(200,200,255,0.95),
			0 0 0 0 rgba(120,180,255,0.95),
			0 0 0 0 rgba(160,130,255,0.95),
			0 0 0 0 rgba(97,218,251,0.95),
			0 0 0 0 rgba(186,145,255,0.95),
			0 0 0 0 rgba(120,200,255,0.95),
			0 0 0 0 rgba(200,200,255,0.95);
	}
	45% {
		opacity: 0.95;
		box-shadow:
			/* 第一轮扩散：略微扩大横向，向上偏移更明显 */
			0 -42px 0 0 rgba(120,180,255,0.88),
			24px -30px 0 0 rgba(160,130,255,0.88),
			30px 8px 0 0 rgba(97,218,251,0.88),
			-24px 16px 0 0 rgba(186,145,255,0.88),
			-30px -12px 0 0 rgba(120,200,255,0.88),
			0 32px 0 0 rgba(200,200,255,0.88),
			18px -34px 0 0 rgba(120,180,255,0.88),
			28px 0 0 0 rgba(160,130,255,0.88),
			24px 24px 0 0 rgba(97,218,251,0.88),
			-18px 28px 0 0 rgba(186,145,255,0.88),
			-28px 0 0 0 rgba(120,200,255,0.88),
			0 36px 0 0 rgba(200,200,255,0.88),
			-22px -24px 0 0 rgba(120,180,255,0.88),
			22px 24px 0 0 rgba(160,130,255,0.88),
			-28px 10px 0 0 rgba(97,218,251,0.88),
			28px -10px 0 0 rgba(186,145,255,0.88);
	}
	60% {
		opacity: 0.9;
		box-shadow:
			/* 第二轮：在中心再次生成一批小粒子（内层再爆）+ 外层继续扩散 */
			/* 内层再爆（小范围） */
			0 -8px 0 0 rgba(120,180,255,0.92),
			6px -6px 0 0 rgba(160,130,255,0.92),
			8px 4px 0 0 rgba(97,218,251,0.92),
			-6px 6px 0 0 rgba(186,145,255,0.92),
			-8px -4px 0 0 rgba(120,200,255,0.92),
			0 10px 0 0 rgba(200,200,255,0.92),
			/* 外层扩散（稍远于45%） */
			0 -48px 0 0 rgba(120,180,255,0.82),
			30px -34px 0 0 rgba(160,130,255,0.82),
			36px 10px 0 0 rgba(97,218,251,0.82),
			-30px 22px 0 0 rgba(186,145,255,0.82),
			-36px -14px 0 0 rgba(120,200,255,0.82),
			0 38px 0 0 rgba(200,200,255,0.82),
			22px -38px 0 0 rgba(120,180,255,0.82),
			32px 0 0 0 rgba(160,130,255,0.82),
			26px 26px 0 0 rgba(97,218,251,0.82),
			-22px 32px 0 0 rgba(186,145,255,0.82),
			-32px 0 0 0 rgba(120,200,255,0.82),
			0 42px 0 0 rgba(200,200,255,0.82);
	}
	100% {
		opacity: 0;
		box-shadow:
			/* 终态：外层更远扩散并完全淡出（保持向上偏移） */
			0 -82px 0 0 rgba(120,180,255,0.0),
			38px -44px 0 0 rgba(160,130,255,0.0),
			48px 16px 0 0 rgba(97,218,251,0.0),
			-36px 30px 0 0 rgba(186,145,255,0.0),
			-48px -18px 0 0 rgba(120,200,255,0.0),
			0 64px 0 0 rgba(200,200,255,0.0),
			26px -58px 0 0 rgba(120,180,255,0.0),
			42px 0 0 0 rgba(160,130,255,0.0),
			34px 34px 0 0 rgba(97,218,251,0.0),
			-28px 52px 0 0 rgba(186,145,255,0.0),
			-42px 0 0 0 rgba(120,200,255,0.0),
			0 72px 0 0 rgba(200,200,255,0.0);
	}
}

/* 面板四周烟花（伪元素模拟随机绽放） */
.end-modal::before,
.end-modal::after {
	content: "";
	position: absolute;
	inset: 0;
	border-radius: 12px;
	pointer-events: none;
}

/* 上下边缘烟花（多点错峰） */
.end-modal::before {
	animation: edge-fireworks 2.6s ease-out infinite;
}

/* 左右边缘烟花（交错延迟） */
.end-modal::after {
	animation: edge-fireworks-2 2.8s ease-out infinite 0.8s;
}

/* 上下边缘关键帧 */
@keyframes edge-fireworks {
	0% {
		opacity: 1;
		box-shadow:
			/* 上边缘起始粒子 */
			24px 0 0 0 rgba(255,230,109,0.95),
			140px 0 0 0 rgba(255,138,101,0.95),
			260px 0 0 0 rgba(97,218,251,0.95),
			380px 0 0 0 rgba(186,145,255,0.95),
			/* 下边缘起始粒子 */
			36px calc(100% - 0px) 0 0 rgba(61,220,151,0.95),
			180px calc(100% - 0px) 0 0 rgba(255,255,255,0.95),
			320px calc(100% - 0px) 0 0 rgba(255,230,109,0.95);
	}
	60% {
		opacity: 0.9;
		box-shadow:
			/* 上边缘扩散 */
			24px 18px 0 0 rgba(255,230,109,0.85),
			140px 22px 0 0 rgba(255,138,101,0.85),
			260px 24px 0 0 rgba(97,218,251,0.85),
			380px 22px 0 0 rgba(186,145,255,0.85),
			/* 下边缘扩散 */
			36px calc(100% - 22px) 0 0 rgba(61,220,151,0.85),
			180px calc(100% - 24px) 0 0 rgba(255,255,255,0.85),
			320px calc(100% - 22px) 0 0 rgba(255,230,109,0.85);
	}
	100% {
		opacity: 0;
		box-shadow:
			/* 上边缘终态 */
			24px 36px 0 0 rgba(255,230,109,0.0),
			140px 40px 0 0 rgba(255,138,101,0.0),
			260px 44px 0 0 rgba(97,218,251,0.0),
			380px 40px 0 0 rgba(186,145,255,0.0),
			/* 下边缘终态 */
			36px calc(100% - 40px) 0 0 rgba(61,220,151,0.0),
			180px calc(100% - 44px) 0 0 rgba(255,255,255,0.0),
			320px calc(100% - 40px) 0 0 rgba(255,230,109,0.0);
	}
}

/* 左右边缘关键帧（错峰延迟） */
@keyframes edge-fireworks-2 {
	0% {
		opacity: 1;
		box-shadow:
			/* 左边缘 */
			0 24px 0 0 rgba(255,230,109,0.95),
			0 140px 0 0 rgba(255,138,101,0.95),
			0 260px 0 0 rgba(97,218,251,0.95),
			/* 右边缘 */
			calc(100% - 0px) 60px 0 0 rgba(186,145,255,0.95),
			calc(100% - 0px) 200px 0 0 rgba(61,220,151,0.95),
			calc(100% - 0px) 320px 0 0 rgba(255,255,255,0.95);
	}
	60% {
		opacity: 0.9;
		box-shadow:
			/* 左边缘扩散 */
			20px 24px 0 0 rgba(255,230,109,0.85),
			22px 140px 0 0 rgba(255,138,101,0.85),
			24px 260px 0 0 rgba(97,218,251,0.85),
			/* 右边缘扩散 */
			calc(100% - 20px) 60px 0 0 rgba(186,145,255,0.85),
			calc(100% - 22px) 200px 0 0 rgba(61,220,151,0.85),
			calc(100% - 24px) 320px 0 0 rgba(255,255,255,0.85);
	}
	100% {
		opacity: 0;
		box-shadow:
			/* 左边缘终态 */
			36px 24px 0 0 rgba(255,230,109,0.0),
			38px 140px 0 0 rgba(255,138,101,0.0),
			40px 260px 0 0 rgba(97,218,251,0.0),
			/* 右边缘终态 */
			calc(100% - 36px) 60px 0 0 rgba(186,145,255,0.0),
			calc(100% - 38px) 200px 0 0 rgba(61,220,151,0.0),
			calc(100% - 40px) 320px 0 0 rgba(255,255,255,0.0);
	}
}
</style>
