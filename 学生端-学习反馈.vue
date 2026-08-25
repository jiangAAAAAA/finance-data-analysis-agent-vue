<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">学习反馈</span>
      <span class="page-head-sub">本次任务自动评分与 AI 复盘 · 增值 Agent</span>
    </div>

    <!-- 能力雷达 -->
    <div class="card">
      <div class="section-title"><span class="bar"></span>能力雷达 · 多维画像<span class="card-sub">评学增值 · 增值 Agent</span></div>
      <svg class="svg-chart" viewBox="0 0 260 220" role="img">
        <polygon v-for="g in grids" :key="g" :points="ringPoints(g)" fill="none" stroke="#EEF2F8" />
        <g v-for="(a, i) in radar.axes" :key="'ax' + i">
          <line :x1="RADAR.cx" :y1="RADAR.cy" :x2="axisEnd(i)[0]" :y2="axisEnd(i)[1]" stroke="#EEF2F8" />
          <text :x="axisEnd(i)[0]" :y="axisEnd(i)[1] + (axisEnd(i)[1] > RADAR.cy ? 12 : -4)" font-size="10" fill="#5A6577" text-anchor="middle">{{ a }}</text>
        </g>
        <polygon :points="poly(radar.avg)" fill="rgba(151,161,178,.15)" stroke="#97A1B2" stroke-width="1.5" />
        <polygon :points="poly(radar.cur)" fill="rgba(43,108,214,.16)" stroke="#2B6CD6" stroke-width="2" />
        <circle v-for="(p, i) in curPoints" :key="'d' + i" :cx="p[0]" :cy="p[1]" r="2.5" fill="#2B6CD6" />
      </svg>
      <div class="legend">
        <span><i class="lg-me"></i>林晓</span>
        <span><i class="lg-avg"></i>班级均值</span>
      </div>
    </div>

    <!-- 得分环 + 主内容 -->
    <div class="fb-layout mt">
      <div class="card score-card">
        <svg viewBox="0 0 96 96" class="score-ring" role="img">
          <circle cx="48" cy="48" r="42" fill="none" stroke="#EEF2F8" stroke-width="8" />
          <circle cx="48" cy="48" r="42" fill="none" stroke="#2B6CD6" stroke-width="8"
            stroke-linecap="round" :stroke-dasharray="ringC" :stroke-dashoffset="ringOffset"
            transform="rotate(-90 48 48)" style="transition: stroke-dashoffset .8s ease" />
          <text x="48" y="54" font-size="22" font-weight="600" fill="#1F2733" text-anchor="middle">{{ totalScore }}</text>
        </svg>
        <div class="score-label">综合得分 {{ totalScore }}</div>
        <span class="tag tag-orange score-tag">获得 +{{ fb.xpGain }} 经验 · 徽章待解锁</span>
      </div>

      <div class="fb-main">
        <div class="card">
          <div class="section-title"><span class="bar"></span>分项评分</div>
          <div class="fb-grid">
            <div class="fb-item" v-for="d in fb.dims" :key="d.label">
              <div class="fk">{{ d.label }} <span class="muted">({{ d.full }}分)</span></div>
              <div class="fv">{{ d.got }}<small> / {{ d.full }}</small></div>
            </div>
          </div>
        </div>

        <div class="card mt" v-if="fb.conclusion">
          <div class="section-title"><span class="bar"></span>你的结论</div>
          <div class="fb-echo"><b>核心结论：</b>{{ fb.conclusion.core }}</div>
          <div class="fb-echo"><b>数据支撑：</b>{{ fb.conclusion.evidence }}</div>
          <div class="fb-echo"><b>改进建议：</b>{{ fb.conclusion.adv }}</div>
        </div>

        <div class="card mt" v-if="fb.overview">
          <div class="section-title"><span class="bar"></span>{{ fb.type === 'quiz' ? '作答概览' : '测算概览' }}</div>
          <div class="fb-echo">{{ fb.overview }}</div>
          <ul class="ti-list" v-if="fb.overviewRows" style="margin-top:8px">
            <li v-for="r in fb.overviewRows" :key="r.label">
              {{ r.label }}：{{ r.ok ? '✓ 回答正确' : '✗ 回答错误（正确答案 ' + r.answer + '）' }}
            </li>
          </ul>
          <div class="scroll-x" v-if="fb.calcRows" style="margin-top:8px">
            <table class="data-table">
              <thead><tr><th style="text-align:left">指标</th><th>我的结果</th><th>标准值</th><th>判定</th></tr></thead>
              <tbody>
                <tr v-for="r in fb.calcRows" :key="r.label">
                  <td style="text-align:left">{{ r.label }}</td>
                  <td>{{ r.got || '—' }}</td><td>{{ r.target }}</td>
                  <td :style="{ color: r.good ? '#52C41A' : '#FF4D4F' }">
                    {{ r.good ? '✓ 正确' : '✗ 偏差' }}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <div class="card mt">
          <div class="section-title"><span class="bar"></span>错因分析</div>
          <span v-if="fb.errTags.length" class="tag tag-red err-tag" v-for="e in fb.errTags" :key="e">{{ e }}</span>
          <span v-else class="tag tag-green">全部正确，无显著错因</span>
          <p class="err-note">{{ fb.errNote }}</p>
        </div>

        <div class="card mt">
          <div class="section-title"><span class="bar"></span>AI 评语与下一步<span class="card-sub">增值 Agent 生成</span></div>
          <p class="err-note">{{ fb.aiComment }}</p>
          <div class="fb-actions">
            <button class="btn btn-ghost" @click="aiComment">生成 AI 评语</button>
            <button class="btn btn-primary" @click="go('student-home')">返回任务地图</button>
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

