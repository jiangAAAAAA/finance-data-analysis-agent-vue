<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">启智立德</span>
    </div>

    <!-- ===== 课程思政资源库 ===== -->
    <div class="card">
      <div class="section-title">
        <span class="bar"></span>课程思政资源库
        <span class="lib-meta">{{ ideology.resources.length }} 份资源 · 标签化 · 可检索 · 来源标注</span>
        <button class="btn btn-primary btn-sm" @click="openResource(null)">上传资源</button>
      </div>
      <div class="res-lib">
        <div class="res-item" v-for="r in ideology.resources" :key="r.id">
          <div class="ri-main">
            <div class="ri-name">{{ r.name }}</div>
            <div class="ri-src">{{ r.src }} · {{ r.type }} · {{ r.date }}</div>
            <div class="ri-tags">
              <span class="itag ideo" v-for="x in r.ideo" :key="x">{{ x }}</span>
              <span class="itag skill" v-for="x in r.skill" :key="'s' + x">{{ x }}</span>
            </div>
          </div>
          <div class="ri-act">
            <button class="btn btn-ghost btn-sm" @click="openResource(r)">编辑</button>
            <button class="btn btn-ghost btn-sm del" @click="delResource(r)">删除</button>
          </div>
        </div>
      </div>
      <div class="tag-legend">
        <div class="tl-title">思政资源标签体系</div>
        <div class="tl-chips">
          <span class="itag ideo" v-for="x in IDEOLOGY_TAGS" :key="x">{{ x }}</span>
        </div>
      </div>
    </div>

    <!-- ===== 上传/编辑资源弹窗 ===== -->
    <div class="mask" v-if="resModal.show" @click.self="resModal.show = false">
      <div class="modal">
        <div class="m-h">
          {{ resModal.isEdit ? '编辑课程思政资源' : '上传课程思政资源' }}
          <button class="x-btn" @click="resModal.show = false">✕</button>
        </div>
        <div class="m-b">
          <div class="field" :class="{ invalid: !resModal.valid }">
            <label>资源名称</label>
            <input v-model="resModal.form.name" placeholder="如：把账做准的人——大国工匠徐春梅" />
            <div class="err">请输入资源名称</div>
          </div>
          <div class="field">
            <label>来源</label>
            <input v-model="resModal.form.src" placeholder="如：课程思政教学案例库" />
          </div>
          <div class="field">
            <label>资源内容</label>
            <textarea rows="4" v-model="resModal.form.content" placeholder="粘贴或输入资源正文 / 案例模板 / 文件要点，AI 将据此一键识别思政资源标签与技能知识标签"></textarea>
          </div>
          <div class="row">
            <div class="field" style="flex:1">
              <label>类型</label>
              <select v-model="resModal.form.type">
                <option v-for="t in resTypes" :key="t">{{ t }}</option>
              </select>
            </div>
            <div class="field" style="flex:1">
              <label>日期</label>
              <input type="date" v-model="resModal.form.date" />
            </div>
          </div>
          <div class="field">
            <label>标签识别</label>
            <button class="btn btn-primary btn-sm" :disabled="aiLoading" @click="aiResTags">
              {{ aiLoading ? '识别中…' : 'AI 一键识别思政 / 技能标签' }}
            </button>
            <div class="ai-hint">依据资源名称与资源内容，由立德 Agent 自动打标；失败时降级为关键词匹配</div>
          </div>
          <div class="field">
            <label>思政资源标签</label>
            <div class="tsel-wrap">
              <span
                v-for="x in IDEOLOGY_TAGS" :key="'ri' + x"
                class="tsel" :class="{ on: resModal.form.ideo.includes(x) }"
                @click="toggleResTag('ideo', x)"
              >{{ x }}</span>
            </div>
          </div>
          <div class="field">
            <label>技能知识标签</label>
            <div class="tsel-wrap">
              <span
                v-for="x in SKILL_TAGS" :key="'rs' + x"
                class="tsel" :class="{ on: resModal.form.skill.includes(x) }"
                @click="toggleResTag('skill', x)"
              >{{ x }}</span>
            </div>
          </div>
        </div>
        <div class="m-f">
          <button class="btn btn-default" @click="resModal.show = false">取消</button>
          <button class="btn btn-primary" @click="saveResource">保存</button>
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
const LS_IDEO = "fd_ideo_v2";

