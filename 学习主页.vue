<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">学习主页</span>
      <span class="page-head-sub">财数智析 · 沉浸式学习智能体（学生端）</span>
      <button class="btn-ghost" @click="refresh">刷新数据</button>
    </div>

    <!-- 第一行：Hero 横幅 + 三智能体协同 -->
    <div class="grid-row1">
      <!-- Hero 横幅 -->
      <div class="hero">
        <div class="hero-line">
          <span class="hero-hello">你好，{{ student.name }}！</span>
          <span class="tag tag-light">{{ student.cls }}</span>
          <span class="hero-flame">连续学习 {{ student.streak }} 天</span>
        </div>
        <div class="xp-wrap">
          <div class="xp-bar"><div class="xp-fill" :style="{ width: xpPct + '%' }"></div></div>
        </div>
        <div class="hero-actions">
          <button class="btn btn-light" @click="continueLearn">继续学习</button>
          <span class="hero-hint">当前任务：{{ activeTask.name }}</span>
        </div>
      </div>

      <!-- 三智能体协同 -->
      <div class="card agent-card">
        <div class="section-title"><span class="bar"></span>三智能体协同</div>
        <div class="agent-row">
          <div class="agent-chip chip-ideo-new">立德 · 启智立德</div>
          <div class="agent-chip chip-tech-new">炼技 · 入岗炼技</div>
          <div class="agent-chip chip-grow-new">增值 · 促学成长</div>
        </div>
        <p class="agent-desc mt8">三个智能体贯穿「导学筑基 → 实操历练 → 评价反馈」，德技并育人。</p>
      </div>
    </div>

    <!-- 第二部分：3 列 × 2 行 网格 -->
    <div class="grid-3 mt2">
      <!-- 左列 上：当前等级 -->
      <div class="card level-card">
        <div class="section-title"><span class="bar"></span>当前等级 &nbsp;LV.{{ student.level }}</div>
        <div class="lv-row"><span class="lv-k">经验值</span><span class="lv-v">{{ student.xp }}/{{ student.xpMax }}</span></div>
        <div class="lv-row"><span class="lv-k">已完成任务</span><span class="lv-v c-red">{{ doneCount }} 个</span></div>
        <div class="lv-row"><span class="lv-k">平均准确率</span><span class="lv-v">{{ student.accuracy }}%</span></div>
        <div class="lv-row"><span class="lv-k">综合评价</span><span class="lv-v c-red">{{ overallGrade }}</span></div>
      </div>

      <!-- 中列 上：能力画像 -->
      <div class="card">
        <div class="section-title"><span class="bar"></span>能力画像</div>
        <div v-for="a in student.abilityProg" :key="a.k" class="ability-item">
          <span class="ab-name">{{ a.k }}</span>
          <div class="bar-bg tall"><div class="bar-fill" :class="weakCls(a.v)" :style="{ width: a.v + '%' }"></div></div>
          <span class="ab-val">{{ a.v }}</span>
        </div>
      </div>

      <!-- 右列 上：成就墙 -->
      <div class="card">
        <div class="section-title"><span class="bar"></span>成就墙</div>
        <div class="badge-grid">
          <div v-for="b in topBadges" :key="b.n" class="badge-item" :class="{ off: !b.earned }">
            <div class="badge-ic">{{ b.n[0] }}</div>
            <div class="badge-name">{{ b.n }}</div>
            <div class="badge-d">{{ b.d }}</div>
          </div>
        </div>
      </div>

      <!-- 左列 下：学习动态 -->
      <div class="card">
        <div class="section-title"><span class="bar"></span>学习动态</div>
        <div v-for="(f, i) in student.feed" :key="i" class="feed-item">
          <span class="feed-dot" :class="feedCls(f.icon)"></span>
          <div class="feed-body">
            <div class="feed-txt">{{ f.txt }}</div>
            <div class="feed-time">{{ f.time }}</div>
          </div>
        </div>
      </div>

      <!-- 中列 下：继续学习 -->
      <div class="card">
        <div class="section-title"><span class="bar"></span>继续学习</div>
        <div class="resume" @click="openTask(activeTask)">
          <div class="resume-top">
            <span class="tag" :class="lvTag(activeTask.level)">{{ activeTask.level }}</span>
            <span class="tag tag-gray">{{ activeTask.module }}</span>
            <span class="tag tag-gray">{{ activeTask.ability }}</span>
            <span class="resume-go">继续 →</span>
          </div>
          <div class="resume-name">{{ activeTask.name }}</div>
          <p class="muted">点按进入任务工作台，跟随四步引导完成本次数据分析。</p>
        </div>
      </div>

      <!-- 右列 下：AI 为你推荐 -->
      <div class="card rec-card">
        <div class="section-title"><span class="bar"></span>AI 为你推荐</div>
        <div class="rec-name">{{ student.recommend.name }}</div>
        <div class="rec-ability"><span class="tag tag-orange">{{ student.recommend.ability }}</span></div>
        <p class="rec-reason">{{ student.recommend.reason }}</p>
        <button class="btn btn-primary btn-sm mt8" @click="askMentor">问导师</button>
      </div>
    </div>

    <!-- 提示 -->
    <div v-if="tip" class="tip">{{ tip }}</div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";

