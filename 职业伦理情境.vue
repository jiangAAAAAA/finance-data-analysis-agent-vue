<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">职业伦理情境</span>
      <span class="page-head-sub">立德 Agent 智能推送 · 课程思政资源库</span>
    </div>

    <!-- 主题覆盖横幅 -->
    <div class="ethic-banner">
      <div class="eb-ic">盾</div>
      <div class="eb-body">
        <div class="eb-title">情境主题覆盖</div>
        <div class="eb-chips">
          <span v-for="t in topics" :key="t" class="tag" :class="{ 'tag-blue': cur && cur.topic === t, 'tag-gray': !(cur && cur.topic === t) }">{{ t }}</span>
        </div>
      </div>
    </div>

    <!-- 情境卡片列表 + 右侧讨论面板 -->
    <div class="ethic-layout">
      <div class="ethic-list">
        <div
          v-for="s in student.ethicsScenes"
          :key="s.topic"
          class="ethic-card"
          :class="{ cur: cur && cur.topic === s.topic }"
          @click="selectScene(s)"
        >
          <div class="ec-head">
            <span class="ec-topic">{{ s.topic }}</span>
            <span class="tag tag-blue">{{ s.kp }}</span>
            <span v-if="cur && cur.topic === s.topic" class="tag tag-green">讨论中</span>
          </div>
          <div class="ec-title">{{ s.title }}</div>
          <div class="ethic-scene">{{ s.scene }}</div>
          <div class="ec-sec">AI 引导思考</div>
          <div class="ethic-guide">
            <div class="eg" v-for="(g, i) in s.guide" :key="i">{{ i + 1 }}. {{ g }}</div>
          </div>
          <div class="ethic-case"><b>案例依据：</b>{{ s.case }}</div>
          <button class="btn btn-ghost btn-sm discuss-btn" @click.stop="selectScene(s)">
            {{ cur && cur.topic === s.topic ? '正在讨论此情境 ›' : '与立德 Agent 讨论此情境 ›' }}
          </button>
        </div>
      </div>

      <div class="ethic-chat">
        <div class="chat-panel">
          <div class="chat-head">
            <span class="chat-title">立德 Agent · 伦理讨论</span>
            <span class="tag tag-blue" v-if="cur">{{ cur.topic }}</span>
          </div>
          <template v-if="cur">
            <div class="chat-scene">{{ cur.scene }}</div>
            <div class="chat-sec">问题引导</div>
            <div class="chat-guide">
              <div class="cg" v-for="(g, i) in cur.guide" :key="i">{{ i + 1 }}. {{ g }}</div>
            </div>
            <button class="btn btn-primary mt8" @click="aiDiscuss">与立德 Agent 讨论</button>
            <p class="hint mt8">立德 Agent · 问题引导 / 案例辨析 / 伦理冲突讨论（待接入）</p>
          </template>
          <div v-else class="empty">← 请选择一个职业伦理情境，开始与立德 Agent 讨论</div>
        </div>
      </div>
    </div>

    <!-- 如何运作 -->
    <div class="card mt2">
      <div class="section-title"><span class="bar"></span>如何运作</div>
      <p class="how-it-works">教师端「启智立德」上传课程思政资源并设置思政资源标签与技能知识标签 → 立德 Agent 依据课程能力图谱对知识点、岗位任务、思政主题、案例素材做标签化处理，生成技能知识点与思政素材的智能匹配 → 教师确认 / 修改关联 → 你在学习对应知识点时，系统自动推送相关职业伦理情境（如数据真实性、财务造假、盈余管理、数据隐私、算法偏差、商业秘密）。</p>
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
const cur = ref(null);
const tip = ref("");
let tipTimer = null;
function showTip(t) { tip.value = t; clearTimeout(tipTimer); tipTimer = setTimeout(function () { tip.value = ""; }, 2400); }

// ============ 计算属性 ============
const topics = computed(function () {
  const seen = {};
  const out = [];
  student.value.ethicsScenes.forEach(function (s) {
    if (!seen[s.topic]) { seen[s.topic] = 1; out.push(s.topic); }
  });
  return out;
});

