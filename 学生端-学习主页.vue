<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">学习主页</span>
      <span class="page-head-sub">“立德-炼技-增值”多智能体沉浸式学习平台（学生端）</span>
      <button class="btn-ghost" @click="refresh">刷新数据</button>
    </div>

    <!-- 第一行：Hero 横幅 + 三智能体协同 -->
    <div class="grid-row1">
      <!-- Hero 横幅 -->
      <div class="hero">
        <div class="hero-line">
          <span class="hero-hello">你好，{{ student.name }}！</span>
          <span class="tag tag-light">{{ student.cls }}</span>
          <span class="hero-lv">Lv.{{ student.level }} {{ student.lvName }}</span>
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
        <div class="section-title"><span class="bar"></span>三智能体协同<span class="card-sub">点击卡片进入对应智能体应用</span></div>
        <div class="agent-row">
          <div class="agent-chip chip-ideo-new" @click="go('student-ethics')">
            <span class="agent-ic">
              <svg viewBox="0 0 24 24" fill="none" stroke="#fff" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 21s-7-4.6-9.3-9A5.4 5.4 0 0 1 12 6.6 5.4 5.4 0 0 1 21.3 12C19 16.4 12 21 12 21z"/></svg>
            </span>
            <span class="agent-txt"><b>立德 · 启智立德</b><small>立德Agent·专业思政思辨</small></span>
            <span class="agent-go">进入 ›</span>
          </div>
          <div class="agent-chip chip-tech-new" @click="go('student-task')">
            <span class="agent-ic">
              <svg viewBox="0 0 24 24" fill="none" stroke="#fff" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="9"/><circle cx="12" cy="12" r="4.5"/><circle cx="12" cy="12" r="1" fill="#fff"/></svg>
            </span>
            <span class="agent-txt"><b>炼技 · 入岗炼技</b><small>炼技Agent·智能评分伴学</small></span>
            <span class="agent-go">进入 ›</span>
          </div>
          <div class="agent-chip chip-grow-new" @click="go('student-growth')">
            <span class="agent-ic">
              <svg viewBox="0 0 24 24" fill="none" stroke="#fff" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M3 17l6-6 4 4 8-9"/><path d="M16 6h5v5"/></svg>
            </span>
            <span class="agent-txt"><b>增值 · 促学成长</b><small>增值Agent·多元促学评价</small></span>
            <span class="agent-go">进入 ›</span>
          </div>
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
          <p class="muted">点按进入技能训练，跟随四步引导完成本次数据分析。</p>
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

    <!-- 任务地图 -->
    <div class="card mt2">
      <div class="section-title"><span class="bar"></span>任务地图<span class="card-sub">四大模块 · 初/中/高三级关卡 · 点击关卡进入 2D 学习工作台</span></div>
      <div class="task-grid">
        <div
          v-for="t in student.tasks" :key="t.id"
          class="task-card" :class="{ locked: t.status === 'locked' }"
          @click="openTask(t)"
        >
          <span class="tm-lv tag" :class="lvTag(t.level)">{{ t.level }}</span>
          <h4>{{ t.name }}</h4>
          <div class="tm-meta">
            <span class="tag tag-gray">{{ t.module }}</span>
            <span class="tag tag-blue">{{ t.ability }}</span>
          </div>
          <div class="tm-foot">
            <span v-if="t.status === 'done'" class="tm-score">{{ t.score }} 分</span>
            <span v-else-if="t.status === 'active'" class="tag tag-green">进行中</span>
            <span v-else class="tag tag-gray">待解锁</span>
            <span v-if="t.status !== 'locked'" class="tm-go">进入 ›</span>
          </div>
          <div v-if="t.status === 'locked'" class="tm-lk">
            <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2"><rect x="5" y="11" width="14" height="9" rx="2"/><path d="M8 11V8a4 4 0 0 1 8 0v3"/></svg>
            完成前置任务解锁
          </div>
        </div>
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
const LS_STU = "fd_student_v2";

