<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">评分复核</span>
      <span class="page-head-sub">AI 自动评分 + 教师人工复核，双轨保障评价准确性</span>
    </div>

    <!-- 学生提交记录 -->
    <div class="card">
      <div class="section-title"><span class="bar"></span>学生提交记录<span class="count">待复核 {{ pendingCount }} 份</span></div>
      <div class="table-wrap">
        <table class="table">
          <thead>
            <tr><th>学生</th><th>任务</th><th>AI评分</th><th>错因标签</th><th>状态</th><th>操作</th></tr>
          </thead>
          <tbody>
            <tr v-for="s in teacher.submissions" :key="s.id">
              <td class="td-name">{{ s.student }}</td>
              <td>{{ s.task }}</td>
              <td><b :class="scoreCls(s.ai)">AI {{ s.ai }}</b></td>
              <td>
                <template v-if="s.errors.length"><span class="err-tag" v-for="e in s.errors" :key="e">{{ e }}</span></template>
                <span v-else class="muted">无</span>
              </td>
              <td>
                <span class="tag" :class="s.status === '待复核' ? 'tag-red' : 'tag-green'">{{ s.status }}</span>
              </td>
              <td>
                <button class="btn btn-default btn-sm" @click="openReview(s)">
                  {{ s.status === '待复核' ? '复核' : '查看' }}
                </button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- 复核弹窗 -->
    <div class="mask" v-if="modal.show" @click.self="closeReview">
      <div class="modal">
        <div class="m-h">
          评分复核 · {{ modal.s.student }}
          <button class="x-btn" @click="closeReview">✕</button>
        </div>
        <div class="m-b">
          <div class="rv-meta">
            <div class="rv-item"><span class="rv-k">任务</span><span>{{ modal.s.task }}</span></div>
            <div class="rv-item"><span class="rv-k">AI 评分</span><b>{{ modal.s.ai }} 分</b></div>
            <div class="rv-item"><span class="rv-k">状态</span>
              <span class="tag" :class="modal.s.status === '待复核' ? 'tag-red' : 'tag-green'">{{ modal.s.status }}</span>
            </div>
          </div>

          <div class="field">
            <label>AI 诊断的错因标签</label>
            <div>
              <template v-if="modal.s.errors.length"><span class="err-tag" v-for="e in modal.s.errors" :key="e">{{ e }}</span></template>
              <span v-else class="tag tag-green">无显著错因</span>
            </div>
          </div>

          <div class="field">
            <label>教师复核评分（{{ modal.s.ai }} → ）</label>
            <div class="score-adjust">
              <button class="adj-btn" @click="adjust(-5)">−5</button>
              <div class="adj-val">{{ finalScore }}</div>
              <button class="adj-btn" @click="adjust(5)">+5</button>
              <span class="muted hint-inline">微调 AI 评分，最终成绩以教师复核为准</span>
            </div>
          </div>

          <div class="field">
            <label>复核意见（学生可见）</label>
            <textarea v-model="modal.note" placeholder="如：结论正确但业务解释不足，建议补充产品维度对比。"></textarea>
          </div>
        </div>
        <div class="m-f">
          <button class="btn btn-default" @click="closeReview">取消</button>
          <button class="btn btn-primary" @click="confirmReview">确认复核</button>
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
const LS_TEACHER = "fd_teacher_v2";

