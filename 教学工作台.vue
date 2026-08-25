<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">教学工作台</span>
      <span class="page-head-sub">财数智析 · 教学管理（教师端）</span>
      <button class="btn-ghost" @click="refresh">刷新数据</button>
    </div>

    <!-- Hero 横幅 -->
    <div class="hero">
      <div class="hero-line">
        <span class="hero-hello">{{ teacher.name }}，上午好</span>
        <span class="tag tag-light">{{ teacher.cls }}</span>
        <span class="hero-flame">待复核 {{ pending.length }} 份 · 需重点关注 {{ teacher.needsFocus.length }} 人</span>
      </div>
      <div class="hero-actions">
        <button class="btn btn-light btn-sm" @click="go('teacher-tasks')">发布任务</button>
        <button class="btn btn-light btn-sm" @click="go('teacher-analytics')">学情分析</button>
        <button class="btn btn-light btn-sm" @click="go('teacher-review')">评分复核</button>
      </div>
    </div>

    <!-- 统计卡 -->
    <div class="kpi-row six mt">
      <div class="kpi"><div class="k">任务完成率</div><div class="v c-blue">{{ c.completion }}<small>%</small></div></div>
      <div class="kpi"><div class="k">平均得分</div><div class="v">{{ c.avg }}<small> / 100</small></div></div>
      <div class="kpi"><div class="k">活跃人数</div><div class="v c-green">{{ c.active }}<small> / {{ c.total }}</small></div></div>
      <div class="kpi"><div class="k">待复核</div><div class="v c-orange">{{ pending.length }}<small> 份</small></div></div>
      <div class="kpi"><div class="k">本周提交</div><div class="v c-green">{{ teacher.weeklySubs }}<small> 份</small></div></div>
      <div class="kpi"><div class="k">需关注</div><div class="v c-red">{{ teacher.needsFocus.length }}<small> 人</small></div></div>
    </div>

    <!-- 任务完成度分布 + 待复核任务 -->
    <div class="grid-2 mt">
      <div class="card">
        <div class="section-title"><span class="bar"></span>任务完成度分布</div>
        <div v-for="b in completionBars" :key="b[0]" class="barH-item">
          <div class="barH-lab"><span>{{ b[0] }}</span><span class="muted">{{ b[1] }}%</span></div>
          <div class="bar-bg tall"><div class="bar-fill f-blue" :style="{ width: b[1] + '%' }"></div></div>
        </div>
      </div>
      <div class="card">
        <div class="section-title">
          <span class="bar"></span>待复核任务
          <span class="card-sub">点击进入复核</span>
        </div>
        <template v-if="pending.length">
          <div v-for="s in pending" :key="s.id" class="pd-row" @click="go('teacher-review')">
            <div>
              <div class="pd-name">{{ s.student }}</div>
              <div class="hint">{{ s.task }}</div>
            </div>
            <button class="btn btn-default btn-sm" @click.stop="go('teacher-review')">复核</button>
          </div>
        </template>
        <div v-else class="empty">全部复核完成</div>
      </div>
    </div>

    <!-- 薄弱能力预警 + 高频错因 Top3 -->
    <div class="grid-2 mt">
      <div class="card">
        <div class="section-title">
          <span class="bar"></span>薄弱能力预警
          <span class="card-sub">班级均值最低维度</span>
        </div>
        <div v-for="w in weakDims" :key="w.k" class="weak">
          <div class="ability-row"><span>{{ w.k }}</span><span class="av">{{ w.v }}</span></div>
          <div class="bar-bg tall"><div class="bar-fill f-orange" :style="{ width: w.v + '%' }"></div></div>
        </div>
        <div class="home-tip">建议下节课重点讲解「{{ weakDims[0].k }}」与「{{ weakDims[1].k }}」。</div>
      </div>
      <div class="card">
        <div class="section-title"><span class="bar"></span>高频错因 Top3</div>
        <div v-for="e in topErrors" :key="e.k" class="err-row">
          <span class="err-k">{{ e.k }}</span>
          <div class="bar-bg tall err-bar"><div class="bar-fill f-red" :style="{ width: (e.v / topErrors[0].v * 100) + '%' }"></div></div>
          <span class="err-v">{{ e.v }}</span>
        </div>
        <div class="home-tip">错因数据由 AI 评分自动归集，用于精准讲评。</div>
      </div>
    </div>

    <!-- 重点关注学生 + 教学动态 -->
    <div class="grid-2 mt">
      <div class="card">
        <div class="section-title"><span class="bar"></span>重点关注学生</div>
        <div v-for="f in teacher.needsFocus" :key="f.nm" class="focus-item">
          <span class="f-av">{{ f.nm[0] }}</span>
          <div class="f-body">
            <div class="f-nm">
              {{ f.nm }}
              <span class="tag" :class="f.status === '待复核' ? 'tag-red' : 'tag-gray'">{{ f.status }}</span>
            </div>
            <div class="f-issue">{{ f.issue }}</div>
          </div>
          <span class="focus-score" :class="focusClass(f)">{{ f.score == null ? '—' : f.score }}</span>
        </div>
      </div>
      <div class="card">
        <div class="section-title"><span class="bar"></span>教学动态</div>
        <div v-for="(a, i) in teacher.activity" :key="i" class="feed-item">
          <span class="feed-dot" :class="feedCls(a.icon)"></span>
          <div class="feed-body">
            <div class="feed-txt">{{ a.txt }}</div>
            <div class="feed-time">{{ a.time }}</div>
          </div>
        </div>
      </div>
    </div>

    <!-- AI 教学建议 -->
    <div class="card mt">
      <div class="section-title">
        <span class="bar"></span>AI 教学建议
        <span class="card-sub">增值 Agent · 可追问干预策略</span>
      </div>
      <p class="ai-advice">{{ aiAdvice }}</p>
      <div class="advice-meta">
        <div class="am-item"><div class="am-k">讲评重点</div><div class="am-v">{{ topErrors[0].k }}</div></div>
        <div class="am-item"><div class="am-k">薄弱维度</div><div class="am-v">{{ weakDims[0].k }} · {{ weakDims[1].k }}</div></div>
        <div class="am-item"><div class="am-k">优先辅导</div><div class="am-v">{{ teacher.needsFocus[0].nm }} 等 {{ teacher.needsFocus.length }} 人</div></div>
      </div>
      <button class="btn btn-ghost btn-sm mt8" @click="askAI">AI 生成教学建议</button>
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
const LS_TEACHER = "fd_teacher_v1";