const PRESET_STUDENT = {
  name: "林晓", cls: "财务2433", level: 3, lvName: "财务分析师", xp: 640, xpMax: 1000,
  tasks: [
    { id: "T01", name: "认知利润表", level: "初级", module: "M1", ability: "盈利能力分析", type: "analysis", status: "done", score: 92 },
    { id: "T02", name: "盈利能力建模分析", level: "中级", module: "M1", ability: "盈利能力分析", type: "analysis", status: "done", score: 88 },
    { id: "T03", name: "盈利能力下滑问题追溯与管理建议", level: "高级", module: "M1", ability: "盈利能力分析", type: "analysis", status: "done", score: 90 },
    { id: "T04", name: "认知资产负债表", level: "初级", module: "M2", ability: "财务状况评估", type: "analysis", status: "done", score: 86 },
    { id: "T05", name: "营运能力分析", level: "中级", module: "M2", ability: "财务状况评估", type: "analysis", status: "active", score: null },
    { id: "T06", name: "偿债能力与营运能力综合分析", level: "中级", module: "M2", ability: "财务状况评估", type: "analysis", status: "locked" },
    { id: "T07", name: "偿债能力与营运能力管理建议", level: "高级", module: "M2", ability: "财务状况评估", type: "analysis", status: "locked" },
    { id: "T08", name: "认知现金流量表", level: "初级", module: "M3", ability: "现金流诊断", type: "analysis", status: "locked" },
    { id: "T09", name: "现金流建模分析", level: "中级", module: "M3", ability: "现金流诊断", type: "analysis", status: "locked" },
    { id: "T10", name: "现金流深度分析与财务困境", level: "高级", module: "M3", ability: "现金流诊断", type: "analysis", status: "locked" },
    { id: "T11", name: "撰写财务速览备忘录", level: "初级", module: "M4", ability: "综合财务分析", type: "analysis", status: "locked" },
    { id: "T12", name: "综合财务分析看板设计", level: "中级", module: "M4", ability: "综合财务分析", type: "analysis", status: "locked" },
    { id: "T13", name: "风险洞察与管理建议报告", level: "高级", module: "M4", ability: "综合财务分析", type: "analysis", status: "locked" }
  ],
  currentTask: null,
  lastFeedback: null,
  badges: [
    { n: "数据侦探", d: "发现关键异常指标", earned: true },
    { n: "报表达人", d: "准确解读三大报表", earned: true },
    { n: "现金流卫士", d: "完成现金流诊断任务", earned: false },
    { n: "图表设计师", d: "高质量可视化分析", earned: true },
    { n: "经营参谋", d: "提出有效改进建议", earned: false },
    { n: "财务洞察官", d: "完成综合财务挑战", earned: false }
  ],
  leaderboard: [
    { nm: "王浩", xp: 880, cls: "财务2433" }, { nm: "林晓", xp: 640, cls: "财务2433" },
    { nm: "陈雨", xp: 610, cls: "财务2433" }, { nm: "李娜", xp: 555, cls: "财务2433" },
    { nm: "赵磊", xp: 498, cls: "财务2433" }, { nm: "周婷", xp: 460, cls: "财务2433" }
  ],
  streak: 5, accuracy: 89, lastActive: "今天 09:24",
  recommend: {
    id: "T06", name: "偿债能力与营运能力综合分析", ability: "财务状况评估",
    reason: "你已完成营运能力分析（T05），下一步建议做偿债×营运能力交叉验证，补齐「财务状况评估」短板后进入 M3 现金流诊断。"
  },
  abilityProg: [
    { k: "盈利能力分析", v: 88 }, { k: "财务状况评估", v: 78 },
    { k: "现金流诊断", v: 66 }, { k: "综合财务分析", v: 60 }
  ],
  abilityRadar: {
    axes: ["盈利能力分析", "财务状况评估", "现金流诊断", "综合财务分析"],
    cur: [88, 78, 66, 60], avg: [82, 74, 70, 62]
  },
  ethicsScenes: [
    { topic: "数据真实性", kp: "数据真实性核验", theme: "数据诚信", title: "数字背后的责任",
      scene: "你在录入某上市公司三季度营收数据时发现：把一笔 1,200 万的收入错计到下季度，本月营收指标恰好「达标」。系统提示「数据异常波动」，你是否修正？",
      guide: ["若小数点或归属期错配，可能误导怎样的经营决策？", "数据真实为什么是财务人的第一底线？", "遇到「达标」诱惑时，应如何守住职业底线？"],
      case: "《会计法》明确要求会计核算以实际发生的经济业务为依据——虚构、提前确认收入均属违法。" }
  ],
  feed: [
    { icon: "check", txt: "完成「认知资产负债表」，得分 86", time: "2 天前" },
    { icon: "star", txt: "解锁「报表达人」徽章", time: "3 天前" },
    { icon: "bot", txt: "AI 导师为你推荐「偿债能力与营运能力综合分析」", time: "今天" }
  ]
};

