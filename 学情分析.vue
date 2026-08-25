<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">学情分析</span>
      <span class="page-head-sub">{{ teacher.cls }} · 共 {{ c.total }} 人 · 基于 AI 评分、错因与能力雷达生成的教学诊断报告</span>
    </div>

    <!-- Tab 切换：班级 / 个人 -->
    <div class="tabs">
      <button :class="{ active: tab === 'class' }" @click="tab = 'class'">班级视角</button>
      <button :class="{ active: tab === 'person' }" @click="tab = 'person'">个人视角（林晓）</button>
    </div>

    <!-- ============ 班级视角 ============ -->
    <template v-if="tab === 'class'">
      <!-- 统计行 -->
      <div class="kpi-row">
        <div class="kpi"><div class="k">任务完成率</div><div class="v">{{ c.completion }}<small>%</small></div></div>
        <div class="kpi"><div class="k">班级平均分</div><div class="v">{{ c.avg }}</div></div>
        <div class="kpi"><div class="k">活跃人数</div><div class="v">{{ c.active }}<small>/{{ c.total }}</small></div></div>
        <div class="kpi"><div class="k">待复核</div><div class="v c-orange">{{ pendingCount }}<small>份</small></div></div>
        <div class="kpi"><div class="k">需重点关注</div><div class="v c-red">{{ teacher.needsFocus.length }}<small>人</small></div></div>
      </div>

      <!-- 班级能力雷达 + 班级错因分布 -->
      <div class="grid-2 mt">
        <div class="card">
          <div class="section-title"><span class="bar"></span>班级能力雷达 · 均值 vs 目标线</div>
          <svg class="radar-svg" viewBox="0 0 260 220" role="img" aria-label="班级能力雷达图">
            <polygon v-for="g in grids" :key="g" :points="radarRing(g, classRadar.axes.length)" fill="none" stroke="#EEF2F8" />
            <g v-for="(a, i) in classRadar.axes" :key="'ax' + i">
              <line :x1="cx" :y1="cy" :x2="classEnds[i][0]" :y2="classEnds[i][1]" stroke="#EEF2F8" />
              <text :x="classEnds[i][0]" :y="classEnds[i][1] + (classEnds[i][1] > cy ? 12 : -4)" font-size="10" fill="#5A6577" text-anchor="middle">{{ a }}</text>
            </g>
            <polygon :points="radarPoly(classRadar.compare)" fill="rgba(151,161,178,.15)" stroke="#97A1B2" stroke-width="1.5" />
            <polygon :points="radarPoly(classRadar.cur)" fill="rgba(43,108,214,.16)" stroke="#2B6CD6" stroke-width="2" />
            <circle v-for="(v, i) in classRadar.cur" :key="'d' + i" :cx="radarPt(i, v, classRadar.cur.length)[0]" :cy="radarPt(i, v, classRadar.cur.length)[1]" r="2.5" fill="#2B6CD6" />
          </svg>
          <div class="legend">
            <span><i class="lg-blue"></i>班级均值</span>
            <span><i class="lg-gray"></i>目标线(85)</span>
          </div>
        </div>
        <div class="card">
          <div class="section-title"><span class="bar"></span>班级错因分布（人次）</div>
          <div class="err-row" v-for="e in teacher.errorDist" :key="e.k">
            <div class="err-k">{{ e.k }}</div>
            <div class="bar-bg tall err-bar"><div class="bar-fill f-orange" :style="{ width: e.v / maxError * 100 + '%' }"></div></div>
            <div class="err-v">{{ e.v }}</div>
          </div>
          <div class="section-title mt2"><span class="bar"></span>诊断结论</div>
          <p class="diag-note">高频错因为「{{ topErrors[0].k }}」与「{{ topErrors[1].k }}」，说明学生在公式应用与业务解释环节最薄弱，需在后续任务中重点干预。</p>
        </div>
      </div>

      <!-- 优势分析 + 能力短板及成因 -->
      <div class="grid-2 mt">
        <div class="card">
          <div class="section-title"><span class="bar"></span>优势分析</div>
          <ul class="ti-list">
            <li v-for="s in sw.strengths" :key="s.k"><b>{{ s.k }}</b> <span class="muted">（均值 {{ s.v }}）</span>：整体掌握较好，可据此设计进阶与拓展任务，并让该生承担小组示范。</li>
          </ul>
        </div>
        <div class="card">
          <div class="section-title"><span class="bar"></span>能力短板及成因</div>
          <div class="sw-item" v-for="w in sw.weaknesses" :key="w.k">
            <div class="sw-k">{{ w.k }} <span class="sw-v">{{ w.v }}</span></div>
            <div class="sw-cause">形成原因：{{ teacher.cause.classCause[w.k] || "相关训练不足，需针对性补练。" }}</div>
          </div>
        </div>
      </div>

      <!-- 教学建议 -->
      <div class="card mt">
        <div class="section-title"><span class="bar"></span>教学建议（针对短板）</div>
        <ul class="ti-list">
          <li v-for="w in sw.weaknesses" :key="'g' + w.k"><b>{{ w.k }}</b>：{{ teacher.cause.classSugg[w.k] || "建议针对性推送练习并安排一对一辅导。" }}</li>
        </ul>
      </div>

      <!-- 重点关注学生 + 分层干预建议 -->
      <div class="grid-2 mt">
        <div class="card">
          <div class="section-title"><span class="bar"></span>重点关注学生</div>
          <div class="fi" v-for="f in teacher.needsFocus" :key="f.nm">
            <div class="fi-h">
              <b>{{ f.nm }}</b>
              <span class="tag tag-red">{{ f.score === null ? "未提交" : f.score + "分" }}</span>
            </div>
            <div class="fi-issue">{{ f.issue }}</div>
          </div>
        </div>
        <div class="card">
          <div class="section-title"><span class="bar"></span>分层干预建议</div>
          <div class="grp-item" v-for="g in groups" :key="g.k">
            <div class="grp-h">{{ g.k }} <span class="muted">（{{ g.mem.length }} 人）</span></div>
            <div class="grp-mem">
              <span class="itag skill" v-for="m in g.mem" :key="m">{{ m }}</span>
              <span v-if="!g.mem.length" class="muted">—</span>
            </div>
            <div class="grp-tip">干预建议：{{ g.tip }}</div>
          </div>
        </div>
      </div>
    </template>

    <!-- ============ 个人视角 ============ -->
    <template v-else>
      <div class="grid-2">
        <div class="card">
          <div class="section-title"><span class="bar"></span>林晓 · 能力雷达（vs 班级均值）</div>
          <svg class="radar-svg" viewBox="0 0 260 220" role="img" aria-label="林晓个人能力雷达图">
            <polygon v-for="g in grids" :key="g" :points="radarRing(g, personRadar.axes.length)" fill="none" stroke="#EEF2F8" />
            <g v-for="(a, i) in personRadar.axes" :key="'ax' + i">
              <line :x1="cx" :y1="cy" :x2="personEnds[i][0]" :y2="personEnds[i][1]" stroke="#EEF2F8" />
              <text :x="personEnds[i][0]" :y="personEnds[i][1] + (personEnds[i][1] > cy ? 12 : -4)" font-size="10" fill="#5A6577" text-anchor="middle">{{ a }}</text>
            </g>
            <polygon :points="radarPoly(personRadar.compare)" fill="rgba(151,161,178,.15)" stroke="#97A1B2" stroke-width="1.5" />
            <polygon :points="radarPoly(personRadar.cur)" fill="rgba(43,108,214,.16)" stroke="#2B6CD6" stroke-width="2" />
            <circle v-for="(v, i) in personRadar.cur" :key="'d' + i" :cx="radarPt(i, v, personRadar.cur.length)[0]" :cy="radarPt(i, v, personRadar.cur.length)[1]" r="2.5" fill="#2B6CD6" />
          </svg>
          <div class="legend">
            <span><i class="lg-blue"></i>林晓</span>
            <span><i class="lg-gray"></i>班级均值</span>
          </div>
        </div>
        <div class="card">
          <div class="section-title"><span class="bar"></span>林晓 · 个人错因分布（人次）</div>
          <div class="err-row" v-for="e in teacher.personErrors" :key="e[0]">
            <div class="err-k">{{ e[0] }}</div>
            <div class="bar-bg tall err-bar"><div class="bar-fill f-orange" :style="{ width: e[1] / 6 * 100 + '%' }"></div></div>
            <div class="err-v">{{ e[1] }}</div>
          </div>
        </div>
      </div>
      <div class="grid-2 mt">
        <div class="card">
          <div class="section-title"><span class="bar"></span>个人优势</div>
          <ul class="ti-list">
            <li v-for="s in psw.strengths" :key="s.k"><b>{{ s.k }}</b> <span class="muted">（均值 {{ s.v }}）</span>：整体掌握较好，可据此设计进阶与拓展任务，并让该生承担小组示范。</li>
          </ul>
        </div>
        <div class="card">
          <div class="section-title"><span class="bar"></span>个人能力短板及成因</div>
          <div class="sw-item" v-for="w in psw.weaknesses" :key="w.k">
            <div class="sw-k">{{ w.k }} <span class="sw-v">{{ w.v }}</span></div>
            <div class="sw-cause">形成原因：{{ teacher.cause.personCause[w.k] || "相关训练不足，需针对性补练。" }}</div>
          </div>
        </div>
      </div>
      <div class="card mt">
        <div class="section-title"><span class="bar"></span>个人学习建议</div>
        <ul class="ti-list">
          <li v-for="w in psw.weaknesses" :key="'pg' + w.k"><b>{{ w.k }}</b>：{{ teacher.cause.personSugg[w.k] || "建议针对性推送练习并安排一对一辅导。" }}</li>
        </ul>
      </div>
    </template>

    <!-- AI 教学顾问占位 -->
    <div class="card mt">
      <div class="section-title"><span class="bar"></span>追问 AI 教学顾问</div>
      <p class="muted">增值 Agent · 基于诊断报告生成干预策略，如：中等层学生怎么干预？</p>
      <button class="btn btn-primary mt8" @click="askAI">追问 AI 教学顾问</button>
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