const PRESET_TEACHER = {
  cls: "财务2433",
  tasks: [
    { id: "T01", name: "认知利润表", level: "初级", diff: "易", ability: "盈利能力分析", status: "已发布", subs: 34, skills: ["利润表认知"], ideos: ["大国工匠"] },
    { id: "T02", name: "盈利能力建模分析", level: "中级", diff: "中", ability: "盈利能力分析", status: "已发布", subs: 29, skills: ["盈利能力建模分析"], ideos: ["课程思政教学案例"] },
    { id: "T03", name: "盈利能力下滑问题追溯与管理建议", level: "高级", diff: "难", ability: "盈利能力分析", status: "已发布", subs: 15, skills: ["盈利下滑追溯", "管理建议撰写"], ideos: ["行业伦理"] },
    { id: "T04", name: "认知资产负债表", level: "初级", diff: "易", ability: "财务状况评估", status: "已发布", subs: 33, skills: ["资产负债表结构分析"], ideos: ["红色财经资源"] },
    { id: "T05", name: "营运能力分析", level: "中级", diff: "中", ability: "财务状况评估", status: "已发布", subs: 26, skills: ["营运能力分析"], ideos: [] },
    { id: "T06", name: "偿债能力与营运能力综合分析", level: "中级", diff: "中", ability: "财务状况评估", status: "已发布", subs: 22, skills: ["偿债能力分析", "营运能力分析"], ideos: ["政策文件"] },
    { id: "T07", name: "偿债能力与营运能力管理建议", level: "高级", diff: "难", ability: "财务状况评估", status: "已发布", subs: 11, skills: ["管理建议撰写"], ideos: ["行业楷模"] },
    { id: "T08", name: "认知现金流量表", level: "初级", diff: "易", ability: "现金流诊断", status: "已发布", subs: 30, skills: ["现金流量表解读"], ideos: [] },
    { id: "T09", name: "现金流建模分析", level: "中级", diff: "中", ability: "现金流诊断", status: "草稿", subs: 0, skills: ["现金流建模分析"], ideos: [] },
    { id: "T10", name: "现金流深度分析与财务困境", level: "高级", diff: "难", ability: "现金流诊断", status: "草稿", subs: 0, skills: ["财务困境诊断"], ideos: ["行业伦理"] },
    { id: "T11", name: "撰写财务速览备忘录", level: "初级", diff: "易", ability: "综合财务分析", status: "已发布", subs: 27, skills: ["财务速览备忘录"], ideos: ["专业发展史"] },
    { id: "T12", name: "综合财务分析看板设计", level: "中级", diff: "中", ability: "综合财务分析", status: "已发布", subs: 18, skills: ["综合看板设计"], ideos: ["科技自立自强案例"] },
    { id: "T13", name: "风险洞察与管理建议报告", level: "高级", diff: "难", ability: "综合财务分析", status: "草稿", subs: 0, skills: ["风险洞察", "管理建议撰写"], ideos: ["政策文件"] }
  ],
  submissions: [
    { id: 1, student: "王浩", task: "T05 营运能力分析", ai: 76, status: "待复核", errors: ["结论缺少业务逻辑", "图表选择不当"] },
    { id: 2, student: "陈雨", task: "T03 盈利能力下滑问题追溯", ai: 91, status: "已复核", errors: [] },
    { id: 3, student: "李娜", task: "T06 偿债与营运能力综合", ai: 68, status: "待复核", errors: ["公式错误", "指标理解错误"] },
    { id: 4, student: "赵磊", task: "T07 偿债与营运管理建议", ai: 83, status: "待复核", errors: ["流动比率计算口径不一致"] }
  ],
  classStats: { completion: 82, avg: 79, active: 31, total: 38 },
  weeklySubs: 16,
  errorDist: [
    { k: "公式错误", v: 18 }, { k: "指标理解错误", v: 26 }, { k: "图表选择不当", v: 14 },
    { k: "结论缺业务逻辑", v: 22 }, { k: "分析不全面", v: 11 }
  ],
  radar: {
    axes: ["盈利能力分析", "财务状况评估", "现金流诊断", "综合财务分析"],
    cur: [82, 76, 68, 60], avg: [80, 74, 70, 62]
  },
  needsFocus: [
    { nm: "李娜", score: 68, status: "待复核", issue: "T06 综合实验公式错误 + 指标理解偏差，建议一对一辅导" },
    { nm: "赵磊", score: 83, status: "待复核", issue: "T07 流动比率计算口径不一致，需确认是否剔除预付款项" },
    { nm: "周婷", score: null, status: "未提交", issue: "本周未提交任何任务，活跃度明显下降，需提醒" }
  ],
  activity: [
    { icon: "upload", txt: "李娜 提交了「T06 偿债能力与营运能力综合」", time: "10 分钟前" },
    { icon: "check", txt: "陈雨 的「T03 盈利能力下滑问题追溯」复核通过", time: "1 小时前" },
    { icon: "bot", txt: "AI 完成 4 份作业自动评分", time: "今天" },
    { icon: "warn", txt: "赵磊「T07 偿债与营运管理建议」触发薄弱能力预警", time: "昨天" }
  ],
  cause: {
    classCause: {
      "盈利能力分析": "盈利能力建模分析多停留在指标计算层面，毛利率/净利率变动的归因追溯专项训练不足。",
      "财务状况评估": "偿债与营运能力指标分开计算，缺乏统一口径与两类能力交叉验证的训练。",
      "现金流诊断": "重利润、轻现金，现金流量表结构理解与现金流建模分析的训练欠缺。",
      "综合财务分析": "综合输出能力偏弱，备忘录、看板与风险洞察报告的撰写训练不足。"
    },
    classSugg: {
      "盈利能力分析": "增设盈利能力下滑归因任务，强化「指标计算→归因→管理建议」完整链路。",
      "财务状况评估": "设计偿债能力与营运能力综合实验，统一指标口径并做交叉验证。",
      "现金流诊断": "增设现金流量表结构解读与现金流建模任务，与利润质量联动解读。",
      "综合财务分析": "以财务速览备忘录与看板设计为载体，训练风险洞察与管理建议报告撰写。"
    },
    personCause: {
      "盈利能力分析": "指标计算扎实，但从盈利建模到归因分析的链路不完整，结论缺管理语言。",
      "财务状况评估": "偿债/营运指标会算，但交叉验证经验少，指标口径偶有不一致。",
      "现金流诊断": "现金流建模分析练习偏少，对「造血」能力判断标准理解不深。",
      "综合财务分析": "能完成单项分析，但备忘录/看板/报告式的综合输出不足。"
    },
    personSugg: {
      "盈利能力分析": "强化盈利下滑场景的归因分析与管理语言撰写训练。",
      "财务状况评估": "增加偿债×营运交叉验证综合实验，配套口径核对清单。",
      "现金流诊断": "系统完成现金流建模与财务困境诊断任务。",
      "综合财务分析": "以财务速览备忘录与综合看板设计为载体做综合输出训练。"
    }
  },
  person: {
    axes: ["盈利能力分析", "财务状况评估", "现金流诊断", "综合财务分析"],
    cur: [88, 82, 68, 60]
  },
  personErrors: [["公式错误", 4], ["指标理解错误", 6], ["图表选择不当", 3], ["结论缺业务逻辑", 5], ["分析不全面", 2]]
};