const emit = defineEmits(["navigate"]);

// ============ 数据服务（localStorage 共享，PRESET 兜底） ============
function lsGet(k) { try { const v = localStorage.getItem(k); return v ? JSON.parse(v) : null; } catch (e) { return null; } }
function lsSet(k, v) { try { localStorage.setItem(k, JSON.stringify(v)); } catch (e) { /* 忽略 */ } }
const LS_STU = "fd_student_v1";

const PRESET_STUDENT = {
  name: "林晓", cls: "财管2201", level: 3, lvName: "数据分析员", xp: 640, xpMax: 1000,
  tasks: [
    { id: "T01", name: "财报数据治理：多系统对账与清洗", level: "初级", module: "M1", ability: "数据治理", status: "done", score: 92 },
    { id: "T02", name: "三大报表结构解读与勾稽关系", level: "初级", module: "M1", ability: "报表理解", status: "done", score: 88 },
    { id: "T03", name: "毛利率异常分析与费用率排查", level: "初级", module: "M2", ability: "指标分析", type: "analysis", status: "active", score: null },
    { id: "T04", name: "多维度经营解析（产品/地区/渠道）", level: "中级", module: "M2", ability: "多维分析", type: "analysis", status: "locked" },
    { id: "T05", name: "异常特征识别与帕累托分析", level: "中级", module: "M3", ability: "营运能力", type: "analysis", status: "locked" },
    { id: "T06", name: "成本结构量价拆分模型", level: "中级", module: "M3", ability: "成本控制", type: "analysis", status: "locked" },
    { id: "T07", name: "现金流分析与利润质量评价", level: "中级", module: "M4", ability: "现金流分析", type: "analysis", status: "locked" },
    { id: "T08", name: "偿债能力分层与风险预警诊断", level: "高级", module: "M4", ability: "风险特警", type: "analysis", status: "locked" },
    { id: "T09", name: "非经常性损益与宏观环境影响分析", level: "高级", module: "M5", ability: "综合分析", type: "analysis", status: "locked" },
    { id: "T10", name: "动态仪表盘搭建与BI看板优化", level: "高级", module: "M6", ability: "可视化", type: "analysis", status: "locked" },
    { id: "T11", name: "随堂测验·选择题（现金流/偿债/营运）", level: "初级", module: "M2", ability: "指标分析", type: "quiz", status: "done", score: 100 },
    { id: "T12", name: "随堂练习·综合财报计算题", level: "中级", module: "M4", ability: "综合分析", type: "calc", status: "done", score: 92 }
  ],
  currentTask: null,
  lastFeedback: null,
  badges: [
    { n: "数据侦探", d: "发现关键异常指标", earned: true },
    { n: "报表达人", d: "准确解读三大报表", earned: true },
    { n: "成本管家", d: "完成成本优化任务", earned: false },
    { n: "图表设计师", d: "高质量可视化分析", earned: true },
    { n: "经营参谋", d: "提出有效改进建议", earned: false },
    { n: "财务洞察官", d: "完成综合经营挑战", earned: false }
  ],
  leaderboard: [
    { nm: "王浩", xp: 880, cls: "财管2201" }, { nm: "林晓", xp: 640, cls: "财管2201" },
    { nm: "陈雨", xp: 610, cls: "财管2201" }, { nm: "李娜", xp: 555, cls: "财管2202" },
    { nm: "赵磊", xp: 498, cls: "财管2201" }, { nm: "周婷", xp: 460, cls: "财管2202" }
  ],
  streak: 5, accuracy: 89, lastActive: "今天 09:24",
  recommend: {
    id: "T04", name: "多维度经营解析（产品/地区/渠道）", ability: "多维分析",
    reason: "你已完成毛利率与费用率分析（M2 前半），下一步建议做多维度经营拆解，补齐「多维分析」短板后进入 M3 异常特征识别。"
  },
  abilityProg: [
    { k: "报表理解", v: 92 }, { k: "指标分析", v: 85 }, { k: "异常识别", v: 70 },
    { k: "现金流分析", v: 64 }, { k: "可视化表达", v: 88 }, { k: "风险预警", v: 58 }
  ],
  abilityRadar: {
    axes: ["财报数据治理", "经营解析能力", "异常特征识别", "现金流分析", "风险预警能力", "可视化建模"],
    cur: [78, 82, 76, 70, 58, 84], avg: [75, 72, 74, 70, 66, 71]
  },
  ethicsScenes: [
    { topic: "数据真实性", kp: "数据真实性核验", theme: "数据诚信", title: "数字背后的责任",
      scene: "你在录入某上市公司三季度营收数据时发现：把一笔 1,200 万的收入错计到下季度，本月营收指标恰好「达标」。系统提示「数据异常波动」，你是否修正？",
      guide: ["若小数点或归属期错配，可能误导怎样的经营决策？", "数据真实为什么是财务人的第一底线？", "遇到「达标」诱惑时，应如何守住职业底线？"],
      case: "《会计法》明确要求会计核算以实际发生的经济业务为依据——虚构、提前确认收入均属违法。" }
  ],
  feed: [
    { icon: "check", txt: "完成「三大报表结构解读与勾稽关系」，得分 88", time: "2 天前" },
    { icon: "star", txt: "解锁「图表设计师」徽章", time: "3 天前" },
    { icon: "trophy", txt: "班级排名上升至第 2 名", time: "本周" },
    { icon: "bot", txt: "AI 导师为你推荐「多维度经营解析」", time: "今天" }
  ]
};