const PRESET_TEACHER = {
  cls: "财管2201",
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
  classStats: { completion: 82, avg: 79, active: 31, total: 38 },
  weeklySubs: 16,
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
  ],
  cause: {
    classCause: {
      "财报数据治理": "多源财报口径不统一，数据清洗与治理环节专项训练不足。",
      "经营解析能力": "多维下钻（产品/地区/渠道）训练偏少，易停留于总量描述，缺乏结构拆解。",
      "异常特征识别": "异常波动的识别方法（同比/环比/阈值）练习不足，对离群点敏感度弱。",
      "现金流分析": "重利润、轻现金，现金流与利润质量联动分析的训练欠缺。",
      "风险预警能力": "偿债能力分层与风险预警逻辑不熟，预警阈值设定经验少。",
      "可视化建模": "图表选型与看板迭代训练不足，数据表达效果偏弱。"
    },
    classSugg: {
      "财报数据治理": "增设「财报口径对齐与数据治理」专项，配套清洗 SOP 与校验清单。",
      "经营解析能力": "设计产品/地区/渠道多维拆解任务，强化下钻与对比分析。",
      "异常特征识别": "引入异常检测案例库（同比/环比/阈值），训练识别套路与归因。",
      "现金流分析": "增加「利润质量评价」任务，强化现金流与利润联动解读。",
      "风险预警能力": "增加偿债能力分层与预警诊断任务，明确阈值设定方法。",
      "可视化建模": "开设图表选型与 BI 看板迭代专题，提升可视化表达力。"
    },
    personCause: {
      "数据处理": "报表取数与字段映射仍易出错，数据预处理不够稳健。",
      "指标分析": "指标公式掌握较好，但跨指标联动解读偏弱。",
      "异常识别": "对异常波动的敏感度不足，识别方法训练偏少。",
      "业务解释": "能算出结果但业务语义解释不够，结论缺乏管理语言。",
      "图表表达": "图表表达是强项，保持并用于汇报呈现。",
      "决策建议": "给出可执行建议的能力偏弱，决策闭环训练不足。"
    },
    personSugg: {
      "数据处理": "补充取数校验与字段映射练习，建立预处理清单。",
      "指标分析": "增加跨指标联动解读训练（如毛利率×费用率联动）。",
      "异常识别": "引入异常检测案例，训练识别与归因。",
      "业务解释": "强化「用管理语言解释指标」训练，结论对齐业务。",
      "图表表达": "可承担小组可视化汇报，带动同伴。",
      "决策建议": "增加「给出行动建议」专项，补齐决策闭环。"
    }
  },
  person: {
    axes: ["数据处理", "指标分析", "异常识别", "业务解释", "图表表达", "决策建议"],
    cur: [88, 82, 74, 70, 90, 62]
  },
  personErrors: [["公式错误", 4], ["指标理解错误", 6], ["图表选择不当", 3], ["结论缺业务逻辑", 5], ["分析不全面", 2]]
};