function loadTeacher() {
  const base = JSON.parse(JSON.stringify(PRESET_TEACHER));
  const raw = lsGet(LS_TEACHER) || {};
  const t = Object.assign({}, base, raw);
  // 班级已统一为财务2433
  t.cls = base.cls;
  if (base.cause) t.cause = Object.assign({}, base.cause, t.cause || {});
  if (base.radar) t.radar = Object.assign({}, base.radar, t.radar || {});
  if (!t.tasks && base.tasks) t.tasks = base.tasks;
  if (!t.submissions && base.submissions) t.submissions = base.submissions;
  if (!t.errorDist && base.errorDist) t.errorDist = base.errorDist;
  if (!Array.isArray(t.activity) && base.activity) t.activity = base.activity;
  lsSet(LS_TEACHER, t);
  return t;
}

// ============ 页面状态 ============
const teacher = ref(loadTeacher());
const modal = ref({ show: false, s: null, score: 0, note: "" });
const tip = ref("");
let tipTimer = null;
function showTip(t) { tip.value = t; clearTimeout(tipTimer); tipTimer = setTimeout(function () { tip.value = ""; }, 2400); }

// ============ AI 占位 ============
const AI_URL = "https://agent.gjt-smart.com/v1/chat-messages";
const AI_KEY = "app-xxxxxx"; // TODO 占位

// ============ 计算属性 ============
const pendingCount = computed(function () { return teacher.value.submissions.filter(function (s) { return s.status === "待复核"; }).length; });
const finalScore = computed(function () {
  return Math.max(0, Math.min(100, (modal.value.s ? modal.value.s.ai : 0) + modal.value.score));
});

// ============ 交互 ============
function go(view) { emit("navigate", view); }

