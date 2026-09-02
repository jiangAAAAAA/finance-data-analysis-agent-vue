<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">任务管理</span>
      <span class="page-head-sub">共 {{ teacher.tasks.length }} 个任务 · 支持新增、编辑、删除与发布</span>
      <button class="btn btn-primary btn-sm" @click="openModal(null)">新增任务</button>
    </div>

    <!-- 任务列表 -->
    <div class="card">
      <div class="section-title"><span class="bar"></span>任务列表</div>
      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>任务</th><th>层级</th><th>对应能力</th><th>难度</th>
              <th>提交数</th><th>状态</th><th>思政/技能标签</th><th>操作</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="t in teacher.tasks" :key="t.id">
              <td class="task-name"><b>{{ t.id }}</b> {{ t.name }}</td>
              <td><span class="tag" :class="lvTag(t.level)">{{ t.level }}</span></td>
              <td>{{ t.ability }}</td>
              <td>{{ t.diff }}</td>
              <td>{{ t.subs }} 份</td>
              <td><span class="tag" :class="t.status === '已发布' ? 'tag-green' : 'tag-gray'">{{ t.status }}</span></td>
              <td>
                <div class="tag-cell">
                  <span v-for="x in (t.skills || []).slice(0, 2)" :key="x" class="tag tag-orange">{{ x }}</span>
                  <span v-for="x in (t.ideos || []).slice(0, 1)" :key="'i' + x" class="tag tag-blue">{{ x }}</span>
                  <span v-if="tagCount(t) === 0" class="muted">—</span>
                  <span v-if="tagCount(t) > 3" class="more-tag">+{{ tagCount(t) - 3 }}</span>
                </div>
              </td>
              <td>
                <div class="act">
                  <button class="btn btn-default btn-sm" @click="openModal(t)">编辑</button>
                  <button class="btn btn-default btn-sm del" @click="delTask(t)">删除</button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- 新增 / 编辑弹窗 -->
    <div v-if="modal.show" class="mask" @click.self="closeModal">
      <div class="modal">
        <div class="m-h">
          {{ modal.isEdit ? '编辑任务' : '新增任务' }}
          <button class="x-btn" @click="closeModal">✕</button>
        </div>
        <div class="m-b">
          <div class="field" :class="{ invalid: !modal.valid }">
            <label>任务名称</label>
            <input v-model="modal.form.name" placeholder="如：现金流建模分析" />
            <div class="err">请输入任务名称</div>
          </div>
          <div class="field">
            <label>任务内容</label>
            <textarea v-model="modal.form.content" rows="3" placeholder="描述任务内容、数据范围与要求，AI 将结合任务名称识别标签"></textarea>
          </div>
          <div class="form-row">
            <div class="field">
              <label>层级</label>
              <select v-model="modal.form.level">
                <option>初级</option><option>中级</option><option>高级</option>
              </select>
            </div>
            <div class="field">
              <label>难度</label>
              <select v-model="modal.form.diff">
                <option>易</option><option>中</option><option>难</option>
              </select>
            </div>
          </div>
          <div class="field">
            <label>对应能力</label>
            <input v-model="modal.form.ability" placeholder="如：财务状况评估" />
          </div>
          <div class="field">
            <label>状态</label>
            <select v-model="modal.form.status">
              <option>草稿</option><option>已发布</option>
            </select>
          </div>
          <div class="field">
            <label>技能知识标签</label>
            <div class="tsel-wrap">
              <span
                v-for="x in SKILL_TAGS" :key="x"
                class="tsel" :class="{ on: modal.form.skills.includes(x) }"
                @click="toggleTag('skills', x)"
              >{{ x }}</span>
            </div>
          </div>
          <div class="field ai-field">
            <button class="btn btn-ghost btn-sm" :disabled="aiLoading" @click="aiTags">{{ aiLoading ? '识别中…' : 'AI 一键识别标签' }}</button>
            <span class="hint">依据任务名称与任务内容自动生成技能与思政标签</span>
          </div>
          <div class="field">
            <label>思政资源标签</label>
            <div class="tsel-wrap">
              <span
                v-for="x in IDEOLOGY_TAGS" :key="'id' + x"
                class="tsel" :class="{ on: modal.form.ideos.includes(x) }"
                @click="toggleTag('ideos', x)"
              >{{ x }}</span>
            </div>
          </div>
        </div>
        <div class="m-f">
          <button class="btn btn-default" @click="closeModal">取消</button>
          <button class="btn btn-primary" @click="saveTask">保存</button>
        </div>
      </div>
    </div>

    <!-- 提示 -->
    <div v-if="tip" class="tip">{{ tip }}</div>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";