function loadTeacher() {
  let t = lsGet(LS_TEACHER);
  if (!t) {
    t = JSON.parse(JSON.stringify(PRESET_TEACHER));
    lsSet(LS_TEACHER, t);
  } else {
    // 合并 PRESET 兜底：旧版 localStorage 数据可能缺少新增字段（如 cause），补齐避免渲染报错
    t = Object.assign({}, JSON.parse(JSON.stringify(PRESET_TEACHER)), t);
    t.cause = Object.assign({}, PRESET_TEACHER.cause, t.cause || {});
    lsSet(LS_TEACHER, t);
  }
  return t;
}

// ============ 页面状态 ============
const teacher = ref(loadTeacher());
const tab = ref("class");
const tip = ref("");
let tipTimer = null;
function showTip(t) { tip.value = t; clearTimeout(tipTimer); tipTimer = setTimeout(function () { tip.value = ""; }, 2400); }

// ============ AI 占位 ============
const AI_URL = "https://agent.gjt-smart.com/v1/chat-messages";
const AI_KEY = "app-xxxxxx"; // TODO 占位
function askAI() { showTip("AI 功能待接入：请在 AI_KEY 填入智能体 API Key"); }

// ============ 计算属性 ============
const c = computed(function () { return teacher.value.classStats; });
const pendingCount = computed(function () { return teacher.value.submissions.filter(function (s) { return s.status === "待复核"; }).length; });
const targetLine = computed(function () { return teacher.value.radar.axes.map(function () { return 85; }); });
const maxError = computed(function () { return Math.max.apply(null, teacher.value.errorDist.map(function (x) { return x.v; })); });
const topErrors = computed(function () { return [].concat(teacher.value.errorDist).sort(function (a, b) { return b.v - a.v; }); });