function loadStudent() {
  let s = lsGet(LS_STU);
  if (!s) { s = JSON.parse(JSON.stringify(PRESET_STUDENT)); lsSet(LS_STU, s); }
  return s;
}

// ============ 页面状态 ============
const student = ref(loadStudent());
const tip = ref("");
let tipTimer = null;
function showTip(t) { tip.value = t; clearTimeout(tipTimer); tipTimer = setTimeout(function () { tip.value = ""; }, 2400); }

function refresh() { student.value = loadStudent(); }

// ============ 计算属性 ============
const doneCount = computed(function () { return student.value.tasks.filter(function (x) { return x.status === "done"; }).length; });
const activeTask = computed(function () { return student.value.tasks.find(function (x) { return x.status === "active"; }) || student.value.tasks[2]; });
const xpPct = computed(function () { return Math.round(student.value.xp / student.value.xpMax * 100); });
const topBadges = computed(function () { return student.value.badges.slice(0, 4); });
const overallGrade = computed(function () {
  const acc = student.value.accuracy;
  if (acc >= 90) return "S等";
  if (acc >= 80) return "A等";
  if (acc >= 70) return "B等";
  if (acc >= 60) return "C等";
  return "D等";
});

// ============ 交互 ============
function go(view) { emit("navigate", view); }
function openTask(t) {
  if (!t || t.status === "locked") { showTip("该任务尚未解锁，请先完成前置任务"); return; }
  student.value.currentTask = t;
  lsSet(LS_STU, student.value);
  go("student-task");
}
function continueLearn() { openTask(activeTask.value); }
function askMentor() {
  student.value.currentTask = activeTask.value;
  lsSet(LS_STU, student.value);
  go("student-task");
  showTip("已进入任务工作台，可在右侧向 AI 导师提问");
}

// ============ 样式辅助 ============
function lvTag(lv) { return lv === "初级" ? "tag-blue" : lv === "中级" ? "tag-orange" : "tag-red"; }
function weakCls(v) { return v < 60 ? "f-red" : v < 75 ? "f-orange" : "f-blue"; }
function feedCls(k) { return { check: "d-green", star: "d-orange", trophy: "d-gold", bot: "d-blue" }[k] || "d-blue"; }

