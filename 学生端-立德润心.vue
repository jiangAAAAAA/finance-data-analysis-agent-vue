<template>
  <div class="pg">
    <!-- 页头 -->
    <div class="page-head">
      <div class="page-head-bar"></div>
      <span class="page-head-title">立德润心</span>
      <span class="page-head-sub">立德Agent · 智能推送与引导思辨</span>
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
            <button class="btn btn-primary mt8" @click="aiDiscuss" :disabled="chatLoading">开始与立德 Agent 讨论</button>

            <!-- 对话记录 -->
            <div class="chat-sec mt8">对话</div>
            <div class="chat-list" v-if="chatMsgs.length">
              <div
                v-for="(m, i) in chatMsgs"
                :key="i"
                class="chat-msg"
                :class="m.role"
              >
                <div class="cm-label">{{ m.role === 'user' ? '我' : '立德 Agent' }}</div>
                <div class="cm-text">
                  <template v-if="m.role === 'assistant' && chatLoading && i === chatMsgs.length - 1 && !m.text">思考中…</template>
                  <template v-else>{{ m.text }}</template>
                </div>
              </div>
            </div>
            <p v-else class="hint mt8">输入你的思考 / 回答，与立德 Agent 展开伦理冲突讨论</p>

            <!-- 输入框 -->
            <div class="chat-input-row">
              <input
                v-model="chatInput"
                class="chat-input"
                type="text"
                placeholder="输入你的回答后回车…"
                :disabled="chatLoading"
                @keyup.enter="sendMessage"
              />
              <button class="btn btn-primary btn-sm" :disabled="chatLoading || !chatInput.trim()" @click="sendMessage">{{ chatLoading ? '思考中' : '发送' }}</button>
            </div>
          </template>
          <div v-else class="empty">← 请选择一个立德润心情境，开始与立德 Agent 讨论</div>
        </div>
      </div>
    </div>

    <!-- 如何运作 -->
    <div class="card mt2">
      <div class="section-title"><span class="bar"></span>如何运作</div>
      <p class="how-it-works">教师端「启智立德」上传课程思政资源并设置思政资源标签与技能知识标签 → 立德 Agent 依据课程能力图谱对知识点、岗位任务、思政主题、案例素材做标签化处理，生成技能知识点与思政素材的智能匹配 → 教师确认 / 修改关联 → 你在学习对应知识点时，系统自动推送相关立德润心情境（如数据真实性、财务造假、盈余管理、数据隐私、算法偏差、商业秘密）。</p>
    </div>

    <!-- 提示 -->
    <div v-if="tip" class="tip">{{ tip }}</div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";

const emit = defineEmits(["navigate"]);

// ============ AI 配置（立德 Agent · Dify 风格对话接口） ============
const AI_URL = "https://agent.gjt-smart.com/v1/chat-messages";
const AI_KEY = "app-xplYEWYKIQmzCnudOlDlGHnD";

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

// ============ 立德 Agent 多轮对话 ============
const chatMsgs = ref([]);      // 聊天记录 [{ role: 'user'|'assistant', text }]
const chatInput = ref("");     // 输入框内容（学生回答 → query）
const chatLoading = ref(false);
const conversationId = ref(""); // 会话记忆：首次为空，响应后回填，后续轮次回传

function selectScene(s) {
  // 切换情境：task_context 变更，重置对话与会话记忆，避免上下文串扰
  if (cur.value !== s) {
    cur.value = s;
    conversationId.value = "";
    chatMsgs.value = [];
  }
}

function aiDiscuss() {
  // 发起讨论：自动发送当前情境的第一条引导问题，作为首轮 query
  if (!cur.value) { showTip("请先选择一个立德润心情境"); return; }
  chatInput.value = cur.value.guide[0] || "请引导我围绕当前情境进行伦理思考。";
  sendMessage();
}