// ============ AI 占位（接入后替换 AI_KEY） ============
const AI_URL = "https://agent.gjt-smart.com/v1/chat-messages";
const AI_KEY = "app-xxxxxx"; // TODO 占位

const PRESET_TEACHER = {
  name: "陈老师",
  cls: "财管2201",
  classStats: { completion: 82, avg: 79, active: 31, total: 38 },
  weeklySubs: 16,
  completionBars: [
    ["T03 毛利率分析", 93], ["T04 费用排查", 88], ["T08 驾驶舱诊断", 75]
  ],
  tasks: [
    { id: "T03", name: "毛利率异常分析与费用率排查", level: "初级", diff: "易", ability: "指标分析", status: "已发布", subs: 28, skills: ["毛利率分析", "费用率分析"], ideos: ["行业伦理"] },
    { id: "T04", name: "多维度经营解析（产品/地区/渠道）", level: "中级", diff: "中", ability: "多维分析", status: "已发布", subs: 21, skills: ["常规经营解析"], ideos: ["课程思政教学案例"] },
    { id: "T07", name: "现金流分析与利润质量评价", level: "中级", diff: "中", ability: "现金流分析", status: "草稿", subs: 0, skills: ["现金流分析", "利润质量评价"], ideos: [] },
    { id: "T08", name: "偿债能力分层与风险预警诊断", level: "高级", diff: "难", ability: "风险特警", status: "已发布", subs: 12, skills: ["风险预警"], ideos: ["政策文件"] }
  ],
  submissions: [
    { id: 1, student: "王浩", task: "T03 毛利率异常分析", ai: 76, status: "待复核", errors: ["结论缺少业务逻辑", "图表选择不当"] },
    { id: 2, student: "陈雨", task: "T03 毛利率异常分析", ai: 91, status: "已复核", errors: [] },
    { id: 3, student: "李娜", task: "T04 多维度经营解析", ai: 68, status: "待复核", errors: ["公式错误", "指标理解错误"] },
    { id: 4, student: "赵磊", task: "T08 风险预警诊断", ai: 83, status: "待复核", errors: ["流动比率计算口径不一致"] }
  ],
  errorDist: [
    { k: "公式错误", v: 18 }, { k: "指标理解错误", v: 26 }, { k: "图表选择不当", v: 14 },
    { k: "结论缺业务逻辑", v: 22 }, { k: "分析不全面", v: 11 }
  ],
  radar: {
    axes: ["财报数据治理", "经营解析能力", "异常特征识别", "现金流分析", "风险预警能力", "可视化建模"],
    cur: [82, 76, 70, 64, 88, 58], avg: [75, 72, 68, 70, 74, 66]
  },
  needsFocus: [
    { nm: "李娜", score: 68, status: "待复核", issue: "T04 多维解析公式错误 + 指标理解偏差，建议一对一辅导" },
    { nm: "赵磊", score: 83, status: "待复核", issue: "T08 流动比率计算口径不一致，需确认是否剔除预付款项" },
    { nm: "周婷", score: null, status: "未提交", issue: "本周未提交任何任务，活跃度明显下降，需提醒" }
  ],
  activity: [
    { icon: "upload", txt: "李娜 提交了「T04 多维度经营解析」", time: "10 分钟前" },
    { icon: "check", txt: "陈雨 的「T03 毛利率异常分析」复核通过", time: "1 小时前" },
    { icon: "bot", txt: "AI 完成 4 份作业自动评分", time: "今天" },
    { icon: "warn", txt: "赵磊「T08 风险预警诊断」触发薄弱能力预警", time: "昨天" }
  ]
};