// ============ 交互 ============
function go(view) { emit("navigate", view); }
function selectScene(s) { cur.value = s; }
function aiDiscuss() { showTip("AI 功能待接入：请在 AI_KEY 填入智能体 API Key"); }

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
.btn-sm { height: 28px; padding: 0 12px; font-size: 13px; }
.mt { margin-top: 16px; }
.mt2 { margin-top: 20px; }
.mt8 { margin-top: 8px; }
.muted { color: #5A6577; font-size: 13px; }
.hint { font-size: 12px; color: #97A1B2; }
.card { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px; box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); }
.section-title { display: flex; align-items: center; gap: 8px; font-size: 15px; font-weight: 600; color: #1F2733; margin-bottom: 14px; }
.section-title .bar { width: 4px; height: 15px; border-radius: 2px; background: #2B6CD6; }
.tag { display: inline-flex; align-items: center; font-size: 12px; padding: 1px 8px; border-radius: 4px; font-weight: 500; white-space: nowrap; }
.tag-blue { background: #EAF2FC; color: #1D4FA8; }
.tag-green { background: #F1FAE8; color: #3C8E12; }
.tag-gray { background: #F7FAFD; color: #97A1B2; }
.tip { margin-top: 12px; background: #EAF2FC; border: 1px solid #DCEBFB; color: #1D4FA8; font-size: 13px; border-radius: 6px; padding: 8px 12px; }
.empty { padding: 40px; text-align: center; color: #97A1B2; font-size: 13px; }

/* ============ 主题覆盖横幅 ============ */
.ethic-banner { display: flex; align-items: center; gap: 14px; background: linear-gradient(135deg, #EAF2FC, #fff); border: 1px solid #E3E9F2; border-radius: 8px; padding: 14px 16px; margin-bottom: 16px; }
.eb-ic { width: 40px; height: 40px; border-radius: 10px; flex-shrink: 0; background: #2B6CD6; color: #fff; font-size: 16px; font-weight: 700; display: flex; align-items: center; justify-content: center; }
.eb-title { font-weight: 600; font-size: 13px; margin-bottom: 7px; color: #1F2733; }
.eb-chips { display: flex; flex-wrap: wrap; gap: 7px; }

/* ============ 双栏布局 ============ */
.ethic-layout { display: grid; grid-template-columns: 1fr 380px; gap: 16px; align-items: start; }
.ethic-list { display: grid; grid-template-columns: 1fr 1fr; gap: 14px; }
.ethic-card { border: 1px solid #E3E9F2; border-radius: 8px; padding: 14px 16px; background: #fff; box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); cursor: pointer; transition: all .18s ease; }
.ethic-card:hover { border-color: #2B6CD6; transform: translateY(-2px); }
.ethic-card.cur { border-color: #52C41A; box-shadow: 0 0 0 2px #F1FAE8; }
.ec-head { display: flex; align-items: center; gap: 8px; margin-bottom: 8px; flex-wrap: wrap; }
.ec-topic { font-weight: 700; font-size: 14px; color: #2B6CD6; }
.ec-title { font-weight: 600; font-size: 13px; margin-bottom: 9px; color: #1F2733; }
.ethic-scene { font-size: 13px; line-height: 1.7; background: #F7FAFD; border-left: 3px solid #FAAD14; border-radius: 6px; padding: 10px 12px; color: #1F2733; }
.ec-sec { font-size: 13px; font-weight: 600; margin: 12px 0 6px; color: #1F2733; }
.ethic-guide { display: flex; flex-direction: column; gap: 7px; }
.eg { font-size: 12px; color: #5A6577; line-height: 1.6; }
.ethic-case { font-size: 12px; color: #5A6577; background: #EAF2FC; border-radius: 6px; padding: 9px 11px; margin-top: 10px; line-height: 1.6; }
.discuss-btn { margin-top: 12px; }

/* ============ 讨论面板（占位） ============ */
.ethic-chat { position: sticky; top: 16px; }
.chat-panel { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px; box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); }
.chat-head { display: flex; align-items: center; justify-content: space-between; gap: 8px; margin-bottom: 12px; padding-bottom: 10px; border-bottom: 1px solid #EEF2F8; }
.chat-title { font-size: 15px; font-weight: 600; color: #1F2733; }
.chat-scene { font-size: 13px; line-height: 1.7; background: #F7FAFD; border-radius: 6px; padding: 10px 12px; color: #1F2733; }
.chat-sec { font-size: 13px; font-weight: 600; margin: 12px 0 6px; color: #1F2733; }
.chat-guide { display: flex; flex-direction: column; gap: 7px; }
.cg { font-size: 12px; color: #5A6577; line-height: 1.6; }

/* ============ 如何运作 ============ */
.how-it-works { font-size: 13px; line-height: 1.8; color: #5A6577; }

@media (max-width: 900px) {
  .ethic-layout { grid-template-columns: 1fr; }
  .ethic-chat { position: static; }
  .ethic-list { grid-template-columns: 1fr; }
}
</style>
