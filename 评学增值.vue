<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">评学增值</span>
      <span class="page-head-sub">增值 Agent · 基于能力图谱与岗位标准的多元评价</span>
    </div>

    <!-- 能力雷达 + 等级经验/徽章 -->
    <div class="grid-2">
      <div class="card">
        <div class="section-title"><span class="bar"></span>能力雷达 · 六维能力画像<span class="card-sub">增值 Agent 生成</span></div>
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
        <p class="hint mt8">短板维度：「风险预警(58)」「现金流分析(70)」，建议优先补强。</p>
      </div>

      <div class="card">
        <div class="section-title"><span class="bar"></span>等级与经验</div>
        <div class="xp-lab">
          <span class="muted">距离下一级还需 {{ student.xpMax - student.xp }} XP</span>
          <span class="xp-pct">{{ xpPct }}%</span>
        </div>
        <div class="bar-bg tall"><div class="bar-fill f-blue" :style="{ width: xpPct + '%' }"></div></div>
        <p class="xp-note">Lv.{{ student.level }} {{ student.lvName }} · 晋升 Lv.4 成本分析师需完成成本与费用分析任务（T04/T07）。</p>

        <div class="section-title badge-title"><span class="bar"></span>成就徽章<span class="card-sub">{{ earnedCount }}/{{ student.badges.length }} 已解锁</span></div>
        <div class="badge-grid">
          <div v-for="b in student.badges" :key="b.n" class="badge-item" :class="{ off: !b.earned }">
            <div class="badge-ic">{{ b.n[0] }}</div>
            <div class="badge-name">{{ b.n }}</div>
            <div class="badge-d">{{ b.d }}</div>
          </div>
        </div>
      </div>
    </div>

    <!-- 电子档案袋 -->
    <div class="card mt2">
      <div class="section-title"><span class="bar"></span>电子档案袋 · 一生一档<span class="card-sub">任务过程与结果留痕</span></div>
      <div class="port-wrap">
        <div class="port-item" v-for="x in student.tasks" :key="x.id">
          <span class="port-av">{{ x.id.slice(1) }}</span>
          <div class="port-body">
            <div class="port-name">{{ x.name }}</div>
            <div class="port-sub">
              {{ x.level }} · {{ x.ability }} ·
              <span v-if="x.status === 'done'" class="tag tag-green">已完成</span>
              <span v-else-if="x.status === 'active'" class="tag tag-orange">进行中</span>
              <span v-else class="tag tag-gray">待解锁</span>
            </div>
          </div>
          <div class="port-score" :style="{ color: scoreColor(x) }">{{ x.score == null ? '—' : x.score + ' 分' }}</div>
        </div>
      </div>
    </div>

    <!-- 个性化学习路径 -->
    <div class="card mt2">
      <div class="section-title"><span class="bar"></span>个性化学习路径<span class="card-sub">按能力短板生成</span></div>
      <div class="path-wrap">
        <div class="path-item" v-for="p in paths" :key="p.t">
          <span class="path-tag" :class="p.cls">{{ p.act }}</span>
          <div class="path-body">
            <div class="path-name">{{ p.t }}</div>
            <div class="path-why">{{ p.why }}</div>
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
  let s = lsGet(LS_STU);
  if (!s) { s = JSON.parse(JSON.stringify(PRESET_STUDENT)); lsSet(LS_STU, s); }
  return s;
}

// ============ 页面状态 ============
const student = ref(loadStudent());
const tip = ref("");
let tipTimer = null;
function showTip(t) { tip.value = t; clearTimeout(tipTimer); tipTimer = setTimeout(function () { tip.value = ""; }, 2400); }

// ============ 个性化学习路径（内联演示数据） ============
const paths = [
  { t: "T03 毛利率异常分析", act: "进行中", cls: "orange", why: "当前任务，完成后解锁中级关卡" },
  { t: "T04 月度费用增长排查", act: "推荐", cls: "blue", why: "补齐「费用分析」，与本次结论能力衔接" },
  { t: "T05 产品盈利能力分析", act: "推荐", cls: "blue", why: "补足「多维分析」短板（当前 85 → 目标 90）" },
  { t: "T09 财务分析报告撰写", act: "进阶", cls: "gray", why: "将分析转化为管理建议，提升「决策支持」" }
];

// ============ 计算属性 ============
const radar = computed(function () { return student.value.abilityRadar; });
const xpPct = computed(function () { return Math.round(student.value.xp / student.value.xpMax * 100); });
const earnedCount = computed(function () { return student.value.badges.filter(function (b) { return b.earned; }).length; });

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

// ============ 样式辅助 ============
function scoreColor(x) {
  if (x.score == null) return "#97A1B2";
  return x.score >= 85 ? "#52C41A" : "#5A6577";
}

// ============ 交互 ============
function go(view) { emit("navigate", view); }