// ============ AI 占位配置 ============
const AI_URL = "https://agent.gjt-smart.com/v1/chat-messages";
const AI_KEY = "app-xxxxxx"; // TODO 占位

// ============ 数据服务（localStorage 共享，PRESET 兜底） ============
function lsGet(k) { try { const v = localStorage.getItem(k); return v ? JSON.parse(v) : null; } catch (e) { return null; } }
function lsSet(k, v) { try { localStorage.setItem(k, JSON.stringify(v)); } catch (e) { /* 忽略 */ } }
const LS_STU = "fd_student_v1";

const PRESET_STUDENT = {
  name: "林晓", cls: "财务2433", level: 3, lvName: "数据分析员", xp: 640, xpMax: 1000,
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
    { nm: "王浩", xp: 880, cls: "财务2433" }, { nm: "林晓", xp: 640, cls: "财务2433" },
    { nm: "陈雨", xp: 610, cls: "财务2433" }, { nm: "李娜", xp: 555, cls: "财务2433" },
    { nm: "赵磊", xp: 498, cls: "财务2433" }, { nm: "周婷", xp: 460, cls: "财务2433" }
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
      case: "《会计法》明确要求会计核算以实际发生的经济业务为依据——虚构、提前确认收入均属违法。" },
    { topic: "财务造假", kp: "利润质量评价", theme: "行业伦理", title: "瑞幸式陷阱",
      scene: "你正在评价某公司利润质量，发现其净利润连年增长但经营现金流持续为负、应收账款激增。导师提示：这可能是「纸面富贵」。",
      guide: ["利润与现金流背离意味着什么？", "收入确认激进可能踩中哪条监管红线？", "如何从财报勾稽关系识别财务造假迹象？"],
      case: "瑞幸咖啡虚构 22 亿元交易额被重罚——财务造假的代价远超短期股价收益。" },
    { topic: "盈余管理", kp: "费用率分析", theme: "职业判断", title: "费用的「挪移」",
      scene: "季度末，部门为完成利润目标建议你将一笔市场费用顺延至下季度确认。报表会「更好看」，但这是否触碰合规边界？",
      guide: ["费用跨期确认与财务造假的法律边界在哪？", "盈余管理与企业长期价值的冲突？", "如果你是财务总监，会怎么做？"],
      case: "某企业财务主管因通过费用跨期进行盈余管理被处罚——真实代价远比想象更大。" },
    { topic: "数据隐私", kp: "财报数据治理", theme: "数据安全", title: "报表里的敏感信息",
      scene: "你导出的分析表中包含客户名单、供应商结算价等未脱敏字段，准备分享给外部咨询团队。是否存在泄露风险？",
      guide: ["哪些财报字段属于敏感/受限数据？", "对外提供数据前应做哪些脱敏？", "泄露商业秘密的法律后果是什么？"],
      case: "《数据安全法》《个人信息保护法》对商业秘密与个人信息处理有明确要求，违规将追责。" },
    { topic: "算法偏差", kp: "可视化建模", theme: "科技伦理", title: "模型不会说谎，但人会",
      scene: "你搭建的信用评分模型在测试中发现：对某区域客户系统性给出更低评分。这可能源于训练数据的历史偏见。",
      guide: ["算法偏差从何而来？", "分析师对模型结果负有什么责任？", "如何在建模中识别并修正偏差？"],
      case: "模型偏差若不加审视，会把历史不公固化为「客观结论」——分析师须做偏差审查。" },
    { topic: "商业秘密", kp: "财报数据治理", theme: "职业操守", title: "未公开的预告",
      scene: "你提前看到尚未披露的毛利率下滑数据，朋友恰好持有该公司股票并询问「最近业绩怎么样」。你该如何回应？",
      guide: ["内幕信息知情人的保密义务是什么？", "随意透露未公开财报可能构成什么行为？", "如何建立「知密不言」的职业习惯？"],
      case: "利用未公开财务信息交易属内幕交易，知情人负有法定保密与回避义务。" }
  ],
  feed: [
    { icon: "check", txt: "完成「三大报表结构解读与勾稽关系」，得分 88", time: "2 天前" },
    { icon: "star", txt: "解锁「图表设计师」徽章", time: "3 天前" },
    { icon: "trophy", txt: "班级排名上升至第 2 名", time: "本周" },
    { icon: "bot", txt: "AI 导师为你推荐「多维度经营解析」", time: "今天" }
  ]
};