const emit = defineEmits(["navigate"]);

// ============ 数据服务（localStorage 共享，PRESET 兜底） ============
function lsGet(k) { try { const v = localStorage.getItem(k); return v ? JSON.parse(v) : null; } catch (e) { return null; } }
function lsSet(k, v) { try { localStorage.setItem(k, JSON.stringify(v)); } catch (e) { /* 忽略 */ } }
const LS_TEACHER = "fd_teacher_v2";

// ============ AI 智能体配置（立德 Agent · Dify 风格接口） ============
const AI_URL = "https://agent.gjt-smart.com/v1/chat-messages";
const AI_KEY = "app-xplYEWYKIQmzCnudOlDlGHnD";
const AI_USER = "task-manager";

const SKILL_TAGS = ['利润表认知', '盈利能力建模分析', '盈利下滑追溯', '资产负债表结构分析', '营运能力分析', '偿债能力分析', '现金流量表解读', '现金流建模分析', '财务困境诊断', '财务速览备忘录', '综合看板设计', '风险洞察', '管理建议撰写'];
const IDEOLOGY_TAGS = ['大国工匠', '行业楷模', '科技自立自强案例', '行业伦理', '红色财经资源', '政策文件', '专业发展史', '课程思政教学案例'];

const PRESET_TEACHER = {
  name: "陈老师",
  cls: "财务2433",
  classStats: { completion: 82, avg: 79, active: 31, total: 38 },
  weeklySubs: 16,
  completionBars: [
    ["T01 利润表认知", 93], ["T05 营运能力分析", 88], ["T12 综合看板设计", 75]
  ],
  tasks: [
    { id: "T01", name: "认知利润表", content: "认知利润表基本结构，完成「财务费用求证」小实验，理解各利润项目的计算路径。", level: "初级", diff: "易", ability: "盈利能力分析", status: "已发布", subs: 34, skills: ["利润表认知"], ideos: ["大国工匠"] },
    { id: "T02", name: "盈利能力建模分析", content: "以收入、成本、费用数据搭建分析模型，完成「盈利能力建模分析」实验并撰写分析报告。", level: "中级", diff: "中", ability: "盈利能力分析", status: "已发布", subs: 29, skills: ["盈利能力建模分析"], ideos: ["课程思政教学案例"] },
    { id: "T03", name: "盈利能力下滑问题追溯与管理建议", content: "基于建模结论追溯盈利能力下滑成因，完成「盈利能力下滑问题追溯」实验并撰写管理建议。", level: "高级", diff: "难", ability: "盈利能力分析", status: "已发布", subs: 15, skills: ["盈利下滑追溯", "管理建议撰写"], ideos: ["行业伦理"] },
    { id: "T04", name: "认知资产负债表", content: "认知资产负债表结构，理解「资产总额=负债总额+所有者权益总额」平衡规则，完成资产负债表结构分析实验。", level: "初级", diff: "易", ability: "财务状况评估", status: "已发布", subs: 33, skills: ["资产负债表结构分析"], ideos: ["红色财经资源"] },
    { id: "T05", name: "营运能力分析", content: "测算应收账款周转率等营运能力指标，完成营运能力分析实验并输出解读结论。", level: "中级", diff: "中", ability: "财务状况评估", status: "已发布", subs: 26, skills: ["营运能力分析"], ideos: [] },
    { id: "T06", name: "偿债能力与营运能力综合分析", content: "完成「偿债能力与营运能力综合实验」，对两类能力进行交叉验证并统一指标口径。", level: "中级", diff: "中", ability: "财务状况评估", status: "已发布", subs: 22, skills: ["偿债能力分析", "营运能力分析"], ideos: ["政策文件"] },
    { id: "T07", name: "偿债能力与营运能力管理建议", content: "基于综合实验结论撰写偿债与营运能力管理建议，完成管理建议撰写实验。", level: "高级", diff: "难", ability: "财务状况评估", status: "已发布", subs: 11, skills: ["管理建议撰写"], ideos: ["行业楷模"] },
    { id: "T08", name: "认知现金流量表", content: "认知现金流量表结构，理解各项目含义，完成「现金流量表结构与现金流解读」实验。", level: "初级", diff: "易", ability: "现金流诊断", status: "已发布", subs: 30, skills: ["现金流量表解读"], ideos: [] },
    { id: "T09", name: "现金流建模分析", content: "搭建现金流分析模型，完成「现金流建模分析」实验并撰写分析报告。", level: "中级", diff: "中", ability: "现金流诊断", status: "草稿", subs: 0, skills: ["现金流建模分析"], ideos: [] },
    { id: "T10", name: "现金流深度分析与财务困境", content: "基于建模结论深度分析现金流质量并判断财务困境，撰写财务困境诊断报告。", level: "高级", diff: "难", ability: "现金流诊断", status: "草稿", subs: 0, skills: ["财务困境诊断"], ideos: ["行业伦理"] },
    { id: "T11", name: "撰写财务速览备忘录", content: "综合阅读三大报表，完成「财务速览」实验并撰写财务速览备忘录。", level: "初级", diff: "易", ability: "综合财务分析", status: "已发布", subs: 27, skills: ["财务速览备忘录"], ideos: ["专业发展史"] },
    { id: "T12", name: "综合财务分析看板设计", content: "设计综合财务分析看板，完成「综合看板」实验并撰写看板设计说明。", level: "中级", diff: "中", ability: "综合财务分析", status: "已发布", subs: 18, skills: ["综合看板设计"], ideos: ["科技自立自强案例"] },
    { id: "T13", name: "风险洞察与管理建议报告", content: "基于综合分析结论进行风险洞察，完成「风险洞察与管理建议」实验并撰写管理建议报告。", level: "高级", diff: "难", ability: "综合财务分析", status: "草稿", subs: 0, skills: ["风险洞察", "管理建议撰写"], ideos: ["政策文件"] }
  ],
  submissions: [
    { id: 1, student: "王浩", task: "T05 营运能力分析", ai: 76, status: "待复核", errors: ["结论缺少业务逻辑", "图表选择不当"] },
    { id: 2, student: "陈雨", task: "T03 盈利能力下滑问题追溯", ai: 91, status: "已复核", errors: [] },
    { id: 3, student: "李娜", task: "T06 偿债与营运能力综合", ai: 68, status: "待复核", errors: ["公式错误", "指标理解错误"] },
    { id: 4, student: "赵磊", task: "T07 偿债与营运管理建议", ai: 83, status: "待复核", errors: ["流动比率计算口径不一致"] }
  ],
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
  ]
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
const tip = ref("");
let tipTimer = null;
const aiLoading = ref(false);
function showTip(t) { tip.value = t; clearTimeout(tipTimer); tipTimer = setTimeout(function () { tip.value = ""; }, 2400); }