onMounted(function () { });
</script>

<style scoped>
/* ============ 基线（对齐学习主页 / Ant 风格） ============ */
.pg { max-width: 1100px; margin: 0 auto; padding: 18px 16px 40px; }
.page-head { display: flex; align-items: center; gap: 10px; margin-bottom: 16px; flex-wrap: wrap; }
.page-head-bar { width: 4px; height: 22px; border-radius: 2px; background: #1677FF; }
.page-head-title { font-size: 20px; font-weight: 600; color: #1F2733; }
.page-head-sub { font-size: 13px; color: #97A1B2; flex: 1; }
.card { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px; box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); }
.section-title { display: flex; align-items: center; gap: 8px; font-size: 15px; font-weight: 600; color: #1F2733; margin-bottom: 14px; }
.section-title .bar { width: 4px; height: 15px; border-radius: 2px; background: #2B6CD6; }
.card-sub { font-size: 12px; font-weight: 400; color: #97A1B2; margin-left: auto; }
.grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
.tag { display: inline-flex; align-items: center; font-size: 12px; padding: 1px 8px; border-radius: 4px; font-weight: 500; white-space: nowrap; }
.tag-blue { background: #EAF2FC; color: #1D4FA8; }
.tag-green { background: #F1FAE8; color: #3C8E12; }
.tag-orange { background: #FFF3E8; color: #C96A06; }
.tag-gray { background: #F7FAFD; color: #97A1B2; }
.bar-bg { background: #EEF3FA; border-radius: 99px; overflow: hidden; }
.bar-bg.tall { height: 12px; }
.bar-fill { height: 100%; border-radius: 99px; transition: width .4s ease; }
.f-blue { background: #2B6CD6; }
.mt { margin-top: 16px; }
.mt2 { margin-top: 20px; }
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

/* ============ 等级与经验 / 徽章 ============ */
.xp-lab { display: flex; justify-content: space-between; align-items: center; font-size: 13px; margin-bottom: 6px; }
.xp-pct { font-weight: 600; color: #1F2733; }
.xp-note { font-size: 12px; color: #5A6577; margin-top: 10px; line-height: 1.6; }
.badge-title { margin-top: 16px; }
.badge-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; }
.badge-item { border: 1px solid #E3E9F2; border-radius: 8px; padding: 12px; text-align: center; background: #FBFEF9; }
.badge-item.off { background: #F7FAFD; opacity: .6; }
.badge-ic { width: 36px; height: 36px; border-radius: 50%; background: linear-gradient(135deg, #52C41A, #95DE64); color: #fff; font-size: 16px; font-weight: 700; display: flex; align-items: center; justify-content: center; margin: 0 auto 6px; }
.badge-item.off .badge-ic { background: #D9DEE8; }
.badge-name { font-size: 13px; font-weight: 600; color: #1F2733; }
.badge-d { font-size: 11px; color: #97A1B2; margin-top: 2px; }

/* ============ 电子档案袋 ============ */
.port-wrap { display: flex; flex-direction: column; gap: 8px; }
.port-item { display: flex; align-items: center; gap: 10px; padding: 8px 10px; border: 1px solid #EEF2F8; border-radius: 6px; transition: all .15s ease; }
.port-item:hover { border-color: #C7D6EC; background: #F7FAFD; }
.port-av { width: 26px; height: 26px; border-radius: 50%; flex-shrink: 0; background: #EAF2FC; color: #2B6CD6; display: flex; align-items: center; justify-content: center; font-weight: 600; font-size: 12px; }
.port-body { flex: 1; min-width: 0; }
.port-name { font-weight: 600; font-size: 13px; color: #1F2733; }
.port-sub { font-size: 12px; color: #5A6577; display: flex; align-items: center; gap: 5px; flex-wrap: wrap; margin-top: 2px; }
.port-score { font-weight: 600; white-space: nowrap; font-variant-numeric: tabular-nums; }

/* ============ 个性化学习路径 ============ */
.path-wrap { display: flex; flex-direction: column; gap: 8px; }
.path-item { display: flex; align-items: center; gap: 10px; padding: 8px 10px; background: #F7FAFD; border-radius: 6px; }
.path-tag { font-size: 11px; font-weight: 600; padding: 2px 9px; border-radius: 99px; flex-shrink: 0; color: #fff; }
.path-tag.blue { background: #2B6CD6; }
.path-tag.orange { background: #FA8C16; }
.path-tag.gray { background: #97A1B2; }
.path-body { flex: 1; min-width: 0; }
.path-name { font-weight: 600; font-size: 13px; color: #1F2733; }
.path-why { font-size: 12px; color: #5A6577; margin-top: 1px; }

@media (max-width: 900px) {
  .grid-2 { grid-template-columns: 1fr; }
  .badge-grid { grid-template-columns: repeat(2, 1fr); }
}
</style>
