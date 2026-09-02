<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">学情分析</span>
      <span class="page-head-sub">增值 Agent · 多元评价与班级学情诊断</span>
    </div>

    <!-- Tab 切换：班级 / 个人 -->
    <div class="tabs">
      <button :class="{ active: tab === 'class' }" @click="tab = 'class'">班级视角</button>
      <button :class="{ active: tab === 'person' }" @click="tab = 'person'">个人视角（林晓）</button>
    </div>

    <!-- ============ 班级视角 ============ -->
    <template v-if="tab === 'class'">
      <!-- 增值 Agent 状态条 -->
      <div v-if="agLoading" class="ag-banner ag-info">增值 Agent 正在分析班级学习数据，请稍候…</div>
      <div v-else-if="agError" class="ag-banner ag-err">
        <span>增值 Agent 分析失败：{{ agError }}</span>
        <button class="btn btn-default btn-sm" @click="fetchAgentReport">重试</button>
      </div>
      <div v-else-if="agReport" class="ag-banner ag-ok">增值 Agent 已生成班级学习增值分析报告（生成时间 {{ agTime }}）
        <button class="btn btn-default btn-sm" @click="fetchAgentReport">重新分析</button>
      </div>

      <!-- 统计行 -->
      <div class="kpi-row" v-if="agRadar.axes.length">
        <div class="kpi"><div class="k">任务完成率</div><div class="v">{{ report ? report.overview.completion_rate : c.completion }}<small>%</small></div></div>
        <div class="kpi"><div class="k">班级平均分</div><div class="v">{{ report ? report.overview.average_score : c.avg }}</div></div>
        <div class="kpi"><div class="k">活跃人数</div><div class="v">{{ report ? report.overview.active_average : c.active }}<small v-if="!report">/{{ c.total }}</small></div></div>
        <div class="kpi"><div class="k">待复核</div><div class="v c-orange">{{ report ? report.overview.review_count : pendingCount }}<small>份</small></div></div>
        <div class="kpi"><div class="k">需重点关注</div><div class="v c-red">{{ report ? report.overview.attention_count : teacher.needsFocus.length }}<small>人</small></div></div>
      </div>

      <!-- 班级能力雷达 + 班级错因分布 -->
      <div class="grid-2 mt">
        <div class="card" v-if="agRadar.axes.length">
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
            <span><i class="lg-gray"></i>目标线({{ agTarget }})</span>
          </div>
        </div>
        <div class="card">
          <div class="section-title"><span class="bar"></span>班级错因分布（人次）</div>
          <div class="err-row" v-for="e in teacher.errorDist" :key="e.k">
            <div class="err-k">{{ e.k }}</div>
            <div class="bar-bg tall err-bar"><div class="bar-fill f-orange" :style="{ width: e.v / maxError * 100 + '%' }"></div></div>
            <div class="err-v">{{ e.v }}<span v-if="agReport" class="muted"> · {{ e.percentage }}%</span></div>
          </div>
          <div class="section-title mt2"><span class="bar"></span>诊断结论</div>
          <p v-if="agReport" class="diag-note">{{ (agReport.error_diagnosis && agReport.error_diagnosis.diagnosis) || "暂无诊断结论" }}</p>
          <p v-else class="diag-note">高频错因为「{{ topErrors[0].k }}」与「{{ topErrors[1].k }}」，说明学生在公式应用与业务解释环节最薄弱，需在后续任务中重点干预。</p>
        </div>
      </div>

      <!-- 优势分析 + 能力短板及成因 -->
      <div class="grid-2 mt">
        <div class="card">
          <div class="section-title"><span class="bar"></span>班级优势<span v-if="agReport" class="muted ag-src">增值 Agent 生成</span></div>
          <template v-if="agReport">
            <div class="sw-item" v-for="s in agReport.strengths" :key="'st' + s.dimension">
              <div class="sw-k">{{ s.dimension }} <span class="sw-v sw-good">{{ s.score }}</span></div>
              <div class="sw-cause">数据依据：{{ s.evidence }}</div>
              <div class="sw-cause">优势分析：{{ s.analysis }}</div>
            </div>
            <div v-if="!agReport.strengths.length" class="muted">暂无达标优势维度</div>
          </template>
          <ul v-else class="ti-list">
            <li v-for="s in sw.strengths" :key="s.k"><b>{{ s.k }}</b> <span class="muted">（均值 {{ s.v }}）</span>：整体掌握较好，可据此设计进阶与拓展任务，并让该生承担小组示范。</li>
          </ul>
        </div>
        <div class="card">
          <div class="section-title"><span class="bar"></span>能力短板及成因<span v-if="agReport" class="muted ag-src">增值 Agent 生成</span></div>
          <template v-if="agReport">
            <div class="sw-item" v-for="w in agReport.weaknesses" :key="'wk' + w.dimension">
              <div class="sw-k">{{ w.dimension }} <span class="sw-v">{{ w.score }}</span></div>
              <div class="sw-cause">形成原因：{{ w.cause }}</div>
              <div class="sw-cause">数据依据：{{ w.evidence }}</div>
            </div>
            <div v-if="!agReport.weaknesses.length" class="muted">暂无低于目标线的能力短板</div>
          </template>
          <template v-else>
            <div class="sw-item" v-for="w in sw.weaknesses" :key="w.k">
              <div class="sw-k">{{ w.k }} <span class="sw-v">{{ w.v }}</span></div>
              <div class="sw-cause">形成原因：{{ teacher.cause.classCause[w.k] || "相关训练不足，需针对性补练。" }}</div>
            </div>
          </template>
        </div>
      </div>

      <!-- 教学建议 -->
      <div class="card mt">
        <div class="section-title"><span class="bar"></span>教学建议（针对短板）<span v-if="agReport" class="muted ag-src">增值 Agent 生成</span></div>
        <ul class="ti-list">
          <template v-if="agReport">
            <li v-for="g in agReport.teaching_suggestions" :key="'ts' + g.target_dimension"><b>{{ g.target_dimension }}</b>：{{ g.suggestion }}<div class="sw-cause">当前问题：{{ g.problem }}</div><div class="sw-cause">实施方式：{{ g.implementation }}</div></li>
            <li v-if="!agReport.teaching_suggestions.length" class="muted">暂无教学建议</li>
          </template>
          <template v-else>
            <li v-for="w in sw.weaknesses" :key="'g' + w.k"><b>{{ w.k }}</b>：{{ teacher.cause.classSugg[w.k] || "建议针对性推送练习并安排一对一辅导。" }}</li>
          </template>
        </ul>
      </div>

      <!-- 重点关注学生 + 分层干预建议 -->
      <div class="grid-2 mt">
        <div class="card">
          <div class="section-title"><span class="bar"></span>重点关注学生</div>
          <template v-if="agReport">
            <div class="fi" v-for="f in agReport.attention_students" :key="'at' + f.student_id">
              <div class="fi-h">
                <b>{{ f.student_name }}</b>
                <span class="tag tag-red">重点关注</span>
              </div>
              <div class="fi-issue">原因：{{ f.reason }}</div>
              <div class="fi-issue">干预建议：{{ f.intervention }}</div>
            </div>
            <div v-if="!agReport.attention_students.length" class="muted">暂无需重点关注的学生</div>
          </template>
          <template v-else>
            <div class="fi" v-for="f in teacher.needsFocus" :key="f.nm">
              <div class="fi-h">
                <b>{{ f.nm }}</b>
                <span class="tag tag-red">{{ f.score === null ? "未提交" : f.score + "分" }}</span>
              </div>
              <div class="fi-issue">{{ f.issue }}</div>
            </div>
          </template>
        </div>
        <div class="card">
          <div class="section-title"><span class="bar"></span>分层干预建议</div>
          <template v-if="agReport">
            <div class="grp-item" v-for="g in agGroups" :key="'ag' + g.key">
              <div class="grp-h">{{ g.label }} <span class="muted">（{{ g.data.student_count }} 人）</span></div>
              <div class="grp-strategy">策略：{{ g.data.strategy }}</div>
              <div class="grp-tip">干预措施：{{ g.data.actions.join("；") }}</div>
            </div>
          </template>
          <template v-else>
            <div class="grp-item" v-for="g in groups" :key="g.k">
              <div class="grp-h">{{ g.k }} <span class="muted">（{{ g.mem.length }} 人）</span></div>
              <div class="grp-mem">
                <span class="itag skill" v-for="m in g.mem" :key="m">{{ m }}</span>
                <span v-if="!g.mem.length" class="muted">—</span>
              </div>
              <div class="grp-tip">干预建议：{{ g.tip }}</div>
            </div>
          </template>
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
      <div class="section-title"><span class="bar"></span>增值 Agent · 班级学习增值分析</div>
      <p class="muted">对班级整体学习数据进行聚合分析，自动生成班级学习画像、能力短板诊断及教学干预建议（即上方班级视角报告）。</p>
      <button class="btn btn-primary mt8" :disabled="agLoading" @click="fetchAgentReport">{{ agLoading ? "分析中…" : (agReport ? "重新分析" : "生成增值分析报告") }}</button>
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
const tab = ref("class");
const tip = ref("");
let tipTimer = null;
function showTip(t) { tip.value = t; clearTimeout(tipTimer); tipTimer = setTimeout(function () { tip.value = ""; }, 2400); }

