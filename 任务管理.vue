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
            <input v-model="modal.form.name" placeholder="如：毛利率异常分析与费用率排查" />
            <div class="err">请输入任务名称</div>
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
            <input v-model="modal.form.ability" placeholder="如：指标分析" />
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
          <div class="field ai-field">
            <button class="btn btn-ghost btn-sm" @click="aiTags">AI 一键识别标签</button>
            <span class="hint">依据任务名称自动生成技能与思政标签（待接入）</span>
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
const LS_TEACHER = "fd_teacher_v1";

// ============ AI 占位（接入后替换 AI_KEY） ============
const AI_URL = "https://agent.gjt-smart.com/v1/chat-messages";
const AI_KEY = "app-xxxxxx"; // TODO 占位

const SKILL_TAGS = ['财报数据治理', '常规经营解析', '异常特征识别', '现金流分析', '风险预警', '非经营与宏观支持', '可视化建模', '毛利率分析', '费用率分析', '流动比率', '速动比率', '资产负债率', '应收账款周转率', '利润质量评价', '杜邦分析'];
const IDEOLOGY_TAGS = ['大国工匠', '行业楷模', '科技自立自强案例', '行业伦理', '红色财经资源', '政策文件', '专业发展史', '课程思政教学案例'];

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

const modal = ref({
  show: false, isEdit: false, valid: true, editId: null,
  form: { name: '', level: '初级', diff: '易', ability: '', status: '草稿', skills: [], ideos: [] }
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
      name: task.name, level: task.level, diff: task.diff, ability: task.ability,
      status: task.status, skills: [].concat(task.skills || []), ideos: [].concat(task.ideos || [])
    };
  } else {
    modal.value.isEdit = false;
    modal.value.editId = null;
    modal.value.form = { name: '', level: '初级', diff: '易', ability: '', status: '草稿', skills: [], ideos: [] };
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
    if (t) Object.assign(t, { name: f.name, level: f.level, diff: f.diff, ability: f.ability, status: f.status, skills: [].concat(f.skills), ideos: [].concat(f.ideos) });
    lsSet(LS_TEACHER, teacher.value);
    showTip("任务已更新");
  } else {
    const nid = 'T' + String(teacher.value.tasks.length + 10);
    teacher.value.tasks.push({
      id: nid, name: f.name, level: f.level, diff: f.diff, ability: f.ability,
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
function aiTags() { showTip("AI 功能待接入：请在 AI_KEY 填入智能体 API Key"); }

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