async function sendMessage() {
  const q = chatInput.value.trim();
  if (!q || chatLoading.value || !cur.value) return;
  chatMsgs.value.push({ role: "user", text: q });
  chatInput.value = "";
  chatLoading.value = true;
  const c = cur.value;
  try {
    const resp = await fetch(AI_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json", "Authorization": "Bearer " + AI_KEY },
      body: JSON.stringify({
        inputs: {
          is_gen_label: "0",
          course_name: "财务大数据分析",
          knowledge_point: c.kp,
          task_context: c.scene
        },
        response_mode: "streaming",
        conversation_id: conversationId.value,
        query: q,
        user: "apifox",
        files: []
      })
    });
    if (!resp.ok || !resp.body) { throw new Error("HTTP " + resp.status); }
    const reader = resp.body.getReader();
    const decoder = new TextDecoder("utf-8");
    let buf = "";
    let asst = { role: "assistant", text: "" };
    chatMsgs.value.push(asst);
    // 读取 SSE 流：逐行解析 data: JSON 事件
    for (;;) {
      const { done, value } = await reader.read();
      if (done) break;
      buf += decoder.decode(value, { stream: true });
      let nl;
      while ((nl = buf.indexOf("\n")) >= 0) {
        const line = buf.slice(0, nl).replace(/\r$/, "");
        buf = buf.slice(nl + 1);
        if (!line || line.indexOf("data:") !== 0) continue;
        const payload = line.slice(5).trim();
        if (!payload || payload === "[DONE]") continue;
        let ev;
        try { ev = JSON.parse(payload); } catch (e) { continue; }
        // answer 为增量片段，逐段累积
        if (ev.event === "message" && ev.answer) { asst.text += ev.answer; }
        // 会话结束 / 工作流结束事件携带 conversation_id，保存用于后续多轮
        if (ev.conversation_id && (ev.event === "message_end" || ev.event === "workflow_finished")) {
          conversationId.value = ev.conversation_id;
        }
      }
    }
  } catch (err) {
    chatMsgs.value.push({ role: "assistant", text: "请求失败：" + (err && err.message ? err.message : err) + "，请稍后重试。" });
  } finally {
    chatLoading.value = false;
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

/* ============ 讨论面板 ============ */
.ethic-chat { position: sticky; top: 16px; }
.chat-panel { background: #fff; border: 1px solid #E3E9F2; border-radius: 8px; padding: 16px; box-shadow: 0 1px 2px rgba(16, 38, 76, .04), 0 2px 8px rgba(16, 38, 76, .05); }
.chat-head { display: flex; align-items: center; justify-content: space-between; gap: 8px; margin-bottom: 12px; padding-bottom: 10px; border-bottom: 1px solid #EEF2F8; }
.chat-title { font-size: 15px; font-weight: 600; color: #1F2733; }
.chat-scene { font-size: 13px; line-height: 1.7; background: #F7FAFD; border-radius: 6px; padding: 10px 12px; color: #1F2733; }
.chat-sec { font-size: 13px; font-weight: 600; margin: 12px 0 6px; color: #1F2733; }
.chat-guide { display: flex; flex-direction: column; gap: 7px; }
.cg { font-size: 12px; color: #5A6577; line-height: 1.6; }

/* 对话记录 */
.chat-list { display: flex; flex-direction: column; gap: 8px; max-height: 320px; overflow-y: auto; }
.chat-msg { display: flex; flex-direction: column; gap: 3px; }
.chat-msg.user { align-items: flex-end; }
.chat-msg.assistant { align-items: flex-start; }
.cm-label { font-size: 11px; color: #97A1B2; padding: 0 2px; }
.cm-text { font-size: 13px; line-height: 1.6; color: #1F2733; background: #F7FAFD; border: 1px solid #EEF2F8; border-radius: 8px; padding: 8px 10px; white-space: pre-wrap; word-break: break-word; max-width: 100%; }
.chat-msg.user .cm-text { background: #EAF2FC; border-color: #DCEBFB; color: #1D4FA8; }

/* 输入框 */
.chat-input-row { display: flex; gap: 8px; margin-top: 10px; }
.chat-input { flex: 1; height: 32px; padding: 0 10px; border: 1px solid #D6DEE9; border-radius: 6px; font-size: 13px; color: #1F2733; outline: none; box-sizing: border-box; }
.chat-input:focus { border-color: #2B6CD6; }
.chat-input:disabled { background: #F7FAFD; cursor: not-allowed; }

/* ============ 如何运作 ============ */
.how-it-works { font-size: 13px; line-height: 1.8; color: #5A6577; }

@media (max-width: 900px) {
  .ethic-layout { grid-template-columns: 1fr; }
  .ethic-chat { position: static; }
  .ethic-list { grid-template-columns: 1fr; }
}
</style>