const PRESET_IDEO = {
  resources: [
    { id: "R1", name: "把账做准的人——大国工匠徐春梅", ideo: ["大国工匠", "课程思政教学案例"], skill: ["利润表认知", "盈利能力建模分析"], src: "课程思政教学案例库", type: "案例", date: "2025-09-01" },
    { id: "R2", name: "财务造假的代价——瑞幸咖啡舞弊警示", ideo: ["行业伦理", "红色财经资源"], skill: ["现金流建模分析", "财务困境诊断"], src: "红色财经资源·舞弊警示录", type: "警示", date: "2025-09-02" },
    { id: "R3", name: "国产财务软件的突破", ideo: ["科技自立自强案例"], skill: ["综合看板设计"], src: "行业资讯·工信部专栏", type: "资讯", date: "2025-09-03" },
    { id: "R4", name: "边区经济中的财经纪律", ideo: ["红色财经资源", "政策文件"], skill: ["资产负债表结构分析"], src: "政策文件·红色财经史料", type: "史料", date: "2025-09-04" },
    { id: "R5", name: "关于数据安全的若干规定", ideo: ["政策文件", "行业伦理"], skill: ["利润表认知", "现金流量表解读"], src: "政策文件·国家数据局", type: "文件", date: "2025-09-05" },
    { id: "R6", name: "一张报表背后的十年——财务人的坚守", ideo: ["行业楷模", "专业发展史"], skill: ["盈利下滑追溯", "管理建议撰写"], src: "专业发展史·《财务人的坚守》", type: "故事", date: "2025-09-06" }
  ]
};

const IDEOLOGY_TAGS = ["大国工匠", "行业楷模", "科技自立自强案例", "行业伦理", "红色财经资源", "政策文件", "专业发展史", "课程思政教学案例"];
const SKILL_TAGS = ['利润表认知', '盈利能力建模分析', '盈利下滑追溯', '资产负债表结构分析', '营运能力分析', '偿债能力分析', '现金流量表解读', '现金流建模分析', '财务困境诊断', '财务速览备忘录', '综合看板设计', '风险洞察', '管理建议撰写'];
const resTypes = ["案例", "警示", "资讯", "史料", "文件", "故事"];

function loadIdeo() {
  let d = lsGet(LS_IDEO);
  if (!d) { d = JSON.parse(JSON.stringify(PRESET_IDEO)); lsSet(LS_IDEO, d); }
  return d;
}

// ============ 页面状态 ============
const ideology = ref(loadIdeo());
const tip = ref("");
let tipTimer = null;
function showTip(t) { tip.value = t; clearTimeout(tipTimer); tipTimer = setTimeout(function () { tip.value = ""; }, 2400); }

const resModal = ref({
  show: false, isEdit: false, valid: true, editId: null,
  form: { name: "", src: "", type: "案例", date: "", content: "", ideo: [], skill: [] }
});
const aiLoading = ref(false);

// ============ AI 智能体配置（立德 Agent · Dify 风格对话接口） ============
const AI_URL = "https://agent.gjt-smart.com/v1/chat-messages";
const AI_KEY = "app-xplYEWYKIQmzCnudOlDlGHnD";
const AI_USER = "ideo-resource";

// ============ 交互 ============
function go(view) { emit("navigate", view); }

/* ===== 资源 CRUD ===== */
function openResource(r) {
  if (r) {
    resModal.value.isEdit = true;
    resModal.value.editId = r.id;
    resModal.value.form = {
      name: r.name, src: r.src, type: r.type, date: r.date, content: r.content || "",
      ideo: [].concat(r.ideo), skill: [].concat(r.skill)
    };
  } else {
    resModal.value.isEdit = false;
    resModal.value.editId = null;
    var d = new Date();
    resModal.value.form = {
      name: "", src: "", type: "案例",
      date: d.getFullYear() + "-" + String(d.getMonth() + 1).padStart(2, "0") + "-" + String(d.getDate()).padStart(2, "0"),
      content: "", ideo: [], skill: []
    };
  }
  resModal.value.valid = true;
  resModal.value.show = true;
}
function toggleResTag(kind, tag) {
  var arr = resModal.value.form[kind];
  var i = arr.indexOf(tag);
  if (i >= 0) { arr.splice(i, 1); } else { arr.push(tag); }
}
function saveResource() {
  var f = resModal.value.form;
  if (!f.name.trim()) { resModal.value.valid = false; return; }
  if (resModal.value.isEdit) {
    var r = ideology.value.resources.find(function (x) { return x.id === resModal.value.editId; });
    if (r) { Object.assign(r, { name: f.name, src: f.src, type: f.type, date: f.date, content: f.content || "", ideo: [].concat(f.ideo), skill: [].concat(f.skill) }); }
    showTip("资源已更新");
  } else {
    var nid = "R" + (ideology.value.resources.length + 1);
    ideology.value.resources.unshift({
      id: nid, name: f.name, src: f.src || "本地上传", type: f.type, date: f.date, content: f.content || "",
      ideo: [].concat(f.ideo), skill: [].concat(f.skill)
    });
    showTip("已上传并存入课程思政资源库");
  }
  lsSet(LS_IDEO, ideology.value);
  resModal.value.show = false;
}
function delResource(r) {
  ideology.value.resources = ideology.value.resources.filter(function (x) { return x.id !== r.id; });
  lsSet(LS_IDEO, ideology.value);
  showTip("资源已删除");
}