function loadStudent() {
  const base = JSON.parse(JSON.stringify(PRESET_STUDENT));
  const raw = lsGet(LS_STU) || {};
  const s = Object.assign({}, base, raw);
  // 班级已统一为财务2433：学生身份与排行榜成员一并归一；称号随版本统一覆盖，避免旧缓存残留
  s.cls = base.cls;
  s.lvName = base.lvName;
  if (!Array.isArray(s.leaderboard) || !s.leaderboard.length) s.leaderboard = base.leaderboard;
  s.leaderboard = s.leaderboard.map(function (p, i) {
    return Object.assign({}, base.leaderboard[i] || { cls: base.cls }, p, { cls: base.cls });
  });
  if (!s.tasks) s.tasks = base.tasks;
  if (!s.badges) s.badges = base.badges;
  if (!s.ethicsScenes) s.ethicsScenes = base.ethicsScenes;
  // 动态为静态演示数据，随预置统一覆盖，避免旧缓存残留已下线条目（如班级排名）
  s.feed = base.feed;
  lsSet(LS_STU, s);
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

// ============ 页面跳转（低代码平台多 Tab 菜单，无刷新切换） ============
const VIEW_URL = {
  "student-task": "https://1040053947.yxd.ep.caih.com/apps/desktop/sapp/app_5ex10jl1o4/sapp_qguqtkatxm/box/form/form_ix7gbehpyi/0859277446244543?menuId=0859277446244543&resource_type=FORM"
};
// 三智能体卡片 → 平台菜单名（评学档案兼容旧菜单名“评学增值”）
const TAB_OF = {
  "student-task": ["技能训练"],
  "student-ethics": ["立德润心"],
  "student-growth": ["评学档案", "评学增值"]
};
function clickMenu(menuName) {
  // 顶部内部 Tab 栏（inner-tabbar 下的 .tab-box）
  const boxes = document.querySelectorAll('.inner-tabbar-list .tab-box, .tab-box');
  for (let i = 0; i < boxes.length; i++) {
    if ((boxes[i].textContent || "").indexOf(menuName) >= 0) { boxes[i].click(); return true; }
  }
  // 侧边菜单项
  const items = document.querySelectorAll('.menu-box [class*="menu-item"], [class*="menu-item"]');
  for (let i = 0; i < items.length; i++) {
    if ((items[i].textContent || "").indexOf(menuName) >= 0) { items[i].click(); return true; }
  }
  return false;
}

// ============ 交互 ============
function go(view) {
  // ① 优先：模拟点击平台对应 Tab / 菜单项（无刷新）
  const names = TAB_OF[view] || [];
  for (let i = 0; i < names.length; i++) { if (clickMenu(names[i])) return; }

  const url = VIEW_URL[view];
  if (!url) { emit("navigate", view); return; }

  // ② 其次：平台 Vue Router（无刷新，仅改地址）
  try {
    const app = document.querySelector("#app");
    const vueApp = app && app.__vue_app__;
    const router = vueApp && vueApp.config && vueApp.config.globalProperties && vueApp.config.globalProperties.$router;
    if (router && typeof router.push === "function") {
      const tgt = new URL(url, location.origin);
      let path = tgt.pathname;
      // 去掉 router 自身的 base 前缀，避免 push 时重复拼接（如 /apps/desktop 重复）
      try {
        const base = router.options && router.options.history && router.options.history.base;
        if (base && base !== "/" && path.indexOf(base) === 0) {
          path = path.slice(base.length);
          if (path.charAt(0) !== "/") path = "/" + path;
        }
      } catch (e2) { /* 读不到 base 则原样传入 */ }
      router.push(path + tgt.search);
      return;
    }
  } catch (e) { /* 忽略，继续下一级 */ }

  // ③ 再次：浏览器 Navigation API（无刷新 SPA 导航）
  try {
    if (window.navigation && typeof window.navigation.navigate === "function") {
      window.navigation.navigate(url);
      return;
    }
  } catch (e) { /* 忽略，继续下一级 */ }

  // ④ 兜底：整页跳转
  window.location.href = url;
}
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
  showTip("已进入技能训练，可在右侧向 AI 导师提问");
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
.hero-lv { display: inline-flex; align-items: center; margin-left: auto; font-size: 12px; font-weight: 600; background: rgba(13, 48, 110, .45); border: 1px solid rgba(255, 255, 255, .35); padding: 2px 10px; border-radius: 99px; }
.hero-flame { display: inline-flex; align-items: center; font-size: 12px; font-weight: 600; background: rgba(255, 255, 255, .22); padding: 2px 10px; border-radius: 99px; }
.xp-wrap { margin-bottom: 14px; }
.xp-bar { width: 100%; height: 10px; background: rgba(255, 255, 255, .28); border-radius: 99px; overflow: hidden; }
.xp-fill { height: 100%; background: linear-gradient(90deg, #FA8C16, #FFD591); border-radius: 99px; transition: width .5s ease; }
.hero-actions { display: flex; align-items: center; gap: 12px; }
.hero-hint { font-size: 13px; opacity: .92; }

/* ============ 三智能体协同（渐变卡片，可点击跳转） ============ */
.agent-card { border: 2px dashed #86B0E8; background: #F8FBFF; }
.agent-row { display: flex; flex-wrap: wrap; gap: 12px; }
.agent-chip { position: relative; display: flex; align-items: center; gap: 10px; padding: 14px 14px 12px; border-radius: 10px; color: #fff; flex: 1; min-width: 170px; cursor: pointer; overflow: hidden; box-shadow: 0 2px 6px rgba(16, 38, 76, .12); transition: transform .18s ease, box-shadow .18s ease; }
.agent-chip:hover { transform: translateY(-3px); box-shadow: 0 8px 18px rgba(16, 38, 76, .22); }
.agent-chip::after { content: ""; position: absolute; right: -20px; top: -20px; width: 70px; height: 70px; border-radius: 50%; background: rgba(255, 255, 255, .14); }
.chip-ideo-new { background: linear-gradient(135deg, #FF7B7B, #E54C4C); }
.chip-tech-new { background: linear-gradient(135deg, #5B9BF0, #2B6CD6); }
.chip-grow-new { background: linear-gradient(135deg, #7BD64C, #4CAF50); }
.agent-ic { width: 34px; height: 34px; border-radius: 9px; background: rgba(255, 255, 255, .22); display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.agent-ic svg { width: 18px; height: 18px; }
.agent-txt { flex: 1; min-width: 0; }
.agent-txt b { display: block; font-size: 14px; line-height: 1.3; }
.agent-txt small { display: block; font-size: 11px; opacity: .85; margin-top: 2px; }
.agent-go { position: relative; z-index: 1; font-size: 12px; font-weight: 600; background: rgba(255, 255, 255, .22); padding: 2px 9px; border-radius: 99px; white-space: nowrap; }
.agent-desc { color: #52A443; font-size: 14px; font-weight: 600; line-height: 22px; }

/* ============ 任务地图 ============ */
.card-sub { margin-left: auto; font-size: 12px; font-weight: 400; color: #97A1B2; }
.task-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 14px; }
.task-card { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px; transition: all .15s ease; position: relative; overflow: hidden; cursor: pointer; }
.task-card:hover { border-color: #C7D6EC; box-shadow: 0 2px 8px rgba(16, 38, 76, .08); }
.task-card.locked { opacity: .62; background: #F7FAFD; }
.tm-lv { position: absolute; top: 0; right: 0; padding: 3px 10px; border-bottom-left-radius: 8px; }
.task-card h4 { font-size: 14px; font-weight: 600; color: #1F2733; margin: 6px 0 8px; padding-right: 54px; line-height: 1.5; }
.tm-meta { display: flex; gap: 6px; flex-wrap: wrap; margin-bottom: 10px; }
.tm-foot { display: flex; align-items: center; justify-content: space-between; margin-top: 8px; }
.tm-score { font-weight: 600; color: #3C8E12; font-size: 13px; }
.tm-go { font-size: 12px; color: #97A1B2; }
.tm-lk { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; gap: 5px; color: #97A1B2; font-size: 13px; background: rgba(247, 250, 253, .45); }

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