function buildSW(axes, vals) {
  var arr = axes.map(function (k, i) { return { k: k, v: vals[i] }; }).sort(function (a, b) { return b.v - a.v; });
  return { strengths: arr.slice(0, 2), weaknesses: arr.slice(-2).reverse() };
}
const sw = computed(function () { return buildSW(teacher.value.radar.axes, teacher.value.radar.avg); });
const psw = computed(function () { return buildSW(teacher.value.person.axes, teacher.value.person.cur); });

const groups = computed(function () {
  return [
    { k: "优秀层（≥85）", mem: teacher.value.submissions.filter(function (s) { return s.ai >= 85; }).map(function (s) { return s.student; }), tip: "可布置挑战性拓展任务，并让其担任小组导师，带动同伴。" },
    { k: "中等层（70–84）", mem: teacher.value.submissions.filter(function (s) { return s.ai >= 70 && s.ai < 85; }).map(function (s) { return s.student; }), tip: "针对薄弱维度个性化推送练习，强化跨指标联动解读。" },
    { k: "待提升层（<70）/未提交", mem: teacher.value.submissions.filter(function (s) { return s.ai < 70; }).map(function (s) { return s.student; }).concat(teacher.value.needsFocus.filter(function (f) { return f.score === null; }).map(function (f) { return f.nm; })), tip: "安排一对一辅导，先补基础概念，降低任务粒度分阶段达成。" }
  ];
});

// ============ 内联 SVG 雷达图（自适应） ============
const cx = 130, cy = 108, R = 88;
const grids = [0.25, 0.5, 0.75, 1];
function ang(i, n) { return -Math.PI / 2 + i * 2 * Math.PI / n; }
function pt(i, r, n) { return [cx + r * Math.cos(ang(i, n)), cy + r * Math.sin(ang(i, n))]; }
function radarRing(g, n) { return Array.from({ length: n }, function (_, i) { return pt(i, R * g, n).join(","); }).join(" "); }
function radarPoly(vals) { var n = vals.length; return vals.map(function (v, i) { return pt(i, R * v / 100, n).join(","); }).join(" "); }
function radarEnds(n) { return Array.from({ length: n }, function (_, i) { return pt(i, R, n); }); }
function radarPt(i, v, n) { return pt(i, R * v / 100, n); }