onMounted(function () { });
</script>

<style scoped>
/* ============ 基线 ============ */
.pg { max-width: 1220px; margin: 0 auto; padding: 18px 16px 40px; }
.page-head { display: flex; align-items: center; gap: 10px; margin-bottom: 16px; flex-wrap: wrap; }
.page-head-bar { width: 4px; height: 22px; border-radius: 2px; background: #1677FF; }
.page-head-title { font-size: 20px; font-weight: 600; color: #1F2733; }
.page-head-sub { font-size: 13px; color: #97A1B2; flex: 1; }
.btn { display: inline-flex; align-items: center; gap: 6px; height: 34px; padding: 0 16px; border-radius: 6px; font-size: 14px; font-weight: 600; border: 1px solid transparent; cursor: pointer; transition: all .15s ease; white-space: nowrap; }
.btn:disabled { opacity: .5; cursor: not-allowed; }
.btn-primary { background: #2B6CD6; color: #fff; }
.btn-primary:hover { background: #4D8BE8; }
.btn-ghost { color: #2B6CD6; background: #EAF2FC; }
.btn-ghost:hover { background: #DCEBFB; }
.btn-light { background: #fff; color: #2B6CD6; }
.btn-sm { height: 28px; padding: 0 12px; font-size: 13px; }
.mt { margin-top: 16px; }
.mt2 { margin-top: 16px; }
.mt8 { margin-top: 8px; }
.muted { color: #5A6577; font-size: 13px; }
.card { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px; box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); }
.section-title { display: flex; align-items: center; gap: 8px; font-size: 15px; font-weight: 600; color: #1F2733; margin-bottom: 14px; }
.section-title .bar { width: 4px; height: 15px; border-radius: 2px; background: #2B6CD6; }
.tag { display: inline-flex; align-items: center; font-size: 12px; padding: 1px 8px; border-radius: 4px; font-weight: 500; white-space: nowrap; }
.tag-blue { background: #EAF2FC; color: #1D4FA8; }
.tag-orange { background: #FFF3E8; color: #C96A06; }
.tag-red { background: #FFECEC; color: #D93025; }
.tag-green { background: #F1FAE8; color: #3C8E12; }
.tag-gray { background: #F7FAFD; color: #97A1B2; }
.tag-light { background: rgba(255, 255, 255, .25); color: #fff; }
.bar-bg { background: #EEF3FA; border-radius: 99px; overflow: hidden; }
.bar-bg.tall { height: 8px; }
.bar-fill { height: 100%; border-radius: 99px; transition: width .4s ease; }
.f-blue { background: #2B6CD6; }
.f-orange { background: #FA8C16; }
.f-red { background: #FF4D4F; }
.c-red { color: #E64545 !important; }
.tip { margin-top: 12px; background: #EAF2FC; border: 1px solid #DCEBFB; color: #1D4FA8; font-size: 13px; border-radius: 6px; padding: 8px 12px; }

/* ============ 第一行：Hero + 三智能体 ============ */
.grid-row1 { display: grid; grid-template-columns: 1.15fr 1fr; gap: 16px; }

/* ============ Hero 横幅 ============ */
.hero { background: linear-gradient(135deg, #2B6CD6 0%, #4D8BE8 60%, #6FA3F0 100%); border-radius: 10px; padding: 22px 22px 18px; color: #fff; box-shadow: 0 4px 14px rgba(43, 108, 214, .25); }
.hero-line { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; margin-bottom: 12px; }
.hero-hello { font-size: 20px; font-weight: 700; }
.hero-flame { display: inline-flex; align-items: center; font-size: 12px; font-weight: 600; background: rgba(255, 255, 255, .22); padding: 2px 10px; border-radius: 99px; }
.xp-wrap { margin-bottom: 14px; }
.xp-bar { width: 100%; height: 10px; background: rgba(255, 255, 255, .28); border-radius: 99px; overflow: hidden; }
.xp-fill { height: 100%; background: linear-gradient(90deg, #FA8C16, #FFD591); border-radius: 99px; transition: width .5s ease; }
.hero-actions { display: flex; align-items: center; gap: 12px; }
.hero-hint { font-size: 13px; opacity: .92; }

/* ============ 三智能体协同（新配色 chip） ============ */
.agent-card { border: 2px dashed #86B0E8; background: #F8FBFF; }
.agent-row { display: flex; flex-wrap: wrap; gap: 12px; }
.agent-chip { padding: 16px 14px; border-radius: 8px; font-size: 16px; font-weight: 700; color: #fff; text-align: center; flex: 1; min-width: 110px; }
.chip-ideo-new { background: #FF6B6B; }
.chip-tech-new { background: #4D8BE8; }
.chip-grow-new { background: #67C23A; }
.agent-desc { color: #52A443; font-size: 14px; font-weight: 600; line-height: 22px; }

/* ============ 3 列 × 2 行 网格 ============ */
.grid-3 { display: grid; grid-template-columns: 1fr 1.4fr 1fr; gap: 16px; }
@media (max-width: 1020px) { .grid-3 { grid-template-columns: 1fr 1fr; } .grid-row1 { grid-template-columns: 1fr; } }
@media (max-width: 680px) { .grid-3 { grid-template-columns: 1fr; } }

/* ============ 当前等级卡片 ============ */
.level-card .lv-row { display: flex; align-items: center; justify-content: space-between; padding: 8px 0; border-bottom: 1px solid #EEF2F8; }
.level-card .lv-row:last-child { border-bottom: none; }
.lv-k { font-size: 14px; color: #5A6577; }
.lv-v { font-size: 15px; font-weight: 600; color: #1F2733; }

/* ============ 继续学习 / AI 推荐 ============ */
.resume { border: 1px solid #E3E9F2; border-radius: 8px; padding: 14px; cursor: pointer; transition: all .15s ease; }
.resume:hover { border-color: #2B6CD6; box-shadow: 0 2px 8px rgba(43, 108, 214, .12); }
.resume-top { display: flex; align-items: center; gap: 6px; margin-bottom: 8px; flex-wrap: wrap; }
.resume-go { margin-left: auto; color: #2B6CD6; font-weight: 600; font-size: 13px; }
.resume-name { font-size: 16px; font-weight: 600; color: #1F2733; margin-bottom: 4px; }
.rec-card { background: linear-gradient(180deg, #EAF2FC, #fff 70%); }
.rec-name { font-size: 16px; font-weight: 600; color: #1F2733; margin-bottom: 6px; }
.rec-reason { font-size: 13px; color: #5A6577; line-height: 21px; }

/* ============ 能力画像 / 学习动态 ============ */
.ability-item { display: flex; align-items: center; gap: 10px; margin-bottom: 10px; }
.ab-name { width: 72px; font-size: 13px; color: #5A6577; flex: none; }
.bar-bg.tall { flex: 1; }
.ab-val { width: 28px; text-align: right; font-size: 13px; font-weight: 600; color: #1F2733; }
.feed-item { display: flex; gap: 10px; padding: 8px 0; border-bottom: 1px solid #EEF2F8; }
.feed-item:last-child { border-bottom: none; }
.feed-dot { width: 8px; height: 8px; border-radius: 50%; margin-top: 6px; flex: none; }
.d-green { background: #52C41A; }
.d-orange { background: #FA8C16; }
.d-gold { background: #FAAD14; }
.d-blue { background: #2B6CD6; }
.feed-txt { font-size: 13px; color: #1F2733; line-height: 20px; }
.feed-time { font-size: 12px; color: #97A1B2; margin-top: 2px; }

/* ============ 成就墙 ============ */
.badge-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 10px; }
.badge-item { border: 1px solid #E3E9F2; border-radius: 8px; padding: 12px; text-align: center; background: #FBFEF9; }
.badge-item.off { background: #F7FAFD; opacity: .6; }
.badge-ic { width: 40px; height: 40px; border-radius: 50%; background: linear-gradient(135deg, #52C41A, #95DE64); color: #fff; font-size: 18px; font-weight: 700; display: flex; align-items: center; justify-content: center; margin: 0 auto 6px; }
.badge-item.off .badge-ic { background: #D9DEE8; }
.badge-name { font-size: 13px; font-weight: 600; color: #1F2733; }
.badge-d { font-size: 11px; color: #97A1B2; margin-top: 2px; }
</style>