// ============ 增值 Agent 对接（班级学习增值分析） ============
const AI_URL = "https://agent.gjt-smart.com/v1/chat-messages";
const AI_KEY = "app-F3bPKGDUbcNcJjH7JLmvJSwf";

// 将页面现有教学数据转换为接口要求的 students 结构，缺失的能力分项按班级能力均值补齐（演示值）
function buildClassData(t) {
  const axes = t.radar.axes, avg = t.radar.avg;
  const focusMap = {};
  (t.needsFocus || []).forEach(function (f) { focusMap[f.nm] = f; });
  function makeStudent(name, idx, score, errors, attention, reviewStatus) {
    const ability = {};
    axes.forEach(function (k, i) { ability[k] = Math.max(0, Math.min(100, Math.round((avg[i] || 70) + ((idx + i) % 5) - 2))); });
    const s = score == null ? 0 : score;
    return {
      student_id: "S" + String(idx + 1).padStart(3, "0"),
      student_name: name,
      completion: reviewStatus !== "未提交",
      score: score == null ? 0 : score,
      active: score == null ? 2 : (s >= 85 ? 9 : s >= 70 ? 7 : 5),
      review_status: reviewStatus,
      attention: attention,
      ability_scores: ability,
      error_reasons: errors || []
    };
  }
  const students = [], seen = {};
  (t.submissions || []).forEach(function (sb, i) {
    students.push(makeStudent(sb.student, i, sb.ai, sb.errors, focusMap[sb.student] != null, sb.status === "待复核" ? "待复核" : "正常"));
    seen[sb.student] = 1;
  });
  (t.needsFocus || []).forEach(function (f) {
    if (seen[f.nm]) return;
    const notSubmit = f.score == null;
    students.push(makeStudent(f.nm, students.length, notSubmit ? null : f.score, [], true, notSubmit ? "未提交" : "待复核"));
  });
  return {
    class_id: "CWSZ2433",
    class_name: t.cls,
    course_name: "财务大数据分析",
    target_score: 85,
    students: students
  };
}