function loadTeacher() {
  let t = lsGet(LS_TEACHER);
  if (!t) { t = JSON.parse(JSON.stringify(PRESET_TEACHER)); lsSet(LS_TEACHER, t); }
  return t;
}

// ============ 页面状态 ============
const teacher = ref(loadTeacher());
const tip = ref("");
let tipTimer = null;
function showTip(t) { tip.value = t; clearTimeout(tipTimer); tipTimer = setTimeout(function () { tip.value = ""; }, 2400); }

function refresh() { teacher.value = loadTeacher(); }

// ============ 计算属性 ============
const c = computed(function () { return teacher.value.classStats; });
const pending = computed(function () { return teacher.value.submissions.filter(function (s) { return s.status === "待复核"; }); });
const completionBars = computed(function () { return teacher.value.completionBars; });
const weakDims = computed(function () {
  return teacher.value.radar.axes
    .map(function (k, i) { return { k: k, v: teacher.value.radar.avg[i] }; })
    .sort(function (a, b) { return a.v - b.v; })
    .slice(0, 3);
});
const topErrors = computed(function () { return [].concat(teacher.value.errorDist).sort(function (a, b) { return b.v - a.v; }).slice(0, 3); });
const aiAdvice = computed(function () {
  return "本周班级平均得分 " + c.value.avg + "，待复核 " + pending.value.length + " 份。薄弱点集中在「" + weakDims.value[0].k + "」「" + weakDims.value[1].k + "」，建议在下次课用真实业务案例串讲" + weakDims.value[0].k + "方法；同时优先辅导 " + teacher.value.needsFocus[0].nm + " 等 " + teacher.value.needsFocus.length + " 名需关注学生，可将高频错因「" + topErrors.value[0].k + "」作为讲评重点。";
});

// ============ 平台页面跳转（无刷新，等同点击平台 Tab/菜单） ============
const MENU_OF = {
  "student-home": "学习主页", "student-task": "任务工作台", "student-ethics": "职业伦理情境", "student-feedback": "学习反馈",
  "teacher-tasks": "任务管理", "teacher-analytics": "学情分析", "teacher-review": "评分复核"
};
const URL_OF = {
  "student-home": "https://1040053947.yxd.ep.caih.com/apps/desktop/sapp/app_5ex10jl1o4/sapp_qguqtkatxm/box/form/form_udp0sjx0td/4328371521401607?menuId=4328371521401607&resource_type=FORM",
  "student-task": "https://1040053947.yxd.ep.caih.com/apps/desktop/sapp/app_5ex10jl1o4/sapp_qguqtkatxm/box/form/form_ix7gbehpyi/0859277446244543?menuId=0859277446244543&resource_type=FORM"
};
function clickMenu(menuName) {
  const boxes = document.querySelectorAll('.inner-tabbar-list .tab-box, .tab-box');
  for (let i = 0; i < boxes.length; i++) { if ((boxes[i].textContent || "").indexOf(menuName) >= 0) { boxes[i].click(); return true; } }
  const items = document.querySelectorAll('.menu-box [class*="menu-item"], [class*="menu-item"]');
  for (let i = 0; i < items.length; i++) { if ((items[i].textContent || "").indexOf(menuName) >= 0) { items[i].click(); return true; } }
  return false;
}
function go(view) {
  const menuName = MENU_OF[view];
  if (menuName && clickMenu(menuName)) return;
  const url = URL_OF[view];
  if (url) {
    try {
      const app = document.querySelector("#app");
      const vueApp = app && app.__vue_app__;
      const router = vueApp && vueApp.config && vueApp.config.globalProperties && vueApp.config.globalProperties.$router;
      if (router && typeof router.push === "function") {
        const tgt = new URL(url, location.origin);
        let path = tgt.pathname;
        try {
          const base = router.options && router.options.history && router.options.history.base;
          if (base && base !== "/" && path.indexOf(base) === 0) { path = path.slice(base.length); if (path.charAt(0) !== "/") path = "/" + path; }
        } catch (e2) { /* 读不到 base 则原样传入 */ }
        router.push(path + tgt.search);
        return;
      }
    } catch (e) { /* 忽略，继续下一级 */ }
    try { if (window.navigation && typeof window.navigation.navigate === "function") { window.navigation.navigate(url); return; } } catch (e) { /* 忽略 */ }
    window.location.href = url;
    return;
  }
  emit("navigate", view);
}
function focusClass(f) { if (f.score == null) return ""; return f.score < 70 ? "low" : "mid"; }
function feedCls(k) { return { upload: "d-green", check: "d-blue", bot: "d-blue", warn: "d-orange" }[k] || "d-blue"; }
function askAI() { showTip("AI 功能待接入：请在 AI_KEY 填入智能体 API Key"); }