function scoreCls(v) { return v >= 85 ? "hi" : v >= 70 ? "mid" : "lo"; }
function openReview(s) { modal.value = { show: true, s: s, score: 0, note: "" }; }
function closeReview() { modal.value.show = false; }
function adjust(d) { modal.value.score += d; }
function confirmReview() {
  const s = modal.value.s;
  s.status = "已复核";
  s.final = finalScore.value;
  s.note = modal.value.note;
  lsSet(LS_TEACHER, teacher.value);
  showTip(s.student + " 复核完成：" + finalScore.value + " 分");
  closeReview();
}

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
.btn-default { background: #fff; border-color: #E3E9F2; color: #1F2733; }
.btn-default:hover { border-color: #2B6CD6; color: #2B6CD6; }
.btn-sm { height: 28px; padding: 0 12px; font-size: 13px; }
.muted { color: #5A6577; font-size: 13px; }
.card { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px; box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); }
.section-title { display: flex; align-items: center; gap: 8px; font-size: 15px; font-weight: 600; color: #1F2733; margin-bottom: 14px; }
.section-title .bar { width: 4px; height: 15px; border-radius: 2px; background: #2B6CD6; }
.section-title .count { margin-left: auto; font-size: 12px; font-weight: 400; color: #97A1B2; }
.tag { display: inline-flex; align-items: center; font-size: 12px; padding: 1px 8px; border-radius: 4px; font-weight: 500; white-space: nowrap; }
.tag-red { background: #FFECEC; color: #D93025; }
.tag-green { background: #F1FAE8; color: #3C8E12; }
.tag-blue { background: #EAF2FC; color: #1D4FA8; }
.tip { margin-top: 12px; background: #EAF2FC; border: 1px solid #DCEBFB; color: #1D4FA8; font-size: 13px; border-radius: 6px; padding: 8px 12px; }

/* ============ 表格 ============ */
.table-wrap { overflow-x: auto; }
.table { width: 100%; border-collapse: collapse; font-size: 13px; }
.table th { background: #F7FAFD; text-align: left; padding: 11px 12px; font-weight: 600; color: #1F2733; border-bottom: 1px solid #E3E9F2; white-space: nowrap; }
.table td { padding: 11px 12px; border-bottom: 1px solid #EEF2F8; color: #1F2733; }
.table tbody tr { transition: background .15s; }
.table tbody tr:hover td { background: #F7FAFD; }
.td-name { font-weight: 600; }
b.hi { color: #52C41A; }
b.mid { color: #2B6CD6; }
b.lo { color: #FF4D4F; }
.err-tag { display: inline-block; background: #FFECEC; color: #D93025; font-size: 12px; padding: 2px 8px; border-radius: 99px; margin: 2px; }

/* ============ 弹窗 ============ */
.mask { position: fixed; inset: 0; background: rgba(16, 24, 40, .45); z-index: 50; display: flex; align-items: center; justify-content: center; padding: 20px; animation: maskFade .2s; }
@keyframes maskFade { from { opacity: 0; } to { opacity: 1; } }
.modal { background: #fff; border-radius: 8px; width: 480px; max-width: 100%; max-height: 88vh; overflow: auto; box-shadow: 0 6px 16px rgba(16, 38, 76, .10), 0 3px 6px rgba(16, 38, 76, .06); animation: modalIn .22s cubic-bezier(.3, 1.2, .5, 1); }
@keyframes modalIn { from { opacity: 0; transform: translateY(14px) scale(.97); } to { opacity: 1; transform: none; } }
.modal .m-h { padding: 18px 22px; border-bottom: 1px solid #E3E9F2; font-size: 16px; font-weight: 600; display: flex; justify-content: space-between; align-items: center; }
.modal .m-b { padding: 20px 22px; }
.modal .m-f { padding: 14px 22px; border-top: 1px solid #E3E9F2; display: flex; justify-content: flex-end; gap: 10px; }
.x-btn { width: 28px; height: 28px; border-radius: 6px; color: #97A1B2; display: flex; align-items: center; justify-content: center; transition: .15s; }
.x-btn:hover { background: #F7FAFD; color: #1F2733; }

/* ============ 表单 ============ */
.field { margin-bottom: 14px; }
.field label { display: block; font-size: 13px; font-weight: 600; margin-bottom: 6px; }
.field input, .field select, .field textarea { width: 100%; height: 34px; border: 1px solid #E3E9F2; border-radius: 6px; padding: 0 11px; background: #fff; color: #1F2733; transition: border-color .15s, box-shadow .15s; }
.field textarea { height: auto; padding: 8px 11px; resize: vertical; min-height: 70px; }
.field input:focus, .field select:focus, .field textarea:focus { outline: none; border-color: #2B6CD6; box-shadow: 0 0 0 2px #EAF2FC; }

/* ============ 复核详情 ============ */
.rv-meta { display: flex; gap: 18px; flex-wrap: wrap; margin-bottom: 16px; background: #F7FAFD; border-radius: 6px; padding: 11px 14px; }
.rv-item { font-size: 13px; display: flex; align-items: center; gap: 7px; }
.rv-k { color: #97A1B2; font-size: 12px; }
.score-adjust { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }
.adj-btn { width: 34px; height: 34px; border-radius: 6px; border: 1px solid #E3E9F2; color: #5A6577; font-size: 15px; font-weight: 600; transition: .15s; }
.adj-btn:hover { border-color: #2B6CD6; color: #2B6CD6; }
.adj-val { min-width: 64px; text-align: center; font-size: 20px; font-weight: 700; color: #2B6CD6; background: #EAF2FC; border-radius: 6px; padding: 4px 0; font-variant-numeric: tabular-nums; }
.hint-inline { font-size: 12px; margin-left: 4px; }
</style>
