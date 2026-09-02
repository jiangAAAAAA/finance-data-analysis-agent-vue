<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">评学档案</span>
      <span class="page-head-sub">增值 Agent · 能力雷达与多元评价</span>
    </div>

    <!-- 能力雷达 + 等级经验/徽章 -->
    <div class="grid-2">
      <div class="card">
        <div class="section-title"><span class="bar"></span>能力雷达<span class="card-sub">增值 Agent 生成</span></div>
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
        <p class="hint mt8">短板维度：「综合财务分析(60)」「现金流诊断(66)」，建议优先补强。</p>
      </div>

      <div class="card">
        <div class="section-title"><span class="bar"></span>等级与经验</div>
        <div class="xp-lab">
          <span class="muted">距离下一级还需 {{ student.xpMax - student.xp }} XP</span>
          <span class="xp-pct">{{ xpPct }}%</span>
        </div>
        <div class="bar-bg tall"><div class="bar-fill f-blue" :style="{ width: xpPct + '%' }"></div></div>
        <p class="xp-note">Lv.{{ student.level }} {{ student.lvName }} · 晋升 Lv.4 高级财务分析师需完成偿债与营运能力综合实验任务（T06/T07）。</p>

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
      <div class="section-title"><span class="bar"></span>电子学习档案<span class="card-sub">任务过程与结果留痕</span></div>
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
          <button v-if="x.status !== 'locked'" class="btn btn-ghost btn-sm port-detail" @click="openFb(x)">查看详情</button>
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

    <!-- 学习反馈详情弹窗 -->
    <div class="mask" v-if="fbTask" @click.self="closeFb">
      <div class="modal">
        <div class="m-h">
          学习反馈 · {{ fbTask.name }}
          <button class="x-btn" @click="closeFb">✕</button>
        </div>
        <div class="m-b">
          <p class="fb-sub">本次任务自动评分与 AI 复盘 · 增值 Agent</p>

          <!-- 能力雷达 -->
          <div class="card">
            <div class="section-title"><span class="bar"></span>能力雷达 · 多维画像<span class="card-sub">评学增值 · 增值 Agent</span></div>
            <svg class="svg-chart" viewBox="0 0 260 220" role="img">
              <polygon v-for="g in grids" :key="'fg' + g" :points="ringPoints(g)" fill="none" stroke="#EEF2F8" />
              <g v-for="(a, i) in radar.axes" :key="'fax' + i">
                <line :x1="RADAR.cx" :y1="RADAR.cy" :x2="axisEnd(i)[0]" :y2="axisEnd(i)[1]" stroke="#EEF2F8" />
                <text :x="axisEnd(i)[0]" :y="axisEnd(i)[1] + (axisEnd(i)[1] > RADAR.cy ? 12 : -4)" font-size="10" fill="#5A6577" text-anchor="middle">{{ a }}</text>
              </g>
              <polygon :points="poly(radar.avg)" fill="rgba(151,161,178,.15)" stroke="#97A1B2" stroke-width="1.5" />
              <polygon :points="poly(radar.cur)" fill="rgba(43,108,214,.16)" stroke="#2B6CD6" stroke-width="2" />
              <circle v-for="(p, i) in curPoints" :key="'fd' + i" :cx="p[0]" :cy="p[1]" r="2.5" fill="#2B6CD6" />
            </svg>
            <div class="legend">
              <span><i class="lg-me"></i>林晓</span>
              <span><i class="lg-avg"></i>班级均值</span>
            </div>
          </div>

          <!-- 得分环 + 分项评分 / 错因 / AI 评语 -->
          <div class="fb-layout mt">
            <div class="card score-card">
              <svg viewBox="0 0 96 96" class="score-ring" role="img">
                <circle cx="48" cy="48" r="42" fill="none" stroke="#EEF2F8" stroke-width="8" />
                <circle cx="48" cy="48" r="42" fill="none" stroke="#2B6CD6" stroke-width="8"
                  stroke-linecap="round" :stroke-dasharray="ringC" :stroke-dashoffset="ringOffset"
                  transform="rotate(-90 48 48)" style="transition: stroke-dashoffset .8s ease" />
                <text x="48" y="54" font-size="22" font-weight="600" fill="#1F2733" text-anchor="middle">{{ fbTotal }}</text>
              </svg>
              <div class="score-label">综合得分 {{ fbTotal }}</div>
              <span class="tag tag-orange score-tag">获得 +{{ fbData.xpGain }} 经验 · 徽章待解锁</span>
            </div>

            <div class="fb-main">
              <div class="card">
                <div class="section-title"><span class="bar"></span>分项评分</div>
                <div class="fb-grid">
                  <div class="fb-item" v-for="d in fbData.dims" :key="d.label">
                    <div class="fk">{{ d.label }} <span class="muted">({{ d.full }}分)</span></div>
                    <div class="fv">{{ d.got }}<small> / {{ d.full }}</small></div>
                  </div>
                </div>
              </div>

              <div class="card mt">
                <div class="section-title"><span class="bar"></span>错因分析</div>
                <template v-if="fbData.errTags.length"><span class="tag tag-red err-tag" v-for="e in fbData.errTags" :key="e">{{ e }}</span></template>
                <span v-else class="tag tag-green">全部正确，无显著错因</span>
                <p class="err-note">{{ fbData.errNote }}</p>
              </div>

              <div class="card mt">
                <div class="section-title"><span class="bar"></span>AI 评语与下一步<span class="card-sub">增值 Agent 生成</span></div>
                <p class="err-note">{{ fbData.aiComment }}</p>
                <div class="fb-actions">
                  <button class="btn btn-ghost" @click="aiComment">生成 AI 评语</button>
                </div>
              </div>
            </div>
          </div>
        </div>
        <div class="m-f">
          <button class="btn btn-default" @click="closeFb">关闭</button>
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