async function callValueAgent(classData) {
  const ctrl = typeof AbortController !== "undefined" ? new AbortController() : null;
  const timer = ctrl ? setTimeout(function () { ctrl.abort(); }, 60000) : null;
  let resp;
  try {
    resp = await fetch(AI_URL, {
      method: "POST",
      headers: { "Authorization": "Bearer " + AI_KEY, "Content-Type": "application/json" },
      body: JSON.stringify({
        inputs: { class_data_json: JSON.stringify(classData) },
        query: "请对以上班级学习数据进行增值分析",
        response_mode: "blocking",
        user: "teacher-vue"
      }),
      signal: ctrl ? ctrl.signal : undefined
    });
  } finally { if (timer) clearTimeout(timer); }
  if (!resp.ok) throw new Error("接口请求失败（HTTP " + resp.status + "）");
  const data = await resp.json();
  const text = data && data.answer;
  if (!text) throw new Error("Agent 未返回分析结果");
  return text;
}

// Agent 回答可能包裹 markdown 代码块，做兼容解析；解析失败时截取最外层 JSON 片段
function parseAgentJson(text) {
  let s = String(text).trim();
  const fence = s.match(/```(?:json)?\s*([\s\S]*?)```/i);
  if (fence) s = fence[1].trim();
  try { return JSON.parse(s); } catch (e) { /* 继续兜底截取 */ }
  const start = s.search(/[\{\[]/);
  if (start < 0) throw new Error("Agent 返回内容无法解析为 JSON");
  const open = s.charAt(start), close = open === "{" ? "}" : "]";
  let depth = 0, inStr = false, esc = false;
  for (let i = start; i < s.length; i++) {
    const ch = s.charAt(i);
    if (inStr) { if (esc) esc = false; else if (ch === "\\") esc = true; else if (ch === '"') inStr = false; continue; }
    if (ch === '"') inStr = true; else if (ch === open) depth++;
    else if (ch === close) { depth--; if (depth === 0) return JSON.parse(s.slice(start, i + 1)); }
  }
  throw new Error("Agent 返回内容无法解析为 JSON");
}

// ============ 增值 Agent 状态与报告 ============
const agReport = ref(null);
const agLoading = ref(false);
const agError = ref("");
const agTime = ref("");
let agSeq = 0;
const report = computed(function () { return agReport.value; });
async function fetchAgentReport() {
  const seq = ++agSeq;
  agLoading.value = true; agError.value = "";
  try {
    const text = await callValueAgent(buildClassData(teacher.value));
    const result = parseAgentJson(text);
    if (seq !== agSeq) return;
    if (!result || !result.overview) throw new Error("Agent 返回数据结构不完整");
    agReport.value = result;
    const d = new Date();
    agTime.value = (d.getMonth() + 1) + "-" + d.getDate() + " " + String(d.getHours()).padStart(2, "0") + ":" + String(d.getMinutes()).padStart(2, "0");
    showTip("增值分析报告已生成");
  } catch (e) {
    if (seq !== agSeq) return;
    agError.value = (e && e.name === "AbortError") ? "请求超时（60 秒），请稍后重试" : ((e && e.message) || "请求失败");
  } finally { if (seq === agSeq) agLoading.value = false; }
}

// ============ 计算属性 ============
const c = computed(function () { return teacher.value.classStats; });
const pendingCount = computed(function () { return teacher.value.submissions.filter(function (s) { return s.status === "待复核"; }).length; });
const targetLine = computed(function () { return teacher.value.radar.axes.map(function () { return 85; }); });
const agTarget = computed(function () {
  if (agReport.value && agReport.value.ability_radar && agReport.value.ability_radar.length) return agReport.value.ability_radar[0].target;
  return 85;
});
const agGroups = computed(function () {
  if (!agReport.value || !agReport.value.layered_intervention) return [];
  const li = agReport.value.layered_intervention;
  return [
    { key: "excellent", label: "优秀层", data: li.excellent },
    { key: "middle", label: "中等层", data: li.middle },
    { key: "need_improvement", label: "待提升层", data: li.need_improvement },
    { key: "not_submitted", label: "未提交层", data: li.not_submitted }
  ].filter(function (g) { return g.data; });
});
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

const agRadar = computed(function () {
  if (agReport.value && Array.isArray(agReport.value.ability_radar) && agReport.value.ability_radar.length) {
    return {
      axes: agReport.value.ability_radar.map(function (r) { return r.dimension; }),
      cur: agReport.value.ability_radar.map(function (r) { return r.score; }),
      compare: agReport.value.ability_radar.map(function (r) { return r.target; })
    };
  }
  return { axes: teacher.value.radar.axes, cur: teacher.value.radar.avg, compare: targetLine.value };
});
const classRadar = computed(function () { return agRadar.value; });
const classEnds = computed(function () { return radarEnds(classRadar.value.axes.length); });
const personRadar = computed(function () { return { axes: teacher.value.person.axes, cur: teacher.value.person.cur, compare: teacher.value.radar.avg }; });
const personEnds = computed(function () { return radarEnds(personRadar.value.axes.length); });

// ============ 交互 ============
function go(view) { emit("navigate", view); }

// 进入页面即调用增值 Agent 生成班级增值分析报告；失败时页面保留本地预置数据并提示错误
onMounted(function () { fetchAgentReport(); });
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

/* ============ 增值 Agent 状态条 ============ */
.ag-banner { display: flex; align-items: center; gap: 10px; font-size: 13px; border-radius: 6px; padding: 9px 12px; margin-bottom: 14px; }
.ag-info { background: #EAF2FC; border: 1px solid #DCEBFB; color: #1D4FA8; }
.ag-ok { background: #F1FAE8; border: 1px solid #D9F0C4; color: #3C8E12; }
.ag-err { background: #FFECEC; border: 1px solid #FFD6D6; color: #D93025; }
.ag-banner .btn { margin-left: auto; }
.ag-src { font-weight: 400; }
.sw-v.sw-good { background: #F1FAE8; color: #3C8E12; }
.grp-strategy { font-size: 12px; color: #1F2733; margin: 6px 0 4px; line-height: 1.6; }

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