const modal = ref({
  show: false, isEdit: false, valid: true, editId: null,
  form: { name: '', content: '', level: '初级', diff: '易', ability: '', status: '草稿', skills: [], ideos: [] }
});

// ============ 交互 ============
function go(view) { emit("navigate", view); }
function lvTag(lv) { return lv === "初级" ? "tag-blue" : lv === "中级" ? "tag-orange" : "tag-red"; }
function tagCount(t) { return (t.skills || []).length + (t.ideos || []).length; }

function openModal(task) {
  if (task) {
    modal.value.isEdit = true;
    modal.value.editId = task.id;
    modal.value.form = {
      name: task.name, content: task.content || '', level: task.level, diff: task.diff, ability: task.ability,
      status: task.status, skills: [].concat(task.skills || []), ideos: [].concat(task.ideos || [])
    };
  } else {
    modal.value.isEdit = false;
    modal.value.editId = null;
    modal.value.form = { name: '', content: '', level: '初级', diff: '易', ability: '', status: '草稿', skills: [], ideos: [] };
  }
  modal.value.valid = true;
  modal.value.show = true;
}
function closeModal() { modal.value.show = false; }
function toggleTag(kind, tag) {
  const arr = modal.value.form[kind];
  const i = arr.indexOf(tag);
  if (i >= 0) arr.splice(i, 1);
  else arr.push(tag);
}
function saveTask() {
  const f = modal.value.form;
  if (!f.name.trim()) { modal.value.valid = false; return; }
  if (modal.value.isEdit) {
    const t = teacher.value.tasks.find(function (x) { return x.id === modal.value.editId; });
    if (t) Object.assign(t, { name: f.name, content: f.content, level: f.level, diff: f.diff, ability: f.ability, status: f.status, skills: [].concat(f.skills), ideos: [].concat(f.ideos) });
    lsSet(LS_TEACHER, teacher.value);
    showTip("任务已更新");
  } else {
    const nid = (function () {
      let mx = 0;
      teacher.value.tasks.forEach(function (x) { const n = parseInt(String(x.id).replace(/\D/g, ""), 10); if (n > mx) mx = n; });
      return 'T' + String(mx + 1).padStart(2, '0');
    })();
    teacher.value.tasks.push({
      id: nid, name: f.name, content: f.content, level: f.level, diff: f.diff, ability: f.ability,
      status: f.status, subs: 0, skills: [].concat(f.skills), ideos: [].concat(f.ideos)
    });
    lsSet(LS_TEACHER, teacher.value);
    showTip("任务已创建" + (f.status === "已发布" ? "并发布" : "（草稿）"));
  }
  closeModal();
}
function delTask(t) {
  teacher.value.tasks = teacher.value.tasks.filter(function (x) { return x.id !== t.id; });
  lsSet(LS_TEACHER, teacher.value);
  showTip("任务 " + t.id + " 已删除");
}
// ============ AI 一键识别标签（依据任务名称与任务内容） ============
function pickTags(list, max) {
  const out = [];
  (list || []).forEach(function (x) {
    const s = String(x == null ? "" : x).trim();
    if (s && out.indexOf(s) < 0 && out.length < max) out.push(s);
  });
  return out;
}
function localTagScan() {
  const text = (modal.value.form.name || "") + " " + (modal.value.form.content || "");
  modal.value.form.skills = SKILL_TAGS.filter(function (s) { return text.indexOf(s) >= 0; }).slice(0, 4);
  modal.value.form.ideos = IDEOLOGY_TAGS.filter(function (s) { return text.indexOf(s) >= 0; }).slice(0, 3);
}
async function aiTags() {
  const f = modal.value.form;
  if (!f.name.trim()) { showTip("请先填写任务名称"); return; }
  aiLoading.value = true;
  try {
    const resp = await fetch(AI_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json", "Authorization": "Bearer " + AI_KEY },
      body: JSON.stringify({
        inputs: {
          is_gen_label: "1",
          task_name: f.name,
          task_content: f.content || ""
        },
        response_mode: "blocking",
        query: "请根据任务名称和任务内容识别标签，只输出 JSON：{\"skill_knowledge_tags\":[\"技能知识标签\"],\"ideological_resource_tags\":[\"思政资源标签\"]}。技能知识标签选 0-4 个，思政资源标签选 0-3 个，不要输出其他内容。",
        user: AI_USER,
        files: []
      })
    });
    if (!resp.ok) throw new Error("HTTP " + resp.status);
    const data = await resp.json();
    const raw = (data && data.answer) || "";
    const m = String(raw).match(/\{[\s\S]*\}/);
    if (!m) throw new Error("AI 返回格式异常");
    const obj = JSON.parse(m[0]);
    const skills = pickTags(obj.skill_knowledge_tags, 4);
    const ideos = pickTags(obj.ideological_resource_tags, 3);
    if (!skills.length && !ideos.length) {
      localTagScan();
      showTip("AI 未返回有效标签，已按关键词智能匹配");
    } else {
      modal.value.form.skills = skills;
      modal.value.form.ideos = ideos;
      showTip("已生成 " + skills.length + " 个技能标签、" + ideos.length + " 个思政标签");
    }
  } catch (err) {
    localTagScan();
    showTip("AI 请求失败，已按关键词智能匹配：" + (err && err.message ? err.message : err));
  } finally {
    aiLoading.value = false;
  }
}

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
.grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
.tag { display: inline-flex; align-items: center; font-size: 12px; padding: 1px 8px; border-radius: 4px; font-weight: 500; white-space: nowrap; }
.tag-blue { background: #EAF2FC; color: #1D4FA8; }
.tag-orange { background: #FFF3E8; color: #C96A06; }
.tag-red { background: #FFECEC; color: #D93025; }
.tag-green { background: #F1FAE8; color: #3C8E12; }
.tag-gray { background: #F7FAFD; color: #97A1B2; }
.tip { margin-top: 12px; background: #EAF2FC; border: 1px solid #DCEBFB; color: #1D4FA8; font-size: 13px; border-radius: 6px; padding: 8px 12px; }
@media (max-width: 900px) { .grid-2 { grid-template-columns: 1fr; } }

/* ============ 表格 ============ */
.table-wrap { overflow-x: auto; }
.table-wrap table { width: 100%; border-collapse: collapse; font-size: 13px; }
.table-wrap th { background: #F7FAFD; text-align: left; padding: 11px 12px; font-weight: 600; color: #1F2733; border-bottom: 1px solid #E3E9F2; white-space: nowrap; }
.table-wrap td { padding: 11px 12px; border-bottom: 1px solid #EEF2F8; color: #1F2733; vertical-align: middle; }
.table-wrap tbody tr { transition: background .15s; }
.table-wrap tbody tr:hover td { background: #F7FAFD; }
.task-name b { color: #2B6CD6; margin-right: 4px; }
.tag-cell { display: flex; flex-wrap: wrap; gap: 5px; align-items: center; }
.more-tag { font-size: 11px; color: #97A1B2; }
.act { display: flex; gap: 6px; }
.del { color: #FF4D4F; }
.del:hover { border-color: #FF4D4F; color: #FF4D4F; }

/* ============ 表单 ============ */
.field { margin-bottom: 14px; }
.field label { display: block; font-size: 13px; font-weight: 600; margin-bottom: 6px; }
.field input, .field select, .field textarea {
  width: 100%; height: 34px; border: 1px solid #E3E9F2; border-radius: 6px; padding: 0 11px;
  background: #fff; color: #1F2733; font-size: 14px; font-family: inherit;
  transition: border-color .15s, box-shadow .15s;
}
.field textarea { height: auto; padding: 8px 11px; resize: vertical; min-height: 70px; }
.field input:focus, .field select:focus, .field textarea:focus { outline: none; border-color: #2B6CD6; box-shadow: 0 0 0 2px #EAF2FC; }
.field .err { color: #FF4D4F; font-size: 12px; margin-top: 4px; display: none; }
.field.invalid input { border-color: #FF4D4F; }
.field.invalid .err { display: block; }
.form-row { display: flex; gap: 14px; }
.form-row .field { flex: 1; }
.ai-field { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }

/* ============ 标签选择器 ============ */
.tsel-wrap { display: flex; flex-wrap: wrap; gap: 7px; }
.tsel {
  font-size: 12px; padding: 5px 11px; border: 1px solid #E3E9F2; border-radius: 99px;
  cursor: pointer; color: #5A6577; user-select: none; transition: .15s; background: #fff;
}
.tsel:hover { border-color: #2B6CD6; }
.tsel.on { background: #2B6CD6; color: #fff; border-color: #2B6CD6; }

/* ============ 弹窗 ============ */
.mask { position: fixed; inset: 0; background: rgba(16, 24, 40, .45); z-index: 50; display: flex; align-items: center; justify-content: center; padding: 20px; animation: maskFade .2s; }
@keyframes maskFade { from { opacity: 0; } to { opacity: 1; } }
.modal {
  background: #fff; border-radius: 8px; width: 480px; max-width: 100%; max-height: 88vh; overflow: auto;
  box-shadow: 0 6px 16px rgba(16, 38, 76, .10), 0 3px 6px rgba(16, 38, 76, .06);
  animation: modalIn .22s cubic-bezier(.3, 1.2, .5, 1);
}
@keyframes modalIn { from { opacity: 0; transform: translateY(14px) scale(.97); } to { opacity: 1; transform: none; } }
.modal .m-h { padding: 18px 22px; border-bottom: 1px solid #E3E9F2; font-size: 16px; font-weight: 600; display: flex; justify-content: space-between; align-items: center; }
.modal .m-b { padding: 20px 22px; }
.modal .m-f { padding: 14px 22px; border-top: 1px solid #E3E9F2; display: flex; justify-content: flex-end; gap: 10px; }
.x-btn { width: 28px; height: 28px; border-radius: 6px; color: #97A1B2; display: flex; align-items: center; justify-content: center; transition: .15s; border: none; background: none; cursor: pointer; font-size: 14px; }
.x-btn:hover { background: #F7FAFD; color: #1F2733; }
</style>