function loadStudent() {
  const base = JSON.parse(JSON.stringify(PRESET_STUDENT));
  const raw = lsGet(LS_STU) || {};
  const s = Object.assign({}, base, raw);
  // 班级已统一为财务2433：学生身份与排行榜成员一并归一
  s.cls = base.cls;
  if (!Array.isArray(s.leaderboard) || !s.leaderboard.length) s.leaderboard = base.leaderboard;
  s.leaderboard = s.leaderboard.map(function (p, i) {
    return Object.assign({}, base.leaderboard[i] || { cls: base.cls }, p, { cls: base.cls });
  });
  if (!s.tasks) s.tasks = base.tasks;
  if (!s.badges) s.badges = base.badges;
  if (!s.ethicsScenes) s.ethicsScenes = base.ethicsScenes;
  if (!s.feed) s.feed = base.feed;
  lsSet(LS_STU, s);
  return s;
}

// ============ 页面状态 ============
const student = ref(loadStudent());
const tip = ref("");
let tipTimer = null;
function showTip(t) { tip.value = t; clearTimeout(tipTimer); tipTimer = setTimeout(function () { tip.value = ""; }, 2400); }

// ============ 学习反馈数据（lastFeedback 为空时兜底） ============
const FALLBACK_FB = {
  taskId: "T03", type: "analysis",
  sub: "本次「毛利率异常分析与费用率排查」已完成自动评分与 AI 复盘",
  dims: [
    { label: "任务完成度", full: 20, got: 20 }, { label: "数据处理准确性", full: 20, got: 18 },
    { label: "指标分析能力", full: 20, got: 17 }, { label: "业务逻辑合理性", full: 15, got: 12 },
    { label: "图表表达效果", full: 10, got: 9 }, { label: "报告规范性", full: 10, got: 8 },
    { label: "创新与决策建议", full: 5, got: 5 }
  ],
  radar: null,
  errTags: ["结论缺少业务逻辑", "图表选择不当"],
  errNote: "AI 诊断：你已正确识别销售费用率上升是主因，但对「产品结构变化」分析不足，建议补充不同产品毛利率对比。",
  aiComment: "优点：指标计算准确、趋势判断到位。建议：补做产品维度毛利率拆解；下个推荐任务 → 产品盈利能力分析（T05）。",
  xpGain: 60
};

// ============ 计算属性 ============
const fb = computed(function () { return student.value.lastFeedback || FALLBACK_FB; });
const radar = computed(function () { return fb.value.radar || student.value.abilityRadar; });
const totalScore = computed(function () { return fb.value.dims.reduce(function (s, d) { return s + d.got; }, 0); });

// ============ 内联 SVG 雷达图 ============
const RADAR = { cx: 130, cy: 108, R: 88 };
const grids = [0.25, 0.5, 0.75, 1];
function ang(i) { return -Math.PI / 2 + i * 2 * Math.PI / radar.value.axes.length; }
function pt(i, r) { return [RADAR.cx + r * Math.cos(ang(i)), RADAR.cy + r * Math.sin(ang(i))]; }
function poly(vals) {
  if (!vals) return "";
  return vals.map(function (v, i) { return pt(i, RADAR.R * v / 100).join(","); }).join(" ");
}
function ringPoints(g) {
  return radar.value.axes.map(function (_, i) { return pt(i, RADAR.R * g).join(","); }).join(" ");
}
function axisEnd(i) { return pt(i, RADAR.R); }
const curPoints = computed(function () {
  return radar.value.cur.map(function (v, i) { return pt(i, RADAR.R * v / 100); });
});