onMounted(function () { });
</script>

<style scoped>
/* ============ 基线（对齐学习主页 / Ant 风格） ============ */
.pg { max-width: 1100px; margin: 0 auto; padding: 18px 16px 40px; }
.page-head { display: flex; align-items: center; gap: 10px; margin-bottom: 16px; flex-wrap: wrap; }
.page-head-bar { width: 4px; height: 22px; border-radius: 2px; background: #2B6CD6; }
.page-head-title { font-size: 20px; font-weight: 600; color: #1F2733; }
.page-head-sub { font-size: 13px; color: #97A1B2; flex: 1; }
.btn { display: inline-flex; align-items: center; gap: 6px; height: 34px; padding: 0 16px; border-radius: 6px; font-size: 14px; font-weight: 600; border: 1px solid transparent; cursor: pointer; transition: all .15s ease; white-space: nowrap; }
.btn:disabled { opacity: .5; cursor: not-allowed; }
.btn-primary { background: #2B6CD6; color: #fff; }
.btn-primary:hover { background: #4D8BE8; }
.btn-default { background: #fff; border-color: #E3E9F2; color: #1F2733; }
.btn-default:hover { border-color: #2B6CD6; color: #2B6CD6; }
.btn-ghost { color: #2B6CD6; background: #EAF2FC; }
.btn-ghost:hover { background: #DCEBFB; }
.btn-light { background: #fff; color: #2B6CD6; }
.btn-sm { height: 28px; padding: 0 12px; font-size: 13px; }
.mt { margin-top: 16px; }
.mt8 { margin-top: 8px; }
.muted { color: #5A6577; font-size: 13px; }
.hint { font-size: 12px; color: #97A1B2; }
.card { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px; box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); }
.section-title { display: flex; align-items: center; gap: 8px; font-size: 15px; font-weight: 600; color: #1F2733; margin-bottom: 14px; }
.section-title .bar { width: 4px; height: 15px; border-radius: 2px; background: #2B6CD6; }
.section-title .card-sub { margin-left: auto; }
.grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
.kpi-row { display: grid; grid-template-columns: repeat(5, 1fr); gap: 12px; }
.kpi-row.six { grid-template-columns: repeat(6, 1fr); }
.kpi { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 13px 15px; }
.kpi .k { font-size: 12px; color: #5A6577; }
.kpi .v { font-size: 21px; font-weight: 600; margin-top: 2px; color: #1F2733; }
.kpi .v small { font-size: 12px; color: #97A1B2; font-weight: 400; }
.c-orange { color: #FA8C16 !important; }
.c-blue { color: #2B6CD6 !important; }
.c-green { color: #52C41A !important; }
.c-red { color: #FF4D4F !important; }
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
.empty { padding: 24px 0; text-align: center; color: #97A1B2; font-size: 13px; }
.tip { margin-top: 12px; background: #EAF2FC; border: 1px solid #DCEBFB; color: #1D4FA8; font-size: 13px; border-radius: 6px; padding: 8px 12px; }
@media (max-width: 900px) { .grid-2 { grid-template-columns: 1fr; } .kpi-row, .kpi-row.six { grid-template-columns: repeat(2, 1fr); } }

/* ============ Hero 横幅 ============ */
.hero { background: linear-gradient(135deg, #2B6CD6 0%, #4D8BE8 60%, #6FA3F0 100%); border-radius: 10px; padding: 20px 22px 16px; color: #fff; box-shadow: 0 4px 14px rgba(43, 108, 214, .25); }
.hero-line { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; margin-bottom: 14px; }
.hero-hello { font-size: 20px; font-weight: 700; }
.hero-flame { display: inline-flex; align-items: center; font-size: 12px; font-weight: 600; background: rgba(255, 255, 255, .22); padding: 2px 10px; border-radius: 99px; }
.hero-actions { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }

/* ============ 任务完成度 / 待复核 ============ */
.card-sub { font-size: 12px; font-weight: 400; color: #97A1B2; }
.home-tip { font-size: 12px; color: #97A1B2; margin-top: 10px; }
.barH-item { margin-bottom: 12px; }
.barH-item:last-child { margin-bottom: 0; }
.barH-lab { display: flex; justify-content: space-between; font-size: 13px; margin-bottom: 4px; }
.pd-row { display: flex; justify-content: space-between; align-items: center; padding: 9px 0; border-bottom: 1px solid #EEF2F8; cursor: pointer; transition: background .15s; }
.pd-row:hover { background: #F7FAFD; }
.pd-row:last-child { border-bottom: none; }
.pd-name { font-weight: 600; font-size: 13px; color: #1F2733; }

/* ============ 薄弱能力预警 / 高频错因 ============ */
.weak { margin-bottom: 11px; }
.weak:last-of-type { margin-bottom: 0; }
.ability-row { display: flex; justify-content: space-between; font-size: 13px; margin-bottom: 5px; }
.ability-row .av { color: #5A6577; font-variant-numeric: tabular-nums; }
.err-row { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; }
.err-row:last-of-type { margin-bottom: 0; }
.err-k { width: 120px; font-size: 13px; flex-shrink: 0; }
.err-bar { flex: 1; }
.err-v { width: 30px; text-align: right; font-size: 13px; color: #5A6577; flex-shrink: 0; }

/* ============ 重点关注学生 / 教学动态 ============ */
.focus-item { display: flex; gap: 11px; align-items: flex-start; padding: 10px 2px; border-bottom: 1px solid #EEF2F8; }
.focus-item:last-child { border-bottom: none; }
.f-av { width: 30px; height: 30px; border-radius: 50%; flex-shrink: 0; background: #EAF2FC; color: #2B6CD6; display: flex; align-items: center; justify-content: center; font-weight: 600; font-size: 13px; }
.f-body { flex: 1; min-width: 0; }
.f-nm { font-weight: 600; font-size: 13px; display: flex; align-items: center; gap: 6px; }
.f-issue { font-size: 12px; color: #5A6577; margin-top: 3px; line-height: 1.5; }
.focus-score { margin-left: auto; font-weight: 700; font-size: 15px; white-space: nowrap; }
.focus-score.low { color: #FF4D4F; }
.focus-score.mid { color: #FAAD14; }
.feed-item { display: flex; gap: 10px; padding: 8px 0; border-bottom: 1px solid #EEF2F8; }
.feed-item:last-child { border-bottom: none; }
.feed-dot { width: 8px; height: 8px; border-radius: 50%; margin-top: 6px; flex: none; }
.d-green { background: #52C41A; }
.d-orange { background: #FA8C16; }
.d-gold { background: #FAAD14; }
.d-blue { background: #2B6CD6; }
.feed-body { flex: 1; min-width: 0; }
.feed-txt { font-size: 13px; color: #1F2733; line-height: 20px; }
.feed-time { font-size: 12px; color: #97A1B2; margin-top: 2px; }

/* ============ AI 教学建议 ============ */
.ai-advice { font-size: 13px; color: #1F2733; line-height: 21px; background: #F7FAFD; border-radius: 6px; padding: 10px 12px; }
.advice-meta { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; margin-top: 12px; }
.am-item { background: #F7FAFD; border-radius: 6px; padding: 10px 12px; }
.am-k { font-size: 12px; color: #97A1B2; }
.am-v { font-size: 13px; font-weight: 600; color: #1D4FA8; margin-top: 3px; }
@media (max-width: 700px) { .advice-meta { grid-template-columns: 1fr; } }
</style>