/* ===== AI 一键识别思政 / 技能标签（入参为 resource，is_gen_label=1） ===== */
function pickTags(list, max) {
  const out = [];
  (list || []).forEach(function (x) {
    const s = String(x == null ? "" : x).trim();
    if (s && out.indexOf(s) < 0 && out.length < max) out.push(s);
  });
  return out;
}
function localResScan() {
  const text = (resModal.value.form.name || "") + " " + (resModal.value.form.content || "");
  resModal.value.form.skill = SKILL_TAGS.filter(function (s) { return text.indexOf(s) >= 0; }).slice(0, 4);
  resModal.value.form.ideo = IDEOLOGY_TAGS.filter(function (s) { return text.indexOf(s) >= 0; }).slice(0, 3);
}
async function aiResTags() {
  const f = resModal.value.form;
  if (!f.name.trim()) { showTip("请先填写资源名称"); return; }
  if (!(f.content || "").trim()) { showTip("请先填写资源内容"); return; }
  aiLoading.value = true;
  try {
    const resp = await fetch(AI_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json", "Authorization": "Bearer " + AI_KEY },
      body: JSON.stringify({
        inputs: {
          is_gen_label: "1",
          resource_name: f.name,
          resource_content: f.content
        },
        response_mode: "blocking",
        query: "请根据思政资源名称和资源内容识别标签，只输出 JSON：{\"skill_knowledge_tags\":[\"技能知识标签\"],\"ideological_resource_tags\":[\"思政资源标签\"]}。技能知识标签选 0-4 个，思政资源标签选 0-3 个，不要输出其他内容。",
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
      localResScan();
      showTip("AI 未返回有效标签，已按关键词智能匹配");
    } else {
      resModal.value.form.skill = skills;
      resModal.value.form.ideo = ideos;
      showTip("已生成 " + ideos.length + " 个思政标签、" + skills.length + " 个技能标签");
    }
  } catch (err) {
    localResScan();
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
.mt { margin-top: 16px; }
.muted { color: #5A6577; font-size: 13px; }
.card { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px; box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); }
.section-title { display: flex; align-items: center; gap: 8px; font-size: 15px; font-weight: 600; color: #1F2733; margin-bottom: 14px; flex-wrap: wrap; }
.section-title .bar { width: 4px; height: 15px; border-radius: 2px; background: #2B6CD6; }
.section-title .lib-meta { margin-left: auto; font-size: 12px; font-weight: 400; color: #97A1B2; }
.tag { display: inline-flex; align-items: center; font-size: 12px; padding: 1px 8px; border-radius: 4px; font-weight: 500; white-space: nowrap; }
.tag-blue { background: #EAF2FC; color: #1D4FA8; }
.tag-green { background: #F1FAE8; color: #3C8E12; }
.tag-orange { background: #FFF3E8; color: #C96A06; }
.tip { margin-top: 12px; background: #EAF2FC; border: 1px solid #DCEBFB; color: #1D4FA8; font-size: 13px; border-radius: 6px; padding: 8px 12px; }

/* ============ 资源库 ============ */
.res-lib { display: flex; flex-direction: column; gap: 10px; }
.res-item { display: flex; align-items: flex-start; gap: 12px; border: 1px solid #E3E9F2; border-radius: 6px; padding: 12px 14px; background: #fff; transition: .15s; }
.res-item:hover { border-color: #2B6CD6; }
.ri-main { flex: 1; min-width: 0; }
.ri-name { font-weight: 600; font-size: 13px; }
.ri-src { font-size: 11px; color: #97A1B2; margin-top: 4px; }
.ri-tags { display: flex; flex-wrap: wrap; gap: 6px; margin-top: 8px; }
.ri-act { display: flex; flex-direction: column; gap: 6px; flex-shrink: 0; }
.itag { font-size: 11px; font-weight: 600; padding: 3px 9px; border-radius: 99px; white-space: nowrap; }
.itag.ideo { color: #1D4FA8; background: #EAF2FC; }
.itag.skill { color: #B26A00; background: #FFF3E0; }
.tag-legend { margin-top: 14px; padding-top: 12px; border-top: 1px dashed #E3E9F2; }
.tl-title { font-size: 12px; color: #5A6577; margin-bottom: 8px; font-weight: 600; }
.tl-chips { display: flex; flex-wrap: wrap; gap: 6px; }
.del { color: #FF4D4F; }
.del:hover { border-color: #FF4D4F; color: #FF4D4F; }

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
.field .err { color: #FF4D4F; font-size: 12px; margin-top: 4px; display: none; }
.field.invalid input { border-color: #FF4D4F; }
.field.invalid .err { display: block; }
.row { display: flex; gap: 12px; }

/* ============ 标签选择器 ============ */
.tsel-wrap { display: flex; flex-wrap: wrap; gap: 7px; }
.tsel { font-size: 12px; padding: 5px 11px; border: 1px solid #E3E9F2; border-radius: 99px; cursor: pointer; color: #5A6577; user-select: none; transition: .15s; }
.tsel:hover { border-color: #2B6CD6; }
.tsel.on { background: #2B6CD6; color: #fff; border-color: #2B6CD6; }
.ai-hint { margin-top: 6px; font-size: 12px; color: #97A1B2; }
</style>