// ============ 内联 SVG 得分环 ============
const ringC = 2 * Math.PI * 42;
const ringOffset = computed(function () { return ringC * (1 - Math.min(totalScore.value / 100, 1)); });

// ============ 平台页面跳转（无刷新，等同点击平台 Tab/菜单） ============
const MENU_OF = {
  "student-home": "学习主页", "student-task": "技能训练", "student-ethics": "立德润心", "student-feedback": "学习反馈",
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
function aiComment() { showTip("AI 功能待接入：请在 AI_KEY 填入智能体 API Key"); }

onMounted(function () { });
</script>

<style scoped>
/* ============ 基线（对齐学习主页 / Ant 风格） ============ */
.pg { max-width: 1100px; margin: 0 auto; padding: 18px 16px 40px; }
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
.card { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px; box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); }
.section-title { display: flex; align-items: center; gap: 8px; font-size: 15px; font-weight: 600; color: #1F2733; margin-bottom: 14px; }
.section-title .bar { width: 4px; height: 15px; border-radius: 2px; background: #2B6CD6; }
.card-sub { font-size: 12px; font-weight: 400; color: #97A1B2; margin-left: auto; }
.tag { display: inline-flex; align-items: center; font-size: 12px; padding: 1px 8px; border-radius: 4px; font-weight: 500; white-space: nowrap; }
.tag-orange { background: #FFF3E8; color: #C96A06; }
.tag-red { background: #FFECEC; color: #D93025; }
.tag-green { background: #F1FAE8; color: #3C8E12; }
.mt { margin-top: 16px; }
.mt8 { margin-top: 8px; }
.muted { color: #5A6577; font-size: 13px; }
.hint { font-size: 12px; color: #97A1B2; }
.tip { margin-top: 12px; background: #EAF2FC; border: 1px solid #DCEBFB; color: #1D4FA8; font-size: 13px; border-radius: 6px; padding: 8px 12px; }

/* ============ 雷达图 ============ */
.svg-chart { width: 100%; max-width: 320px; display: block; margin: 0 auto; }
.legend { display: flex; gap: 14px; flex-wrap: wrap; font-size: 12px; color: #5A6577; margin-top: 8px; justify-content: center; }
.legend i { display: inline-block; width: 10px; height: 10px; border-radius: 2px; margin-right: 5px; vertical-align: middle; }
.lg-me { background: #2B6CD6; }
.lg-avg { background: #97A1B2; }

/* ============ 得分环 / 布局 ============ */
.fb-layout { display: flex; gap: 16px; align-items: flex-start; }
.score-card { flex: 0 0 220px; text-align: center; }
.score-ring { width: 96px; height: 96px; color: #2B6CD6; margin: 0 auto; }
.score-label { font-weight: 600; margin-top: 8px; color: #1F2733; }
.score-tag { margin-top: 8px; }
.fb-main { flex: 1; min-width: 0; }

/* ============ 分项评分 ============ */
.fb-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
.fb-item { background: #F7FAFD; border-radius: 6px; padding: 12px 14px; }
.fb-item .fk { font-size: 12px; color: #5A6577; }
.fb-item .fv { font-size: 18px; font-weight: 600; margin-top: 2px; color: #1F2733; font-variant-numeric: tabular-nums; }
.fb-item .fv small { font-size: 12px; color: #97A1B2; font-weight: 400; }

/* ============ 结论 / 概览 ============ */
.fb-echo { font-size: 13px; line-height: 1.6; margin-bottom: 8px; color: #1F2733; }
.fb-echo b { color: #1D4FA8; }
.ti-list { margin: 0; padding-left: 18px; font-size: 13px; color: #5A6577; line-height: 1.8; list-style: disc; }
.ti-list li { margin-bottom: 8px; }
.scroll-x { overflow-x: auto; }
.data-table { width: 100%; border-collapse: collapse; font-size: 13px; font-variant-numeric: tabular-nums; }
.data-table th, .data-table td { padding: 8px 10px; border-bottom: 1px solid #EEF2F8; text-align: right; }
.data-table th:first-child, .data-table td:first-child { text-align: left; }
.data-table thead th { background: #F7FAFD; font-weight: 600; color: #1F2733; }

/* ============ 错因 / AI 评语 ============ */
.err-tag { margin: 2px; }
.err-note { color: #5A6577; font-size: 13px; margin-top: 10px; line-height: 1.7; }
.fb-actions { display: flex; gap: 10px; margin-top: 14px; flex-wrap: wrap; }

@media (max-width: 900px) {
  .fb-layout { flex-direction: column; }
  .score-card { flex: none; }
  .fb-grid { grid-template-columns: 1fr; }
}
</style>