// ============ 个性化学习路径（内联演示数据） ============
const paths = [
  { t: "T05 营运能力分析", act: "进行中", cls: "orange", why: "当前任务，完成后解锁 M2 高级关卡" },
  { t: "T06 偿债能力与营运能力综合分析", act: "推荐", cls: "blue", why: "交叉验证两类能力，补齐「财务状况评估」短板（当前 78 → 目标 85）" },
  { t: "T09 现金流建模分析", act: "推荐", cls: "blue", why: "补足「现金流诊断」短板（当前 66），掌握「造血」能力判断" },
  { t: "T13 风险洞察与管理建议报告", act: "进阶", cls: "gray", why: "将分析转化为管理建议，提升「综合财务分析」输出能力" }
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

// ============ 学习反馈详情（弹窗内容为学习反馈页，按任务编排演示数据） ============
const FB_DEMO = {
  T01: { xpGain: 30, dims: [
      { label: "任务完成度", full: 20, got: 20 }, { label: "数据处理准确性", full: 20, got: 19 },
      { label: "指标分析能力", full: 20, got: 18 }, { label: "业务逻辑合理性", full: 15, got: 13 },
      { label: "图表表达效果", full: 10, got: 9 }, { label: "报告规范性", full: 10, got: 9 },
      { label: "创新与决策建议", full: 5, got: 4 }
    ], errTags: [],
    errNote: "AI 诊断：利润表各项目计算路径正确，「财务费用求证」小实验结论完整，无显著错因，可尝试延伸解读净利率变化。",
    aiComment: "优点：结构认知到位、计算路径规范。建议：结合盈利指标公式巩固，为建模分析打基础；下个推荐任务 → 盈利能力建模分析（T02）。" },
  T02: { xpGain: 45, dims: [
      { label: "任务完成度", full: 20, got: 20 }, { label: "数据处理准确性", full: 20, got: 18 },
      { label: "指标分析能力", full: 20, got: 17 }, { label: "业务逻辑合理性", full: 15, got: 12 },
      { label: "图表表达效果", full: 10, got: 9 }, { label: "报告规范性", full: 10, got: 8 },
      { label: "创新与决策建议", full: 5, got: 4 }
    ], errTags: ["结论缺少业务逻辑"],
    errNote: "AI 诊断：建模过程规范，但模型结论的解读停留在数值层面，缺少原材料价格上涨对毛利率影响的业务归因。",
    aiComment: "优点：模型结构完整、指标计算准确。建议：补充模型结论的业务归因解读；下个推荐任务 → 盈利能力下滑问题追溯与管理建议（T03）。" },
  T03: { xpGain: 55, dims: [
      { label: "任务完成度", full: 20, got: 20 }, { label: "数据处理准确性", full: 20, got: 18 },
      { label: "指标分析能力", full: 20, got: 18 }, { label: "业务逻辑合理性", full: 15, got: 13 },
      { label: "图表表达效果", full: 10, got: 8 }, { label: "报告规范性", full: 10, got: 9 },
      { label: "创新与决策建议", full: 5, got: 4 }
    ], errTags: ["图表选择不当"],
    errNote: "AI 诊断：盈利下滑追溯链路完整、管理建议可落地，但下滑趋势的呈现宜用趋势线图替代饼图，图表选型需优化。",
    aiComment: "优点：归因逻辑清晰、管理建议有可落地动作。建议：优化数据表达的图表选型；下个推荐任务 → 认知资产负债表（T04）。" },
  T04: { xpGain: 35, dims: [
      { label: "任务完成度", full: 20, got: 20 }, { label: "数据处理准确性", full: 20, got: 18 },
      { label: "指标分析能力", full: 20, got: 16 }, { label: "业务逻辑合理性", full: 15, got: 12 },
      { label: "图表表达效果", full: 10, got: 9 }, { label: "报告规范性", full: 10, got: 8 },
      { label: "创新与决策建议", full: 5, got: 3 }
    ], errTags: ["指标理解错误"],
    errNote: "AI 诊断：平衡规则理解到位，但流动/非流动项目的划分标准理解不够准确，建议巩固「一年或一个营业周期」口径。",
    aiComment: "优点：结构分析完整、平衡规则验证正确。建议：加强资产项目分类口径练习；下个推荐任务 → 营运能力分析（T05）。" },
  T05: { xpGain: 60, dims: [
      { label: "任务完成度", full: 20, got: 20 }, { label: "数据处理准确性", full: 20, got: 18 },
      { label: "指标分析能力", full: 20, got: 17 }, { label: "业务逻辑合理性", full: 15, got: 12 },
      { label: "图表表达效果", full: 10, got: 9 }, { label: "报告规范性", full: 10, got: 8 },
      { label: "创新与决策建议", full: 5, got: 5 }
    ], errTags: ["结论缺少业务逻辑", "图表选择不当"],
    errNote: "AI 诊断：你已正确计算应收账款周转率等指标，但停留于数值层面，缺少与企业信用政策、行业周转天数基准的联动分析，建议补充对比解读。",
    aiComment: "优点：指标计算准确、公式运用规范。建议：补充营运能力变动的业务动因解读；下个推荐任务 → 偿债能力与营运能力综合分析（T06）。" }
};
const fbTask = ref(null);
const fbData = computed(function () { return FB_DEMO[fbTask.value ? fbTask.value.id : ""] || FB_DEMO.T05; });
const fbTotal = computed(function () { return fbData.value.dims.reduce(function (s, d) { return s + d.got; }, 0); });
const ringC = 2 * Math.PI * 42;
const ringOffset = computed(function () { return ringC * (1 - Math.min(fbTotal.value / 100, 1)); });
function openFb(x) { fbTask.value = x; }
function closeFb() { fbTask.value = null; }
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

/* ============ 按钮 ============ */
.btn { display: inline-flex; align-items: center; gap: 6px; height: 34px; padding: 0 16px; border-radius: 6px; font-size: 14px; font-weight: 600; border: 1px solid transparent; cursor: pointer; transition: all .15s ease; white-space: nowrap; }
.btn-primary { background: #2B6CD6; color: #fff; }
.btn-primary:hover { background: #4D8BE8; }
.btn-default { background: #fff; border-color: #E3E9F2; color: #1F2733; }
.btn-default:hover { border-color: #2B6CD6; color: #2B6CD6; }
.btn-ghost { color: #2B6CD6; background: #EAF2FC; }
.btn-ghost:hover { background: #DCEBFB; }
.btn-sm { height: 28px; padding: 0 12px; font-size: 13px; }

/* ============ 电子档案袋 ============ */
.port-wrap { display: flex; flex-direction: column; gap: 8px; }
.port-item { display: flex; align-items: center; gap: 10px; padding: 8px 10px; border: 1px solid #EEF2F8; border-radius: 6px; transition: all .15s ease; }
.port-item:hover { border-color: #C7D6EC; background: #F7FAFD; }
.port-av { width: 26px; height: 26px; border-radius: 50%; flex-shrink: 0; background: #EAF2FC; color: #2B6CD6; display: flex; align-items: center; justify-content: center; font-weight: 600; font-size: 12px; }
.port-body { flex: 1; min-width: 0; }
.port-name { font-weight: 600; font-size: 13px; color: #1F2733; }
.port-sub { font-size: 12px; color: #5A6577; display: flex; align-items: center; gap: 5px; flex-wrap: wrap; margin-top: 2px; }
.port-score { font-weight: 600; white-space: nowrap; font-variant-numeric: tabular-nums; }
.port-detail { flex-shrink: 0; }

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

/* ============ 学习反馈详情弹窗 ============ */
.mask { position: fixed; inset: 0; background: rgba(16, 24, 40, .45); z-index: 50; display: flex; align-items: center; justify-content: center; padding: 20px; animation: maskFade .2s; }
@keyframes maskFade { from { opacity: 0; } to { opacity: 1; } }
.modal { background: #fff; border-radius: 8px; width: 860px; max-width: 100%; max-height: 88vh; overflow: auto; box-shadow: 0 6px 16px rgba(16, 38, 76, .10), 0 3px 6px rgba(16, 38, 76, .06); animation: modalIn .22s cubic-bezier(.3, 1.2, .5, 1); }
@keyframes modalIn { from { opacity: 0; transform: translateY(14px) scale(.97); } to { opacity: 1; transform: none; } }
.modal .m-h { position: sticky; top: 0; z-index: 1; background: #fff; padding: 18px 22px; border-bottom: 1px solid #E3E9F2; font-size: 16px; font-weight: 600; display: flex; justify-content: space-between; align-items: center; }
.modal .m-b { padding: 18px 22px; }
.modal .m-f { padding: 14px 22px; border-top: 1px solid #E3E9F2; display: flex; justify-content: flex-end; gap: 10px; }
.x-btn { width: 28px; height: 28px; border-radius: 6px; color: #97A1B2; display: flex; align-items: center; justify-content: center; transition: .15s; border: none; background: none; cursor: pointer; font-size: 14px; }
.x-btn:hover { background: #F7FAFD; color: #1F2733; }
.fb-sub { font-size: 12px; color: #97A1B2; margin-bottom: 14px; }

/* 得分环 / 布局 */
.fb-layout { display: flex; gap: 16px; align-items: flex-start; }
.score-card { flex: 0 0 220px; text-align: center; }
.score-ring { width: 96px; height: 96px; color: #2B6CD6; margin: 0 auto; }
.score-label { font-weight: 600; margin-top: 8px; color: #1F2733; }
.score-tag { margin-top: 8px; }
.fb-main { flex: 1; min-width: 0; }

/* 分项评分 */
.fb-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
.fb-item { background: #F7FAFD; border-radius: 6px; padding: 12px 14px; }
.fb-item .fk { font-size: 12px; color: #5A6577; }
.fb-item .fv { font-size: 18px; font-weight: 600; margin-top: 2px; color: #1F2733; font-variant-numeric: tabular-nums; }
.fb-item .fv small { font-size: 12px; color: #97A1B2; font-weight: 400; }

/* 错因 / AI 评语 */
.err-tag { margin: 2px; }
.err-note { color: #5A6577; font-size: 13px; margin-top: 10px; line-height: 1.7; }
.fb-actions { display: flex; gap: 10px; margin-top: 14px; flex-wrap: wrap; }

@media (max-width: 900px) {
  .grid-2 { grid-template-columns: 1fr; }
  .badge-grid { grid-template-columns: repeat(2, 1fr); }
  .fb-layout { flex-direction: column; }
  .score-card { flex: none; width: 100%; }
  .fb-grid { grid-template-columns: 1fr; }
}
</style>