const classRadar = computed(function () { return { axes: teacher.value.radar.axes, cur: teacher.value.radar.avg, compare: targetLine.value }; });
const classEnds = computed(function () { return radarEnds(classRadar.value.axes.length); });
const personRadar = computed(function () { return { axes: teacher.value.person.axes, cur: teacher.value.person.cur, compare: teacher.value.radar.avg }; });
const personEnds = computed(function () { return radarEnds(personRadar.value.axes.length); });

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
.btn { display: inline-flex; align-items: center; gap: 6px; height: 34px; padding: 0 16px; border-radius: 6px; font-size: 14px; font-weight: 600; border: 1px solid transparent; cursor: pointer; transition: all .15s ease; white-space: nowrap; }
.btn:disabled { opacity: .5; cursor: not-allowed; }
.btn-primary { background: #2B6CD6; color: #fff; }
.btn-primary:hover { background: #4D8BE8; }
.btn-ghost { color: #2B6CD6; background: #EAF2FC; }
.btn-ghost:hover { background: #DCEBFB; }
.btn-light { background: #fff; color: #2B6CD6; }
.btn-sm { height: 28px; padding: 0 12px; font-size: 13px; }
.mt { margin-top: 16px; }
.mt2 { margin-top: 20px; }
.mt8 { margin-top: 8px; }
.muted { color: #5A6577; font-size: 13px; }
.card { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px; box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); }
.section-title { display: flex; align-items: center; gap: 8px; font-size: 15px; font-weight: 600; color: #1F2733; margin-bottom: 14px; }
.section-title .bar { width: 4px; height: 15px; border-radius: 2px; background: #2B6CD6; }
.grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
.kpi-row { display: grid; grid-template-columns: repeat(5, 1fr); gap: 12px; }
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
.bar-bg { background: #EEF3FA; border-radius: 99px; overflow: hidden; }
.bar-bg.tall { height: 8px; }
.bar-fill { height: 100%; border-radius: 99px; transition: width .4s ease; }
.f-blue { background: #2B6CD6; }
.f-orange { background: #FA8C16; }
.f-red { background: #FF4D4F; }
.tip { margin-top: 12px; background: #EAF2FC; border: 1px solid #DCEBFB; color: #1D4FA8; font-size: 13px; border-radius: 6px; padding: 8px 12px; }
.tabs { display: flex; gap: 4px; border-bottom: 1px solid #E3E9F2; margin-bottom: 16px; }
.tabs button { padding: 8px 14px; font-size: 14px; color: #5A6577; font-weight: 500; border-bottom: 2px solid transparent; margin-bottom: -1px; transition: .15s; }
.tabs button.active { color: #2B6CD6; border-bottom-color: #2B6CD6; font-weight: 600; }
@media (max-width: 900px) { .grid-2 { grid-template-columns: 1fr; } .kpi-row { grid-template-columns: repeat(2, 1fr); } }

/* ============ 雷达图 ============ */
.radar-svg { width: 100%; max-width: 320px; display: block; margin: 0 auto; }
.legend { display: flex; gap: 14px; flex-wrap: wrap; font-size: 12px; color: #5A6577; margin-top: 8px; justify-content: center; }
.legend i { display: inline-block; width: 10px; height: 10px; border-radius: 2px; margin-right: 5px; vertical-align: middle; }
.lg-blue { background: #2B6CD6; }
.lg-gray { background: #97A1B2; }

/* ============ 错因分布 ============ */
.err-row { display: flex; align-items: center; gap: 10px; margin-bottom: 10px; }
.err-row:last-of-type { margin-bottom: 0; }
.err-k { width: 120px; font-size: 13px; flex-shrink: 0; }
.err-bar { flex: 1; }
.err-v { width: 30px; text-align: right; font-size: 13px; color: #5A6577; flex-shrink: 0; }
.diag-note { font-size: 13px; line-height: 1.7; color: #5A6577; }

/* ============ 提示列表 / 短板成因 / 重点学生 / 分层干预 ============ */
.ti-list { margin: 0; padding-left: 18px; font-size: 13px; color: #5A6577; line-height: 1.8; list-style: disc; }
.ti-list li { margin-bottom: 8px; }
.sw-item { margin-bottom: 10px; padding-bottom: 10px; border-bottom: 1px dashed #E3E9F2; }
.sw-item:last-child { border-bottom: none; margin-bottom: 0; padding-bottom: 0; }
.sw-k { font-weight: 600; font-size: 13px; }
.sw-v { display: inline-block; min-width: 28px; text-align: center; background: #FFECEC; color: #FF4D4F; border-radius: 6px; font-size: 12px; padding: 1px 6px; margin-left: 4px; }
.sw-cause { font-size: 12px; color: #5A6577; margin-top: 4px; line-height: 1.6; }
.fi { margin-bottom: 10px; padding-bottom: 10px; border-bottom: 1px dashed #E3E9F2; }
.fi:last-child { border-bottom: none; margin-bottom: 0; padding-bottom: 0; }
.fi-h { font-size: 13px; display: flex; align-items: center; gap: 8px; }
.fi-issue { font-size: 12px; color: #5A6577; margin-top: 4px; line-height: 1.6; }
.grp-item { margin-bottom: 12px; padding: 10px 12px; background: #F7FAFD; border-radius: 6px; }
.grp-item:last-child { margin-bottom: 0; }
.grp-h { font-weight: 600; font-size: 13px; }
.grp-mem { display: flex; flex-wrap: wrap; gap: 6px; margin: 6px 0; }
.grp-tip { font-size: 12px; color: #5A6577; line-height: 1.6; }
.itag { font-size: 11px; font-weight: 600; padding: 3px 9px; border-radius: 99px; white-space: nowrap; }
.itag.ideo { color: #1D4FA8; background: #EAF2FC; }
.itag.skill { color: #B26A00; background: #FFF3E0; }
</style>
